# Arch Linux 安裝攻略

## 說明

針對已經對 GNU/Linux 有基本認識者編寫的輔助性質 cheatsheet

## 先備動作

### Windows

有些機子可能會預設啟用 BitLocker ，但那需要 secure boot ，而且待會需要停用 secure boot ，事先把 BitLocker 停用，才不會進不了 Windows 。如果還是想要保留 secure boot ，參考 [Secure Boot](https://wiki.archlinux.org/index.php/Secure_Boot)

Windows 會把 UEFI/BIOS 的時間設做當地時間，但其他系統通常是用 UTC 時間。創一個 registry `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\TimeZoneInformation\RealTimeIsUniversal` (DWORD) 然後設為 1，重新啟動後 Windows 就改成用 UTC 時間了

### UEFI Settings

這些功能可能可以幫你省下狂按按鍵進 UEFI 設定或開機選單的麻煩

* Windows 的 `進階啟動` (reboot 時壓住 shift)

也可以用來選 boot device 暫時充當開機選單

* GRUB command `fwsetup`

按 c 進入 command line

常用可以加成一個 entry

* systemd-boot `Reboot to firmware`

以 loader.conf `auto-firmware` 控制，預設是 enabled

注意某些 UEFI/BIOS 會把持續壓住的鍵視為故障而不進設定

## 安裝

Arch Linux 與其他大多數主流發行版不同，安裝主要依靠 bootstrap 工具 (pacstrap) 安裝套件後手動設定。開機進 Arch Linux 的 ISO 後，會進入到一個有 zsh 的 kernel console，再進行手動安裝

UEFI 需要停用 secure boot

以下安裝過程皆假設使用 UEFI，legacy BIOS 會不同的地方主要只會有安裝 bootloader 的部份

### HiDPI

如果你的 DPI 高到看不到 Linux kernel console 的字，就 load 一個比較大的 font，目前 ISO 內最大的是 `latarcyrheb-sun32`

```shell
setfont /usr/share/kbd/consolefonts/latarcyrheb-sun32.psfu.gz
```

### 驗證開機模式

UEFI 模式下，路徑 `/sys/firmware/efi` 會存在，如果想確保目前是在 UEFI 下，可以用他來確認

### 設定網路連線

目標是能 ping 到 google.com ，在絕大多數情況下就是成功了

```shell
ping www.google.com
```

如果是有線連線，先確保該 interface 有 up

```shell
ip l set <interface> up
```

如果要列出所有 interfaces

```shell
ip l
```

如果是 wifi ，可以利用 iwd (含內建 DHCP client)

連線方法: iwctl 進入 interactive command line

掃 AP:

```iwctl
station <interface> scan
station <interface> get-networks
```

連 AP: (802.1x 要寫 config, 見 `iwd.network(5)`)

```iwctl
station <interface> connect <ssid>
```

如果是有線，確保有透過 DHCP 拿到 ip

```shell
ip a show <interface>
```

如果沒有，確保有啟動 dhcpcd

```shell
systemctl start dhcpcd.service
```

如果還是沒有，手動設定 ip 和 routing table

```shell
ip a add <ip> dev <interface>
ip r add <subnet> dev <interface>
ip r add default via <gateway> dev <interface>
```

如果 DHCP 設定 DNS 太爛或是需要手動設定，於 /etc/resolv.conf 增加

```
nameserver <dns server>
```

這是暫時的，會被 dhcpcd 覆蓋，如果需要永久設定可以改成加入 `/etc/resolv.conf.head`

如果有程式沒有在查詢失敗時嘗試下一個 server ，加上 `options rotate` 有支援它的 applications 就會自動嘗試

### 分割硬碟分區

在開始分割硬碟區前先確認硬碟有正確讀到

```shell
lsblk -a
```

上面這個指令可以列出硬碟代號及大小，如果需要更詳細的資料

```shell
fdisk -l
```

在 Linux 中 device nodes 位於 /dev 底下，其中 block devices 位於 /dev 或 /dev/block ，在 Arch 為前者，舉例來說透過運行 lsblk 後，得知固態硬碟名稱為 nvme0n1 ，他的 device node 位置便是 /dev/nvme0n1

其中常見 block devices 的命名規則如下

SATA 或 USB: `sd<x><y>` ，其中 x 為英文字母，表示第 x 顆硬碟， y 為數字，表示硬碟上的第 y 個分區

IDE 介面: `hd<x><y>` ，其中 x 為英文字母，表示第 x 顆硬碟， y 為數字，表示硬碟上的第 y 個分區
 
NVMe 介面: `nvme<x>n<y>p<z>` ，其中 x, y, z 為數字， `nvme<x>n<y>` 表示硬碟， `p<z>` 表示分區
 
MMC: `mmcblk<x>p<y>` ，其中 x, y 為數字， x 表示碟， `p<y>` 表示分區

以第一顆 SATA 介面硬碟分區名稱 /dev/sda<y> 為例，並假設硬碟是空的，開始設定分區

```shell
cfdisk /dev/sda
```

* /dev/sda1: /boot

空間通常建議 512MB，類型為 EFI System

(若有其他系統的 EFI 分區可以直接沿用，且不要格式化，格式化你其他系統的 bootloader 就沒了)

我的系統上 Windows 的 bootloader 再加上 Dell 的一些 recovery system 再加上 grub 和 Arch Linux 的 kernel 和 initramfs 用了快 150MB

我會最少也弄個 256MB 省麻煩

* /dev/sda2: Swap

自訂，類型為 Linux Swap

swap 分區是用來儲存部份原本應該在 RAM 上的資訊。如果你覺得你的 RAM 大小足夠，可能不需要這個分區，也可以事後使用基於檔案的 swap

* /dev/sda3: /

自訂，可以使用全部剩餘空間，類型為 Linux filesystem

如果需要調整現有分區的大小，切記要先 resize filesystem (ext 系列是 resize2fs) 再去 resize partition。如果怕出錯，可以用 gparted GUI 懶人工具。Arch Linux live 環境通常會塞不下 gparted ，可以改用 gparted 官方自己的 live system。

### 格式化分區

```shell
mkfs -t vfat /dev/sda1
mkswap /dev/sda2
mkfs -t ext4 /dev/sda3
```

### 掛載分區

```shell
mount /dev/sda3 /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

## 安裝

有些人可能會使用別人寫好的腳本，但這很不 Arch

以下為手動安裝

### 設定 pacman 的 mirrorlist

調整 pacman 啟用的鏡像站，提高下載安裝的速度

於 `/etc/pacman.d/mirrorlist` ，把 Taiwan 的移到前面，視所在位置排序

### 安裝 base metapackage 和 base-devel group

如果想要更小的系統不做開發可能不需要 `base-devel`

這裡會自動刷新一次 repo db 然後下載最新的套件

```shell
pacstrap /mnt base base-devel
```

注意 base 已經不再是 group 而是 metapackage，如果想要到非常精簡建議 `pacman -Si base` 看看他 depends 哪些自己過濾裝

### 建立 fstab

生成 fstab ，其中 -U 代表透過 UUID 來定義，就算 device nodes 的標籤改變了也能順利使用，他定義了各個分區如何掛載於系統

```shell
genfstab -U /mnt >> /mnt/etc/fstab
```

### chroot 至新系統

chroot 是以其他目錄作為系統根目錄執行指令的概念， arch-chroot 這個 helper 會幫忙 setup 更多東西，讓環境近似於直接開進去的系統

```shell
arch-chroot /mnt
```

### HiDPI

如果之後還會繼續用 kernel console，可以讓他自動載入 font

/etc/vconsole.conf:

```
FONT=latarcyrheb-sun32
```

加入之後在進入系統時會被 systemd 載入，如果有需要可以讓他在 initramfs 的階段就載入

/etc/mkinitcpio.conf:

HOOKS 增加 consolefont 後執行 `mkinitcpio -p linux`

又或者日後可以自己編譯 Linux kernel，把 kernel 內建的 font 改為較大的，最大有到 16x32:

```
Library routines
 Select compiled-in fonts
  Terminus 16x32 font (not supported by all drivers)
```

如果喜歡 [Terminus](http://terminus-font.sourceforge.net/)，有 terminus-font package 可以提供 console font，命名規則為 ter-<mapping><size><style>，其中 mapping 以 v 最廣，size 最大 32，style 有 n (normal) 和 b (bold)，推薦 ter-v32b，Terminus 有不少字有官方變種，有各種變種組合，個人推薦 ll2+td1，AUR 套件 terminus-font-ll2-td1

### 設定時區

```shell
timedatectl set-timezone Asia/Taipei
```

### 設定語言環境

生成 `zh_TW.UTF-8` 語系

```shell
echo "zh_TW.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
```

設定預設為 `zh_TW.UTF-8`

```shell
localectl set-locale zh_TW.UTF-8
```

在 kernel console 底下無法直接顯示中文，使用 `zh_TW.UTF-8` 會出現一堆方塊，如果常直接在 kernel console 下做事可以在當下 `export LC_ALL="C"` 暫時修改，也可以只在 xinitrc 之類的地方設定為 `zh_TW.UTF-8`

如果不是用 xinit (例如 Wayland)，可以善用 shell 的 alias，寫進 shell 的 config，例如：

```shell
alias sway="env LC_ALL=zh_TW.UTF-8 sway"
```

### 設定電腦名稱

```shell
hostnamectl set-hostname <your-pc-name>
```

### 設定 root 密碼

在後面加入一般 user 之後可以透過 `passwd -l root` 防止使用 root 登入，但那會造成無法進入 emergency shell ，如果沒有其他救援備案 (例如從其他 Linux chroot 進來，或者 kernel cmdline 用 init=/bin/sh 繞過) 鎖定與否自行斟酌，不鎖就設一下密碼

```shell
passwd
```

### 安裝 bootloader

這裡介紹 systemd-boot 和 GRUB

GRUB:

* 顏值高，可 theme，功能多，支援執行多種 OS kernel
* 相較之下偏大 (x86\_64-efi) EFI binary + modules 約 3.3M
* Arch 套件不小 (約 32M when installed)
* 實測在 4k 螢幕上輸出微慢

systemd-boot:

* Arch 上就在 systemd package 裡，裝了 systemd 就順便送你
* 功能少，只能 load EFI binary

Linux kernel 需要開 `CONFIG_EFI_STUB` 支援以 EFI binary 載入 (Arch `linux` 套件有開)

entries 要自己寫或生成 unified kernel image

* 本文採用自己寫 entries，就不用每次更新 kernel 重新生成 unified image
* 輸出直接交給 EFI，不能 font 或用背景圖

HiDPI 上字小或字醜二選一

但通常跟 UEFI 自己的 boot menu 很搭

* 很小，EFI binary 92K
* 預設就會自己抓 Windows, OS X 的 bootloader 長出 entries
* 預設自己會長出 `Reboot to firmware`


如果之後開機載入了其他系統的 bootloader ，先檢查 `/boot/EFI/Boot/Bootx64.efi` 是否與 `/boot/EFI/systemd/systemd-bootx64.efi` 或 `/boot/EFI/grub/grubx64.efi` 相同，注意在 FAT 系列格式下大小寫不拘。不會太舊的 UEFI 實做大多可以手動設定 `EFI/Boot/Bootx64.efi` 以外的路徑可以試試。 `EFI/Boot/Boot<architecture>.efi` 是 UEFI 規範的 fallback 路徑

* systemd-boot

```shell
bootctl install
```

注意 `/boot/EFI/Boot/Bootx64.efi` 會被覆寫

設定:

main config `/boot/loader/loader.conf` 常見 options:

```
timeout <timeout-seconds>
default <default entry name>
console-mode <0,1,2...,auto,max,keep (UEFI console resolution, 0: 80x25, 1:80x50, 2...: non-standard)>
```

entry definition `/boot/loader/entries/<name>.conf`:

```
title <title>
linux <EFI stub kernel path>
initrd <initramfs path>
options <kernel cmdline>
```

範例:

main config `/boot/loader/loader.conf`:

```
timeout 3
default arch
console-mode 2
```

entry definition `/boot/loader/entries/arch.conf`:

```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=UUID=<root UUID>
```

entry definition `/boot/loader/entries/arch-fallback.conf`:

```
title Arch Linux, fallback initramfs
linux /vmlinuz-linux
initrd /initramfs-linux-fallback.img
options root=UUID=<root UUID>
```

* GRUB

```shell
pacman -S grub os-prober efibootmgr
```

os-prober 可以用以偵測其他系統 (如 Windows)，並加入 grub 選單中，在 grub-mkconfig 內會自動執行，如果只有裝 Arch 就不用安裝

```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg
```

注意不管是哪種， Arch 官方的 package 在更新時都不會幫你順便更新 `/boot` 。如果想要達到自動更新可以採用 pacman hook ， AUR 上有現成的 `systemd-boot-pacman-hook` 和 `grub-hook`

### 安裝 Wi-Fi 連線工具 (iwd)

大量利用 kernel crypto API 的輕量 wireless daemon，自行編譯 kernel 者注意相關 configuration (如果有缺在 log 會有提示)

內建 DHCP client 功能

```shell
pacman -S iwd
```

有 daemon 要先起來，systemd unit iwd.service

預設會自己處理 interface，創成 wlan0 這種 naming，可以在 daemon 加參數避免，enable 的話注意他起來時 interface 有沒有先出現

### 其餘網路工具

如果你不會用 iproute2 的 ip 指令， net-tools 提供了 ifconfig route 等舊指令

如果之後沒辦法連上網路，設定方面參考上面。如果也是使用 dhcpcd ，記得 enable 他的 service

有線可以考慮不裝 dhcpcd ，使用 systemd-networkd 去連，但是會需要寫 config 於 `/etc/systemd/network/*.network`

範例：

```systemd
[Match]
Name=<interface name>

[Link]
RequiredForOnline=no

[Network]
DHCP=yes
```

`RequiredForOnline` 定義這個 network 在 systemd 判斷有沒有 online 會不會需要他，筆電可以設成 no ，不然開機過程大多會等 online

### 安裝 CPU Microcode

安裝 intel-ucode 或 amd-ucode 套件

有些人可能會想跳過省下 Intel 的一堆漏洞的 mitigations 造成的 performance penalty

對於 mitigations 當前的狀況，可以看 `/sys/devices/system/cpu/vulnerabilities/`

如果要再進一步關 mitigations，在 kernel cmdline 增加 mitigations=off

cmdline 調整:

* systemd-boot

修改 entries 的 options

* GRUB

修改 /etc/default/grub 的 GRUB\_CMDLINE\_LINUX\_DEFAULT 然後 `grub-mkconfig -o /boot/grub/grub.cfg`

套用 microcode:

* systemd-boot

在 entries 多加一項 initrd 為 `/boot/intel-ucode.img` 或 `/boot/amd-ucode.img`

* GRUB

grub-mkconfig 時會自動加上載入 microcode 的參數，安裝完 microcode 後，手動執行一次 `grub-mkconfig -o /boot/grub/grub.cfg`


### 建立新使用者

建立新使用者，並加入 wheel 群組

```shell
useradd -m -G wheel <your-user-name>
passwd <your-user-name>
```

安裝 sudo (包含在 base-devel group 裡面)

```shell
pacman -S sudo
```

設定 sudo 群組

```shell
visudo
```

找到這行，並刪除 # 取消註解，讓 wheel 群組能用 sudo

```
# %wheel ALL=(ALL) ALL
```

### 重新啟動進入新系統

```shell
exit
umount -R /mnt
reboot
```

## 初次進入系統

以下大多可以在 chroot 時就進行，但直接進系統比較會省麻煩，才不會有比如說哪個 kernel module 裝下去跟正在跑的 kernel 不符用不了

### 安裝顯示卡驅動

如果你有顯示卡的話，安裝針對顯示晶片的驅動，效能通常更好

在同時有獨顯和內顯的筆電上，如果有獨顯輔助內顯的功能 (例如 NVIDIA Optimus) ，預設螢幕會顯示內顯輸出，可能可以在 BIOS/UEFI 設定主要使用的顯示卡

如果不在意效能，也可以調一下 Xorg config 直接只用內顯

如果要達成混合顯示，參考 [PRIME](https://wiki.archlinux.org/index.php/PRIME)

* NVIDIA

使用 NVIDIA 提供的 nvidia ，如果偏好開源可以跳過，原本預設會使用開源的 nouveau

注意 nvidia 沒有實做 GBM 只有實做自家出品的 EGLStreams ，基於 wlroots 的 Wayland compositors 不支援

安裝套件 nvidia 或者是 nvidia-lts

含有 kernel module ， nvidia-modprobe 或重新啟動以載入

監控檢查狀況用 nvidia-smi 指令 (套件 nvidia-utils)

需要時可以安裝 nvidia-settings 圖形界面程式來調整設定

* AMD

參閱 [AMDGPU](https://wiki.archlinux.org/index.php/AMDGPU)

### 安裝 GUI

安裝你需要的桌面環境 / wm / Wayland compositor 。選擇依賴於個人喜好故跳過。如果你不知道我在說什麼就不該裝 Arch Linux

### 安裝 AUR helper

AUR 是由社群推動的使用者軟體庫，包含了 PKGBUILD 等打包時需要的腳本，可以用 makepkg 打包軟體包，並透過 pacman 安裝。透過 AUR 可以在社群間分享、建構新軟體包，熱門的軟體有機會被收錄進 community 軟體庫

AUR 沒有在管內容有沒有開源，很多都是 binary blobs ，風險自負，建議養成 review PKGBUILD 的習慣

如果想要使用 AUR 上的資源，需要確認有安裝 base metapackage, base-devel group 及 git 。然後使用 AUR helper 來打包 AUR 上的內容，或是手動用 makepkg 一一打包

如果要手動打包，大致上的流程是遞迴找出目標套件和所有他在 AUR 上的 dependencies ，從沒有 depend 到 AUR 套件的開始以 `makepkg -si` 打包回去，`-s` 會用 pacman 安裝缺少的 dependencies ，`-i` 是在打包完後自動安裝

自行參閱 [Arch User Repository](<https://wiki.archlinux.org/index.php/Arch_User_Repository>) 以及 [AUR helpers](https://wiki.archlinux.org/index.php/AUR_helpers) 頁面

有一個我維護的 AUR 自動打包系統，自動把一些 AUR 套件打包並放在一個 [pacman repo](https://aurbuild.eglo.ga/) ， 如果想要加入套件，先確認 PKGBUILD 是好的，可以在他的 dependency 都滿足的情況下用 devtools 打包後聯絡 xdavidwuph@gmail.com 。如果這些看不懂或不會查怎麼加這個 repo 就不該用這個。部份套件的 PKGBUILD 有經過調整加入一些 compile-time 就決定的 features 。只提供自動打包，風險自負，如果新的 PKGBUILD 有問題就不一定是最新，我也不一定有興趣研究別人的 bug 。套件有用 PGP key `F73F137D4573DEFAA097DBF09544CFF6B08A3FD3` 簽名

如果有興趣 contribute to PKGBUILDs ，推薦可以自己包個含 `base-devel` 並且有設自己的 AUR 套件 repo 的 container image，並且使用 container 去確保 dependency 沒有顯著問題

### 安裝中文輸入法 (fcitx)

安裝 fcitx

```shell
sudo pacman -S fcitx-im fcitx-chewing fcitx-configtool
```

在 /etc/environment 添加以下三行

```shell
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS="@im=fcitx"
```

想辦法讓你的 GUI 自動執行 fcitx-autostart

開啟 Fcitx Configuration 圖形界面 (fcitx-configtool) 新增 input method

找到 Chewing (新酷音) 並新增

Wayland 的部份輸入法 [protocol](https://github.com/swaywm/wlroots/blob/master/protocol/input-method-unstable-v2.xml) 已經有 unstable 版本，但還沒有已知的完全實做。目前可以透過 Xwayland 用 toolkit im module (GTK\_IM\_MODULE, QT\_IM\_MODULE) 的方式使用 fcitx ，缺點是依賴於 Xwayland ，需要是使用 GTK 或 QT 才能使用，且因為 Wayland 沒有取得畫面全域座標的方法 (這是 by-design，而且在某些特殊模型下全域座標可能沒意義或不存在) ，選字 popup 的位置大多會不正確

GNOME Wayland 因為 gnome-shell 有對 IBus 做選字 popup，IBus 會正常運作。但是是個別整合而不是通用的 protocol

## 安裝字型

```shell
sudo pacman -S noto-fonts noto-fonts-cjk
```

noto-fonts 是 Google 提供的開放 (OFL) 字型，支援大多數 Unicode 的字元

noto-fonts-cjk 為相同系列的中日韓字型 (Chinese, Japanese, Korean)，建議至少安裝這個

### NTFS 檔案系統讀寫支援

如果需要對 NTFS 有更好的支援，ntfs-3g 提供了以 FUSE 實做的驅動，以及對 NTFS 進行各種操作的指令

實際上如果只需要存取 NTFS 可以嘗試使用位於 Linux kernel 內的驅動

參見 [NTFS-3G](https://wiki.archlinux.org/index.php/NTFS-3G) 以及 [Linux kernel source](https://github.com/torvalds/linux/blob/master/fs/ntfs/Kconfig)

```shell
sudo pacman -S ntfs-3g
```

## Enjoy your new system

Done！
