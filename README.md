# Arch Linux 安裝攻略

## 說明

針對已經對 GNU/Linux 有基本認識者編寫的輔助性質 cheatsheet

## 安裝

Arch Linux 與其他大多數發行版不同，沒有一個專門的 installer ，只有 bootstrap 工具( pacstrap )。開機進 Arch Linux 的 ISO 後，會進入到一個有 zsh 的 kernel console ，需要手動安裝。
如果沒有辦法進入，可能需要停用 secure boot。

以下安裝過程皆假設使用 UEFI ， legacy BIOS 會不同的地方主要只會有安裝 grub 的部份。

### HiDPI

如果你的 DPI 高到看不到 Linux kernel console 的字，就 load 一個比較大的 font，目前最大的是 `latarcyrheb-sun32`

```shell
setfont /usr/share/kbd/consolefonts/latarcyrheb-sun32.psfu.gz
```

### 驗證開機模式

在 UEFI 模式下，會存在目錄 /sys/firmware/efi/efivars ，如果想確保目前是在 UEFI 下，可以用他來確認

```shell
ls /sys/firmware/efi/efivars
```

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

如果是 wifi ，可以利用 netctl 的 wifi-menu

```shell
wifi-menu
```

然後不管是有線還是無線，確保有透過 DHCP 拿到 ip

```shell
ip a <interface>
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

如果 DHCP 設定 DNS 太爛或是需要手動設定

```shell
vi /etc/resolv.conf
```

增加

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

在 linux 中 device nodes 位於 /dev 底下，其中 block devices 位於 /dev 或 /dev/block ，在 Arch 為前者，舉例來說透過運行 lsblk 後，得知固態硬碟名稱為 nvme0n1 ，他的 device node 位置便是 /dev/nvme0n1

其中常見 block devices 的命名規則如下

SATA 或 USB: `sd<x><y>` ，其中 x 為英文字母，表示第 x 顆硬碟， y 為數字，表示硬碟上的第 y 個分區

IDE 介面: `hd<x><y>` ，其中 x 為英文字母，表示第 x 顆硬碟， y 為數字，表示硬碟上的第 y 個分區
 
NVMe 介面: `nvme<x>n<y>p<z>` ，其中 x, y, z 為數字， `nvme<x>n<y>` 表示硬碟， `p<z>` 表示分區
 
MMC: `mmcblk<x>p<y>` ，其中 x, y 為數字， x 表示碟， `p<y>` 表示分區

以最常見的第一顆 SATA 介面硬碟分區名稱 /dev/sda<y> 為例，並假設硬碟是空的，開始設定分區

```shell
cfdisk /dev/sda
```

* /dev/sda1: /boot
  * **空間通常建議 512MB，類型為 EFI System**
  * (若有其他系統的 EFI 分區可以直接沿用，且不要格式化，格式化你其他系統的 bootloader 就沒了)
  * 我的系統上 Windows 的 bootloader 再加上 Dell 的一些 recovery system 再加上 grub 和 Arch Linux 的 kernel 和 initramfs 用了快 150MB 供參考
  * 我會最少也弄個 256MB 省麻煩

* /dev/sda2: Swap
  * **自訂，類型為 Linux Swap**
  * swap 分區是用來儲存部份原本應該在 RAM 上的資訊。如果你覺得你的 RAM 大小足夠，可能不需要這個分區，也可以事後使用基於檔案的 swap 。

* /dev/sda3: /
  * **自訂，可以使用全部剩餘空間，類型為 Linux filesystem**

如果需要調整現有分區的大小，切記要先 resize filesystem ( ext 系列是 resize2fs )再去 resize partition。如果怕出錯，可以用 gparted 懶人工具。Arch Linux live 環境通常會塞不下 gparted (需要 X11 )，可以改用 gparted 官方自己的 live system。

### 格式化磁區

```shell
mkfs -t vfat /dev/sda1
mkswap /dev/sda2
mkfs -t ext4 /dev/sda3
```

### 掛載磁區

```shell
mount /dev/sda3 /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

## 安裝

有些人可能會使用別人寫好的腳本，但這很不 Arch

以下為手動安裝

### 設定 pacman 的 mirrorlist

調整 pacman 啟用的鏡像站，提高下載安裝的速度。

```shell
vi /etc/pacman.d/mirrorlist
```

把非 Taiwan 的註解或刪掉，把 Taiwan 的視所在位置排序

### 安裝 base 和 base-devel group packages 

如果想要更小的系統不寫程式可能不需要 `base-devel`

這裡會自動刷新一次 repo db 然後下載最新的套件

```shell
pacstrap /mnt base base-devel
```

### 建立 fstab

接下來我們要生成一個 fstab 文件，其中 -U 代表透過 UUID 來定義，就算 device nodes 的標籤改變了也能順利使用，他定義了各個分區如何掛載於系統

```shell
genfstab -U /mnt >> /mnt/etc/fstab
```

### chroot 至新系統

chroot 是更改系統根目錄的位置

```shell
arch-chroot /mnt
```

### HiDPI

如果之後還會繼續用 kernel console，可以讓他自動載入 font

