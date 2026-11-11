# RTKLIB 2.4.3 b34 — カスタマイズ版

本リポジトリは RTKLIB 2.4.3 b34 に以下のカスタマイズを加えたものです。

- **シリアルポート絶対パス対応パッチ** (`src/stream.c`)
- **ZED-F9P + ichimill 補正データ配信手順**

---

## 1. str2str のインストール

### 動作確認環境

- Ubuntu 24.04
- gcc 13.3.0
- GNU make

### 必要なパッケージ

```bash
sudo apt update
sudo apt install -y build-essential git
```

`str2str` は標準 C ライブラリと `libm` / `librt` / `libpthread` のみに依存するため、
追加のライブラリは不要です。

### ソースコードの取得

```bash
git clone https://github.com/tomson784/RTKLIB.git
cd RTKLIB
git checkout rtklib_2.4.3
```

### ビルド

```bash
cd app/consapp/str2str/gcc
make
```

ビルドに成功すると、同じディレクトリに実行ファイル `str2str` が生成されます。
ソースを修正した後などにビルドし直す場合は `make clean` してから `make` します。

`makefile` の `OPTION` で有効な機能を指定しています（デフォルト値）。

| オプション | 内容 |
|---|---|
| `-DENAGLO -DENAGAL -DENAQZS -DENACMP -DENAIRN` | GLONASS / Galileo / QZSS / BeiDou / NavIC を有効化 |
| `-DTRACE` | トレースログ出力（`-t` オプション）を有効化 |
| `-DNFREQ=5 -DNEXOBS=3` | 周波数数・拡張観測コード数 |
| `-DSVR_REUSEADDR` | TCP サーバのポート再利用を許可 |

### インストール

```bash
sudo make install
```

`/usr/local/bin/str2str` にコピーされます。インストール先を変えたい場合は
`BINDIR` を指定します。

```bash
# 例: ユーザのホーム配下にインストール（sudo 不要）
mkdir -p ~/.local/bin
make install BINDIR=~/.local/bin
```

`~/.local/bin` に PATH が通っていない場合は `~/.bashrc` に以下を追記します。

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### 動作確認

```bash
str2str -h
```

`usage: str2str [-in stream] [-out stream [-out stream...]] [options]` から始まる
ヘルプが表示されればインストール完了です。

### シリアルポートの権限設定

受信機を USB/シリアルで接続して使う場合、一般ユーザで `/dev/ttyACM*` や
`/dev/ttyUSB*` を開くには `dialout` グループへの所属が必要です。

```bash
sudo usermod -aG dialout $USER
```

設定後、一度ログアウトして再ログインしてください。`groups` コマンドで
`dialout` が含まれていることを確認できます。

### アンインストール

```bash
sudo rm /usr/local/bin/str2str
```

---

## 2. シリアルポート絶対パス対応パッチ

### 概要

`str2str` コマンドでシリアルポートを指定する際、デフォルトでは
`ttyACM0` のような `/dev/` 省略形しか使えません。
このパッチを適用すると `/dev/serial/by-id/` や `/dev/serial/by-path/` などの
絶対パスでポートを指定できるようになります。

### 背景：元のコードの問題

コマンド引数 `serial://ttyACM0:230400#rtcm3` は以下の順で処理されます。

```
serial://ttyACM0:230400#rtcm3
        │
        │ app/consapp/str2str/str2str.c: decodepath()
        │   "://" で前後を分割し、スキーム名からストリーム種別を決定
        ▼
strpath = "ttyACM0:230400#rtcm3"   type = STR_SERIAL
        │
        │ src/stream.c: stropen() → openserial()
        │   strchr(path, ':') で最初の ':' を区切り文字として使用
        ▼
port = "ttyACM0",  brate = 230400
        │
        │ src/stream.c:444  ← ここが問題
        │   sprintf(dev, "/dev/%s", port);   /dev/ を無条件に付加
        ▼
dev = "/dev/ttyACM0"  → open(dev, ...)
```

| ケース | 結果 |
|---|---|
| `serial:///dev/serial/by-id/usb-...-if00:230400` | `/dev//dev/serial/by-id/...` になり `open()` が失敗 |
| `serial:///dev/serial/by-path/pci-0000:00:14.0-usb-0:1.1:1.0-port0:230400` | パス名中のコロンをボーレート区切りと誤認識し、ポート名が壊れる |

### パッチ内容

対象ファイル: `src/stream.c`

**変更 1: `find_baud_sep()` ヘルパー関数を追加**

`openserial()` の直前に追加。パス文字列中のコロンのうち、
直後が有効なボーレート値 (300〜921600) であるものだけを区切り文字として返します。

```c
static const char *find_baud_sep(const char *s)
{
    static const int valid[]={
        300,600,1200,2400,4800,9600,19200,38400,57600,115200,230400,460800,
        921600,0
    };
    const char *p=s;
    while ((p=strchr(p,':'))!=NULL) {
        int br=0,i;
        if (sscanf(p+1,"%d",&br)==1)
            for (i=0;valid[i];i++) if (valid[i]==br) return p;
        p++;
    }
    return NULL;
}
```

