# Raspberry Pi で NAS 構築

## 環境

- Host: Windows 10
    - CommandLine Tools:
        - PowerShell
        - Bash on Windows
- Raspberry Pi 3B+
    - OS: Raspbian Buster Lite `2020-02-13`
    - SDカード: Team microSDHC 8GB
- 玄人志向 GW3.5ACX2-U3.1AC
    - HDD: Western Digital HDD 4TB (3.5インチ) x 2

### Installation
1. Win32 Disk Imager 等を使い SDカードに Raspbian Buster Lite イメージを書き込み
2. SSH接続できるように SDカードの boot 領域に `ssh` ファイル作成
    ```shell
    # SDカードドライブ = H: に移動
    $ cd /h/
    # ssh ファイル作成
    $ touch ssh
    ```
3. SDカードを Raspberry Pi に差し込み、電源を入れる
4. Raspberry Pi に LAN を差し込み、ルータの設定で Client IP アドレスを固定する
    - ASUS RT-AC65U ルータの場合、`ネットワーク` > `クライアント` から設定
        - クライアント名: `raspberrypi`
        - IP: `192.168.11.200` (任意のIPアドレス)
        - MAC and IP address Binding: `ON`
    - ※あらかじめ `LAN` > `LAN IP` からルータのIPを固定しておく必要がある（`192.168.11.1` 等）

### Raspberry Pi 初期設定
上記で Raspberry Pi に割り振った IP 経由で SSH　接続し、セットアップする

```bash
# -- @localhost

# SSH接続
## username: pi
## password: raspberry
$ ssh pi@192.168.11.150

# -- pi@raspberrypi

# raspi-config コマンドで簡易セットアップ
$ sudo raspi-config
### <tui>
# 2. Network Options (※Wifi設定｜システムが不安定になることがあるためスキップした方が良いかも)
#   N2. Wi-fi
#     - SSID を入力して Enter
#     - Password を入力して Enter
# 4. Localisation Options (※日本語環境に最適化)
#   I1. Change Locale
#     - [x] ja_JP.UTF-8 UTF-8 (Spaceキーで✓)
#     - OK (Tab |> Enter)
#   Default locale for the system environment:
#     - ja_JP.UTF-8
#   I2. Change Timezone
#     - Asia > Tokyo
#   I4. Change Wi-fi Country (※これを設定しないと日本では許可されない電波を発して通信をしてしまうので必ず設定する)
#     - JP Japan
# 5. Interfacing Options (※SSHを使用可能にする｜これを設定しないと再起動後に SSH 接続できなくなる)
#   P2. SSH
#     - Yes
#     - ※ Yes/No の選択肢が見えないときは一度 Esc キーで戻ってから再度メニュー選択すると見えるようになることがある
# 7. Advanced Options (※SDカードの残りの領域をファイルストレージとして使用可能にする)
#   A1. Expand Filesystem Ensures ...
#     => 再起動後にファイルシステムが自動的に拡張される
### </tui>

# raspi-config が完了すると自動的に再起動されるため、再度 SSH 接続する

# system update
$ sudo apt update && sudo apt upgrade -y
```

#### Wifi設定によりネットワーク回りが不安定になった場合
Wifiの設定をしたことでネットワーク回りが不安定になった場合は、Wifi設定を削除する

```bash
# -- pi@raspberrypi

# Wifi設定を編集
$ sudo vi /etc/wpa_supplicant/wpa_supplicant.conf
### <diff>
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=JP
# 以下の設定を削除
- network={
-     ssid="Wifi_SSID"
-     psk="Wifi_Password"
- }
### </diff>

# 再起動
$ sudo reboot
```

### 既存NASからの移行
今回は、既存NASから Raspberry Pi NAS への移行だったため、データ移行作業を行う

- 構成
    - 既存NAS: `192.168.11.100` で稼働
        - HDD x 2; RAID 0
    - Raspberry Pi: `192.168.11.200` で稼働
        - 玄人志向 GW3.5ACX2-U3.1AC (HDD x 2; RAID 0)

```bash
# -- pi@raspberrypi

# 既存NASをマウント
$ sudo apt install -y cifs-utils
$ sudo mkdir -p /mnt/nas/
$ sudo mount -t cifs -o username='<username>',password='<password>',vers=1.0 //192.168.11.100 /mnt/nas

# USBデバイス確認
$ sudo fdisk -l
 :
Disk /dev/sda: 7.3 TiB, 8001557627392 bytes, 15628042241 sectors
 :
Device     Start         End     Sectors  Size Type
/dev/sda1     34       32767       32734   16M Microsoft reserved
/dev/sda2  32768 15628040191 15628007424  7.3T Microsoft basic data
## => /dev/sda2 に USB HDD がいることを確認

# USB HDD (NTFSフォーマット) マウント
$ sudo mkdir -p /mnt/usbhdd/
$ sudo mount -t ntfs /dev/sda2 /mnt/usbhdd

# データ移行作業をバックグラウンドで実行できるように tmux 仮想ターミナル導入
$ sudo apt install -y tmux

# tmux 新規セッション（rsync-nas）作成
$ tmux new -s rsync-nas
---
# 既存NAS => USB HDD 同期
$ sudo rsync -rlOtcvz --delete /mnt/nas/ /mnt/usbhdd/

# => Ctrl + B |> D キーでデタッチ
---

# tmux セッション一覧確認
$ tmux ls
rsync-nas: 1 windows ...

# セッションにアタッチする場合
$ tmux a -t rsync-nas

# => rsync コマンドの実行が完了するまで待機
```

***

## OMV インストール

- **OpenMediaVault (OMV)**
    - DebianLinuxをカーネルに用いたフリーのnetwork-attached storageシステム
    - CIFS, FTP, NFS, rsync, AFPなどのプロトコルに対応
    - ソフトウェア RAID を用いることが可能
    - WebUIを用いて環境設定・操作を行うことが可能

### OMV 導入
```bash
# -- pi@raspberrypi

# OMV インストール
## 30分～1時間程度かかる
$ wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | sudo bash

# OMV インストール後、ネットワーク構成がおかしくなってしまうため修正する
$ sudo omv-firstaid

# 再起動
$ sudo reboot
```
