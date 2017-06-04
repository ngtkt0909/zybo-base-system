# zybo-base-system

## 概要
Digilent社のWebページにて公開されている[Zybo Base System](https://reference.digilentinc.com/reference/programmable-logic/zybo/start)およびLinaroで配布されている[Ubuntu](http://releases.linaro.org/ubuntu/images/gnome/latest/)を使用して，ZYBO上でUbuntuを動作させるための手順を示す。

## ホストマシン環境
* Ubuntu 14.04 Desktop (64bit)
* Vivado Design Suite 2017.1

## 事前準備
LinuxカーネルおよびU-Bootをビルドするためのパッケージをインストールする。
```shell
$ sudo apt-get install build-essential git libssl-dev device-tree-compiler
```

ホストマシン（Ubuntu 64bit）上で32bitバイナリを実行できるようにするためのパッケージをインストールする
（Vivado Design Suiteが32bitバイナリを含むため）。
```shell
sudo dpkg --add-architecture i386
sudo apt-get update
sudo apt-get install libc6:i386 libncurses5:i386 libstdc++6:i386
```

Vivado Design Suiteに含まれるXilinx SDKをコマンドライン実行する場合，`gmake`を想定するため，`make`へのリンクを作成する。
```shell
$ sudo ln -s /usr/bin/make /usr/bin/gmake
```

## SDカードのパーティション作成
SDカードのパーティションを作成する。
なお，本手順は[FPGAの部屋の記事](http://marsee101.blog19.fc2.com/blog-entry-2818.html)を参考に行った。

ホストマシンにSDカードを挿入し，以下のコマンドでSDカードを検索する。
```shell
$ lsblk
```

以降では，/dev/sdbにSDカードがある場合を例に説明する。
まず，`fdisk`コマンドでSDカードのフォーマットを開始し，続いて表示されるプロンプトにコマンドを入力していく。
```shell
$ sudo fdisk /dev/sdb       # SDカードをフォーマット開始（/dev/sdbがSDカードの場合）
  > m                       # コマンドヘルプを表示
  > l                       # 既知のパーティションタイプを表示
  > p                       # 既存のパーティションを表示
  > d                       # パーティションを削除
  > p                       # パーティションが削除されていることを確認

# プライマリ・ディスクのパーティション番号1に64MBの領域を確保
  > n                       # 新規パーティションを作成
  > p                       # primaryパーティションを指定
  > 1                       # パーティション番号を1にセット
  > [return key]            # 最初のセクタ
  > +64M                    # 64MBの領域
  > p                       # 作成されたことを確認

# プライマリ・ディスクのパーティション番号2に約7.7GBの領域を確保
  > n                       # 新規パーティションを作成
  > p                       # primaryパーティションを指定
  > 2                       # パーティション番号を2にセット
  > [return key]            # 最初のセクタ
  > [return key]            # Lastのセクタ
  > p                       # 作成されたことを確認  

# プライマリ・ディスクのパーティション番号1をFAT32にして、ブート可能フラグを付加
  > t                       # tコマンド
  > 1                       # パーティション番号を1にセット
  > b                       # W95 FAT32
  > a                       # ブート可能フラグを付加
  > 1                       # パーティション番号を1にセット
  > p                       # ブート可能フラグの付加を確認

# SDカードへの書き込み
  > w                       # 書き込み

$ sudo mkfs.msdos -n ZYBO_BOOT /dev/sdb1
$ sudo mkfs.ext4 -L ROOT_FS /dev/sdb2
```

※最後のSDカードへの書き込みはおそらく失敗するため，一度マウントを解除して再接続して，もう一度「w」を実行する。

## SDカードの第1パーティション

### 作業フォルダ
作業フォルダを作成する。フォルダ名は任意。変更した場合は，適宜読み替えること。
```shell
$ mkdir ~/zybo
```

### 環境変数
Vivado Design Suiteのインストールに含まれるスクリプトを実行する。
```shell
$ source /opt/Xilinx/Vivado/2017.1/settings64.sh
```

### U-Boot
Xilinx社のGitHubリポジトリから，U-Bootをクローンする。
```shell
$ cd ~/zybo
$ git clone https://github.com/Xilinx/u-boot-xlnx.git
```

作成されたフォルダへ移動し，ビルドする。
```shell
$ cd u-boot-xlnx
$ make CROSS_COMPILE=arm-xilinx-linux-gnueabi- zynq_zybo_config
$ make CROSS_COMPILE=arm-xilinx-linux-gnueabi-
$ cp u-boot u-boot.elf
```

以下のファイルが生成される。
* ~/zybo/u-boot-xlnx/u-boot.elf

### Linuxカーネル
Xilinx社のGitHubリポジトリから，Linuxカーネルをクローンする。
```shell
$ cd ~/zybo
$ git clone https://github.com/Xilinx/linux-xlnx.git
```

作成されたフォルダへ移動する。
```shell
$ cd linux-xlnx
```

SDカードの第2パーティションをルートファイルシステムとするため，zynq-zybo.dtsの内容を変更する。
```shell
$ vi arch/arm/boot/dts/zynq-zybo.dts

# 変更前
bootargs = "";
# 変更後
bootargs = "console=ttyPS0,115200 root=/dev/mmcblk0p2 rw earlyprintk rootfstype=ext4 rootwait devtmpfs.mount=1";
```

そのままでは時計のずれが発生するとのことなので，[FPGAの部屋](http://marsee101.blog19.fc2.com/blog-entry-2994.html)の手順に従い，zynq-7000.dtsiを変更する。
```shell
$ vi arch/arm/boot/dts/zynq-7000.dtsi

# 変更前
operating-points = <
	/* kHz    uV */
	666667  1000000
	333334  1000000
>;
# 変更後
operating-points = <650000 1000000>;
```

uImageの作成にmkimageコマンドを使用するため，パスを通す。
```shell
$ export PATH=$PATH:~/zybo/u-boot-xlnx/tools
```

Linuxカーネルをビルドする。
```shell
$ make ARCH=arm CROSS_COMPILE=arm-xilinx-linux-gnueabi- xilinx_zynq_defconfig
$ make ARCH=arm CROSS_COMPILE=arm-xilinx-linux-gnueabi-
$ make ARCH=arm CROSS_COMPILE=arm-xilinx-linux-gnueabi- UIMAGE_LOADADDR=0x8000 uImage
$ make ARCH=arm CROSS_COMPILE=arm-xilinx-linux-gnueabi- modules
$ make ARCH=arm CROSS_COMPILE=arm-xilinx-linux-gnueabi- modules_install INSTALL_MOD_PATH=arch/arm/boot/modules_install
$ cp arch/arm/boot/dts/zynq-zybo.dtb arch/arm/boot/devicetree.dtb
```

以下のファイルが生成される。
* ~/zybo/linux-xlnx/arch/arm/boot/uImage
* ~/zybo/linux-xlnx/arch/arm/boot/devicetree.dtb
* ~/zybo/linux-xlnx/arch/arm/boot/modules_install

### FSBL
作業フォルダを作成する。
```shell
$ cd ~/zybo
$ mkdir fsbl-build
```

ここで，*.hdf を fsbl-build フォルダへコピーする。
次に，hsiコマンドを実行して，FSBLを生成する。
```shell
$ cd fsbl-build
$ hsi
hsi% open_hw_design system_wrapper.hdf
hsi% generate_app -hw system_wrapper -os standalone -proc ps7_cortexa9_0 -app zynq_fsbl -compile -sw fsbl -dir fsbl-build
hsi% exit
```

以下のファイルが生成される。
* ~/zybo/fsbl-build/fsbl-build/executable.elf

### Boot Image
作業フォルダを作成する。
```shell
$ cd
$ mkdir boot-image
```

boot.bifを作成し，bootgenコマンドを使用してBoot Image（boot.bin）を作成する。
```shell
$ echo -e "the_ROM_image:\n{\n\t[bootloader]`pwd`/fsbl-build/fsbl-build/executable.elf\n\t`pwd`/fsbl-build/system_wrapper.bit\n\t`pwd`/u-boot-xlnx/u-boot.elf\n}" > boot-image/boot.bif
$ cd boot-image
$ bootgen -image boot.bif -w on -o i boot.bin
```

以下のファイルが生成される。
* ~/zybo/boot-image/boot.bin

### uEnv.txt
U-Boot起動時に読み込むファイルを作成する。
```
$ cd ~/zybo
$ echo 'uenvcmd=fatload mmc 0 0x03000000 uImage && fatload mmc 0 0x02A00000 devicetree.dtb && bootm 0x03000000 - 0x02A00000' > uEnv.txt
```

以下のファイルが生成される。
* ~/zybo/uEnv.txt

### SDカードへ格納
以下のファイルをSDカードの第1パーティションへ格納する。
* ~/zybo/boot-image/boot.bin
* ~/zybo/uEnv.txt
* ~/zybo/linux-xlnx/arch/arm/boot/uImage
* ~/zybo/linux-xlnx/arch/arm/boot/devicetree.dtb
* ~/zybo/linux-xlnx/arch/arm/boot/modules_install/lib/modules/<kernel_ver>

※<kernel_ver>は，Linuxカーネルのバージョンを示す文字列。

※~/zybo/linux-xlnx/arch/arm/boot/modules_install/lib/modules/<kernel_ver>はブートに必要なものではないため，最終的に削除する。

## SDカードの第2パーティション
(T.B.D.)

## ZYBOを起動
Micro USBケーブルでPCと接続し，作成したSDカードをZYBOに挿入して，
ボードのJP5をSD側でショートさせた状態で，電源をONするとSDカードからブートを開始する。
以下のプロンプトが表示されれば，起動に成功している。
```
root@linaro-gnome:~#
```

以下の手順で，ドライバ類をZYBOの/lib/modulesにコピー（または移動）する。
<kernel_ver>は，作成したカーネルのバージョンを示す文字列に置き換えること。
```
root@linaro-gnome:~# mount /dev/mmcblk0p1 /mnt
root@linaro-gnome:~# cd /mnt
root@linaro-gnome:/mnt# mv <kernel_ver> /lib/modules
root@linaro-gnome:/mnt# depmod -a
root@linaro-gnome:/mnt# reboot
```