/etc/vconsole.conf:
```
FONT=latarcyrheb-sun32
```

然後讓他在 initramfs 的階段就載入

/etc/mkinitcpio.conf:
HOOKS 增加 consolefont

```shell
mkinitcpio -p linux -k <kernel ver in chroot>
```

注意 pacstrap 裝的 kernel 是 repo 上新的，可能跟正在跑的不同，所以要 `-k` 指定，觀察 `/lib/modules` 得知版本

### 設定時區

```shell
ln -sf /usr/share/zoneinfo/Asia/Taipei /etc/localtime
```

### 設定語言環境

生成 `zh_TW.UTF-8` 語系

```shell
echo "zh_TW.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
```

設定預設為 `zh_TW.UTF-8`

```shell
echo "LANG=zh_TW.UTF-8" > /etc/locale.conf
```

在 kernel console 底下無法直接顯示中文，使用 `zh_TW.UTF-8` 會出現一堆方塊，如果常直接在 kernel console 下做事可以在當下 `export LC_ALL="C"` 暫時修改，也可以只在 xinitrc 之類的地方設定為 `zh_TW.UTF-8`

如果不是用 xinit (例如 Wayland)，可以善用 shell 的 alias ，寫進 shell 的 config ，例如：

```shell
alias sway="env LC_ALL=zh_TW.UTF-8 sway"
```

### 設定電腦名稱

```shell
echo "<your-pc-name>" > /etc/hostname
```

```shell
vi /etc/hosts
```

在 /etc/hosts 中加入最後一行

```shell
127.0.0.1  localhost.localdomain       localhost
::1        localhost.localdomain       localhost
127.0.0.1  <your-pc-name>.localdomain  <your-pc-name>
```

### 設定 root 密碼

在後面加入一般 user 之後可以透過 `passwd -l root` 防止使用 root 登入，但那會造成無法進入 emergency shell ，如果沒有其他救援備案(例如從其他 Linux chroot 進來)鎖定與否自行斟酌，不鎖就設一下密碼

```shell
passwd
```

### 安裝 grub 啟動載入程式

```shell
pacman -S grub os-prober efibootmgr
```

os-prober 可以用以偵測其他系統(如 Windows )，並加入 grub 選單中，在 grub-mkconfig 內會自動執行，如果只有裝 Arch 就不用安裝

```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg
```

如果之後開機沒有載入 grub 而是載入了其他系統的 bootloader ，先檢查 `/boot/EFI/Boot/Bootx64.efi` 是否與 `/boot/grub/grubx64.efi` 相同，注意在 FAT 系列格式下大小寫不拘。不會太舊的 UEFI 實做大多可以手動設定 `EFI/Boot/Bootx64.efi` 以外的路徑可以試試。 `EFI/Boot/Boot<arch>.efi` 是 UEFI 規範的 fallback 路徑。

### 安裝選用網路工具

```shell
pacman -S wireless_tools wpa_supplicant dialog
```

其中 wireless_tools wpa_supplicant dialog 只有要用 wifi 才需要， dialog 被 netctl 的 wifi-menu 功能需要

如果你不會用 iproute2 的 ip 指令， net-tools 提供了 ifconfig route 等舊指令

如果之後沒辦法連上網路，設定方面參考上面。如果也是使用 dhcpcd ，記得 enable 他的 service 。

### 建立新使用者

安裝 sudo

```shell
pacman -S sudo
```

設定 sudo 群組

```shell
vi /etc/sudoers
```

找到這行，並刪除 # 取消註解，讓 wheel 群組能用 sudo

```
# %wheel ALL=(ALL) ALL
```

建立新使用者，並加入 wheel 群組

```shell
useradd -m -G wheel <your-user-name>
passwd <your-user-name>
```

### 重新啟動進入新系統

```shell
exit
umount -R /mnt
reboot
```

## 初次進入系統

以下大多可以在 chroot 時就進行，但直接進系統比較會省麻煩

### 安裝 CPU Microcode

有些人可能會想跳過省下 Intel 的一堆漏洞的 mitigations 造成的 performance penalty

如果要再進一步關 mitigations，在 /etc/default/grub 的 GRUB_CMDLINE_LINUX_DEFAULT 多加一項 mitigations=off 然後 grub-mkconfig 一次

對於 mitigations 當前的狀況，可以看 /sys/devices/system/cpu/vulnerabilities/

