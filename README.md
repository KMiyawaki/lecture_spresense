# lecture_spresense

`spresense`マイコンでの開発方法。

## WSL2からUSBデバイスに接続する

[USB デバイスを接続する](https://learn.microsoft.com/ja-jp/windows/wsl/connect-usb)を参照。
コマンドプロンプトを管理者権限で起動し、以下を実行。

```bat
usbipd list
# 実行結果
Connected:
BUSID  VID:PID    DEVICE                                                        STATE
1-1    10c4:ea60  Silicon Labs CP210x USB to UART Bridge (COM4)                 Not shared 
1-3    0461:4d22  USB 入力デバイス                                              Not shared
1-7    04f2:b684  Chicony USB2.0 Camera, IR Camera                              Not shared
1-10   8087:0033  インテル(R) ワイヤレス Bluetooth(R)                           Not shared
# IDを指定してバインド
usbipd bind --busid 1-1
usbipd list
# Sharedになる
Connected:
BUSID  VID:PID    DEVICE                                                        STATE
1-1    10c4:ea60  Silicon Labs CP210x USB to UART Bridge (COM4)                 Shared
1-3    0461:4d22  USB 入力デバイス                                              Not shared
1-7    04f2:b684  Chicony USB2.0 Camera, IR Camera                              Not shared
1-10   8087:0033  インテル(R) ワイヤレス Bluetooth(R)                           Not shared
usbipd attach --wsl --busid 1-1
```

## SDKをインストールする

[Spresense SDK スタートガイド (CLI 版)](https://developer.sony.com/spresense/development-guides/sdk_set_up_ja)を参照し、Windows11(WSL2)向けセットアップを行う。

```shell
cd
wget https://raw.githubusercontent.com/sonydevworld/spresense/master/install-tools.sh
bash install-tools.sh
emacs ~/.bashrc -nw
# 末尾に以下を追記
source ~/spresenseenv/setup
git clone --recursive https://github.com/sonydevworld/spresense.git
```

## Hello World

[3. サンプルアプリケーション "Hello, World!" の実行手順](https://developer.sony.com/spresense/development-guides/sdk_set_up_ja#_%E3%82%B5%E3%83%B3%E3%83%97%E3%83%AB%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3_hello_world_%E3%81%AE%E5%AE%9F%E8%A1%8C%E6%89%8B%E9%A0%86)に従う。

```shell
cd spresense/sdk
tools/config.py examples/hello
# 以下が出たら指示に従う
WARNING: New loader v3.4.1 is required, please download and install.
Download URL   : https://developer.sony.com/file/download/download-spresense-firmware-v3-4-1
# ダウンロードしたファイルを ~/spresense/sdk に置く。
./tools/flash.sh -e spresense-binaries-v3.4.1.zip # ダウンロードしたファイルを指定
./tools/flash.sh -l ~/spresense/firmware/spresense -c /dev/ttyUSB0
# ビルドする
make -j
# ・・・省略
Register: hello
Register: nsh
Register: sh
# ・・・省略
File nuttx.spk is successfully created.
# 書き込み
tools/flash.sh -c /dev/ttyUSB0 nuttx.spk
# 動作確認
sudo apt install screen
screen /dev/ttyUSB0 115200
nsh> hello
Hello, World!!
# Ctrl+K, A を押して Y
```

## プロジェクトを作成する

[6.3. ツールを使用する](https://developer.sony.com/spresense/development-guides/sdk_set_up_ja#_%E3%83%84%E3%83%BC%E3%83%AB%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%99%E3%82%8B)に従う。

[Lチカさせる場合のソース](./myfirstapp_main.c)

```shell
cd spresense/sdk
tools/mkappsdir.py myapps "My Apps"
tools/mkcmd.py -d myapps myfirstapp "My first app example"
# ~/spresense/myapps/myfirstapp/myfirstapp_main.c を編集する
make clean # 重要
tools/config.py -m
# Application Configuration -> Spresense SDK -> My Apps -> My first app example にチェックがあることを確認
# Save でファイル名はデフォルトで良さそう
make -j
# ・・・省略
Register: myfirstapp # 追加される
Register: hello
Register: nsh
Register: sh
# ・・・省略
File nuttx.spk is successfully created.
# 書き込み
tools/flash.sh -c /dev/ttyUSB0 nuttx.spk
# 動作確認
screen /dev/ttyUSB0 115200
nsh> myfirstapp
# 書いたプログラムが実行されればOK
# Ctrl+K, A を押して Y
```

## マルチIMU

一回サンプルをビルドすると`IMU`のサポートが有効になる。
`examples/ahrs_pwbimu`も加えることで`Madgwick`フィルタの関数を使うことができる。

```shell
cd spresense/sdk
make distclean 
tools/config.py examples/cxd5602pwbimu examples/ahrs_pwbimu
make -j
tools/flash.sh -c /dev/ttyUSB0 nuttx.spk
```

サンプルをビルドしない場合は`tools/config.py -m`で`Device Drivers->Sensor Device Support->SONY CXD5602PWBIMU Inertial Measurement Sensor support`を有効にすることで恐らく使えるようになる。

```shell
cd spresense/sdk
tools/config.py -m
```

## ボーレートを変える。

```shell
cd spresense/sdk
tools/config.py -m
```

メニュー表示後に` Device Drivers -> Serial Driver Support -> UART1 Configuration -> BAUD rate`を`2000000`に変更。

```shell
cd spresense/sdk
make -j
# ・・・省略
# 書き込み
tools/flash.sh -c /dev/ttyUSB0 nuttx.spk
# 動作確認
screen /dev/ttyUSB0 2000000
```

以降は高いボーレートで通信できる。

## RNDIS

`tools/config.py feat/rndis`を実行後に書き込むことで`Nuttx`からは`ifconfig`でネットワークインタフェースが見えたが、上位PCから`spresense`のネットワークが見えるまでは至らなかった。