**変更 2: ポート名切り出しロジックの修正**

```c
/* 修正前 */
if ((p=strchr(path,':'))) {
    strncpy(port,path,p-path); port[p-path]='\0';
    sscanf(p,":%d:%d:%c:%d:%s",&brate,&bsize,&parity,&stopb,fctr);
}
else strcpy(port,path);

/* 修正後 */
if (path[0]=='/') {
    /* 絶対パス: パス名自体にコロンを含む可能性があるため
     * ボーレート値で区切り位置を特定する */
    const char *sep=find_baud_sep(path);
    if (sep) {
        strncpy(port,path,sep-path); port[sep-path]='\0';
        sscanf(sep,":%d:%d:%c:%d:%s",&brate,&bsize,&parity,&stopb,fctr);
    }
    else { strncpy(port,path,sizeof(port)-1); port[sizeof(port)-1]='\0'; }
}
else if ((p=strchr(path,':'))) {
    strncpy(port,path,p-path); port[p-path]='\0';
    sscanf(p,":%d:%d:%c:%d:%s",&brate,&bsize,&parity,&stopb,fctr);
}
else strcpy(port,path);
```

**変更 3: デバイスパス構築の修正**

```c
/* 修正前 */
sprintf(dev,"/dev/%.*s",(int)sizeof(port)-6,port);

/* 修正後 */
if (port[0]=='/')
    { strncpy(dev,port,sizeof(dev)-1); dev[sizeof(dev)-1]='\0'; }
else
    sprintf(dev,"/dev/%.*s",(int)(sizeof(dev)-6),port);
```

### ビルド方法