詳細請參閱：[Microcode](<https://wiki.archlinux.org/index.php/Microcode_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)>)

#### AMD

對於 AMD 處理器，其 Microcode 更新以包在 linux-firmware 中，因此不需要額外動作

#### Intel

對於 Intel 處理器需要另外安裝套件，並且在 bootloader 啟用 Microcode 更新

```shell
sudo pacman -S intel-ucode
```

grub-mkconfig 時會自動加上載入 microcode 的參數，安裝完 intel-ucode 後，手動執行一次確保有被套用

```shell
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### 安裝顯示卡驅動

如果你有顯示卡的話，安裝針對顯示晶片的驅動，效能通常更好

在同時有獨顯和內顯的筆電上，如果有獨顯輔助內顯的功能(例如 NVIDIA Optimus )，預設螢幕會顯示內顯輸出

如果 Xorg 無法正常顯示，可能可以在 BIOS/UEFI 設定主要使用的顯示卡

如果不在意效能，也可以調一下 Xorg config 直接只用內顯

如果要達成混合顯示，參考 [PRIME(通用)](https://wiki.archlinux.org/index.php/PRIME) 或 [Bumblebee(NVIDIA)](https://wiki.archlinux.org/index.php/bumblebee)

#### NVIDIA

使用 NVIDIA 提供的 nvidia ，如果偏好開源可以跳過，原本預設會使用完全開源的 nouveau

注意 nvidia 沒有實做 GBM 只有實做自家出品的 EGLStreams ，基於 wlroots 的 Wayland compositors 不支援

```shell
sudo pacman -S nvidia
```

或者是 nvidia-lts

含有 kernel module ， modprobe 或重新啟動以載入

監控檢查狀況用 nvidia-smi 指令

需要時可以透過其提供的 nvidia-settings 圖形界面程式來調整設定

#### AMD

參閱 https://wiki.archlinux.org/index.php/AMD_Catalyst_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)

### 安裝 GUI

安裝你需要的桌面環境/ wm/ Wayland compositor 。選擇依賴於個人喜好故跳過。如果你不知道我在說什麼就不該裝 Arch Linux 。

### 安裝 AUR helper

AUR 是由社群推動的使用者軟體庫，包含了 PKGBUILD 等打包時需要的腳本，可以用 makepkg 打包軟體包，並透過 pacman 安裝。透過 AUR 可以在社群間分享、建構新軟體包，熱門的軟體有機會被收錄進 community 軟體庫。

AUR 沒有在管內容有沒有開源，很多都是 binary blobs ，風險自負，建議養成 review PKGBUILD 的習慣

如果想要使用 AUR 上的資源，需要確認有安裝 base-devel group 及 git 指令。然後使用 AUR helper 來打包 AUR 上的內容，或是手動用 makepkg 一一打包。

自行參閱[Arch AUR](<https://wiki.archlinux.org/index.php/Arch_User_Repository_(%E6%AD%A3%E9%AB%94%E4%B8%AD%E6%96%87)>) 以及[AUR helper](https://wiki.archlinux.org/index.php/AUR_helpers)頁面。

有一個我維護的 AUR 自動打包系統，自動把一些 AUR 套件打包並放在一個 pacman repo ， repo 網址 https://aurbuild.parto.nctu.me/ 如果想要加入套件，先確認 PKGBUILD 是好的，可以在他的 dependency 都滿足的情況下用 devtools 打包後聯絡 xdavidwuph@gmail.com 。如果這些看不懂或不知道怎麼加這個 repo 就不該用這個。部份套件的 PKGBUILD 有經過調整加入一些 compile-time 就決定的 features 。一樣，只提供自動打包，風險自負，如果新的 PKGBUILD 有問題就不一定是最新，我也不一定有興趣研究別人的 bug 。

### 安裝中文輸入法 ( fcitx )

安裝 fcitx

```shell
sudo pacman -S fcitx-im fcitx-chewing fcitx-configtool
```

```shell
sudo vi /etc/environment
```

在最後方添加以下三行

```shell
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS="@im=fcitx"
```

開啟 Fcitx Configuration 圖形界面新增 input method

找到 Chewing (新酷音)並新增

Wayland 的部份據我所知含選字 popup 的 [protocol](https://github.com/swaywm/wlroots/blob/master/protocol/input-method-unstable-v2.xml) 已經有 unstable 版本，但 popup 的部份還沒有已知實做。目前可以透過 Xwayland 用 toolkit im module ( GTK_IM_MODULE, QT_IM_MODULE ) 的方式使用 fcitx ，缺點是依賴於 Xwayland ，需要是使用 GTK 或 QT 才能使用，且因為 Wayland 沒有得到全域座標的方法(這是 by-design ，而且在某些特殊模型下全域座標可能沒意義或不存在)，選字 popup 的位置大多會不正確。

GNOME Wayland 因為有對 IBus 做特別整合，選字 popup 用 IBus 會正常運作。但應該是個別整合而不是通用的 protocol 。

## 安裝字型

```shell
sudo pacman -S noto-fonts noto-fonts-cjk
```

noto-fonts 支援大多數 Unicode 的字元

noto-fonts-cjk 為 Google 提供的免費的中日韓字型(Chinese Japanese Korean)，建議至少安裝這個

### NTFS 檔案系統讀寫支援

如果需要對 NTFS 有更好的支援，ntfs-3g 提供了以 FUSE 實做的驅動，以及對 NTFS 進行各種操作的指令

實際上如果只需要存取 NTFS 可以嘗試只用位於 Linux kernel 內的驅動

參見[NTFS-3G](https://wiki.archlinux.org/index.php/NTFS-3G)
以及[Linux kernel source](https://github.com/torvalds/linux/blob/master/fs/ntfs/Kconfig)

```shell
sudo pacman -S ntfs-3g
```

## Enjoy your new system

Done！