[1. str2str のインストール](#1-str2str-のインストール) を参照してください。

### str2str の使い方

#### 基本構文

```
str2str -in <入力ストリーム> [-out <出力ストリーム> ...] [オプション]
```

#### ストリームの種類

| スキーム | 用途 |
|---|---|
| `serial://` | シリアルポート |
| `tcpsvr://` | TCP サーバ（クライアントの接続を待ち受け） |
| `tcpcli://` | TCP クライアント（サーバに接続） |
| `ntrip://` | NTRIP クライアント |
| `ntrips://` | NTRIP サーバ（出力のみ） |
| `file://` または パス直書き | ファイル |

#### シリアルポートの指定形式

```
serial://<ポート>[:ボーレート[:ビット数[:パリティ[:ストップビット[:フロー制御]]]]]#フォーマット
```

| パラメータ | 選択肢 | デフォルト |
|---|---|---|
| ボーレート | 300 / 600 / 1200 / 2400 / 4800 / 9600 / 19200 / 38400 / 57600 / 115200 / **230400** / 460800 / 921600 | 9600 |
| ビット数 | 7 / **8** | 8 |
| パリティ | **n** (なし) / o (奇数) / e (偶数) | n |
| ストップビット | **1** / 2 | 1 |
| フロー制御 | **off** / rts | off |
| フォーマット (`#`) | rtcm2 / **rtcm3** / ubx / oem4 / oem3 / など | — |

> `#フォーマット` はボーレートなどのパラメータより先に書きます。  
> 例: `serial://ttyACM0:230400#rtcm3`

#### 主なオプション

| オプション | 説明 |
|---|---|
| `-in <path>` | 入力ストリーム |
| `-out <path>` | 出力ストリーム（最大 5 個） |
| `-c <file>` | 起動時に受信機へ送るコマンドファイル |
| `-p <lat> <lon> <hgt>` | 受信機概略位置（RTCM 基準局メッセージ用） |
| `-msg <msgs>` | RTCM メッセージタイプ指定（例: `1005,1077,1087`） |
| `-opt <opt>` | 受信機オプション |
| `-t <sec>` | タイムアウト秒数 |
| `-s <msec>` | 再接続待機時間 [ms] |
| `-r <bps>` | ログ記録レート [bps] |
| `-x <sec>` | 受信機コマンド送信待機秒数 |
| `-b <n>` | バイナリモード出力ストリーム番号 |

#### 使用例

**従来の指定方法（パッチなしでも動作）**

```bash
# /dev/ttyACM0 を 230400bps で開き RTCM3 として TCP 配信
str2str -in serial://ttyACM0:230400#rtcm3 \
        -out tcpsvr://:9000

# RTS フロー制御付き
str2str -in serial://ttyUSB0:115200:8:n:1:rts#rtcm3 \
        -out tcpsvr://:9000
```

**絶対パスによる指定（パッチ適用後）**

```bash
# by-id で指定（パス名にコロンなし — シンプル）
str2str -in serial:///dev/serial/by-id/usb-u-blox_AG_-_www.u-blox.com_u-blox_GNSS_receiver-if00:230400#rtcm3 \
        -out tcpsvr://:9000

# by-path で指定（パス名にコロンを含む — パッチが有効）
str2str -in serial:///dev/serial/by-path/pci-0000:00:14.0-usb-0:1.1:1.0-port0:230400#rtcm3 \
        -out tcpsvr://:9000

# by-path + 全パラメータ明示
str2str -in serial:///dev/serial/by-path/pci-0000:00:14.0-usb-0:1.1:1.0-port0:230400:8:n:1:off#rtcm3 \
        -out tcpsvr://:9000
```

> **スラッシュ 3 本について**: `serial://` の直後に絶対パス `/dev/...` を続けるため、  
> スラッシュが 3 本連続します（`serial:///dev/...`）。

**受信機コマンドファイルと組み合わせる**

```bash
# NEO-M8P (Rover, GLONASS 有効, 5Hz) を設定しながら TCP 配信
str2str \
  -in  serial:///dev/serial/by-id/usb-u-blox_GNSS_receiver-if00:230400#ubx \
  -out tcpsvr://:9000#rtcm3 \
  -c   data/cmd/ubx_m8p_rov_glo_5hz.cmd

# 複数出力：TCP 配信 + ファイル記録（1時間ごとにスワップ）
str2str \
  -in  serial:///dev/serial/by-id/usb-u-blox_GNSS_receiver-if00:230400#ubx \
  -out tcpsvr://:9000#rtcm3 \
  -out /var/log/gnss/%Y%m%d_%h.ubx::S=1 \
  -c   data/cmd/ubx_m8p_rov_glo_5hz.cmd
```

**絶対パスを使うべき場面**

| 状況 | 推奨 |
|---|---|
| USB を差し替えると `ttyACM0` の番号が変わる | `by-id` または `by-path` |
| 複数の GNSS 受信機を同時接続して区別したい | `by-id`（製品シリアル番号で一意に識別） |
| USB ハブの物理ポート位置で固定したい | `by-path` |

**デバイスパスの確認方法**

```bash
# by-id の一覧
ls -l /dev/serial/by-id/

# by-path の一覧
ls -l /dev/serial/by-path/

# udevadm で属性を確認
udevadm info --name=/dev/ttyACM0 | grep -E "ID_SERIAL|ID_PATH"
```

### 受信機コマンドファイル（`.cmd`）

`-c` オプションに指定するファイルです。`data/cmd/` に受信機・用途別のサンプルが収録されています。

#### ファイル構造

```
<接続時コマンド（受信機を有効化）>
@
<切断時コマンド（受信機をクリーンアップ）>
```

行頭の `!` は応答を待たずに次のコマンドを送ることを意味します。

#### 収録ファイルの命名規則

```
<メーカー>_<チップ>_<役割>_<衛星系>_<レート>.cmd
```

| 部分 | 意味 | 例 |
|---|---|---|
| メーカー | 受信機メーカー略称 | `ubx` (u-blox), `oem4` (NovAtel), `javad` |
| チップ | 型番 | `m8p`, `m8t`, `m8n` |
| 役割 | `rov` (Rover) / `ref` (Reference/基準局) / `raw` (生データ) |  |
| 衛星系 | `glo` (GLONASS) / `bds` (BeiDou) / 省略時は GPS のみ | |
| レート | `1hz`, `5hz`, `10hz` など | |

#### ubx_m8p_rov_glo_5hz.cmd の解説

u-blox NEO-M8P を **Rover（移動局）/ GLONASS 有効 / 5Hz** で設定するファイルです。

```
!UBX CFG-GNSS 0 0 30 1 3 8 16 0 0       ← BeiDou (3) を無効化
!UBX CFG-GNSS 0 0 30 1 0 8 16 0 65537   ← GPS (0) を有効化
!UBX CFG-GNSS 0 0 30 1 6 8 16 0 65537   ← GLONASS (6) を有効化
!UBX CFG-MSG 240 0 0 0 0 1 0 0          ← NMEA GGA を UART1 で出力
!UBX CFG-MSG 240 4 0 0 0 1 0 0          ← NMEA RMC を UART1 で出力
!UBX CFG-MSG 240 3 0 0 0 1 0 0          ← NMEA GSV を UART1 で出力
!UBX CFG-RATE 200 1 1                    ← 測位レートを 200ms (5Hz) に設定
@
!UBX CFG-MSG 240 0 0 0 0 0 0 0          ← 切断時: NMEA GGA を停止
!UBX CFG-MSG 240 4 0 0 0 0 0 0          ← 切断時: NMEA RMC を停止
!UBX CFG-MSG 240 3 0 0 0 0 0 0          ← 切断時: NMEA GSV を停止
```

`CFG-GNSS` の第 5 引数が GNSS システム ID (0=GPS, 3=BeiDou, 6=GLONASS)、
最終引数 `65537` = `0x10001` は「有効化 + L1 信号使用」を意味するフラグです。
