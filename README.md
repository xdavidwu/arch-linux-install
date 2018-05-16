# Arch Linux 安裝教學

## 說明

[參考自 Arch Wiki](https://wiki.archlinux.org/index.php/installation_guide)
文章貢獻感謝原作者 Cheng-Yi Hong 以及 Arch Linux Taiwan 社群人員對本文之貢獻以及轉發

## 安裝

一旦你準備好了開機隨身碟，也在 UEFI settings 中使用 UEFI 模式(如果支援建議使用)，並且選擇以 usb 開機後，一般來說你可以毫無意外得進入 Arch ISO 的 shell 畫面中，也就是一個用來安裝 Arch 的 live 系統，會進入到一個有 zsh 的 tty ，我們可以直接在裡頭進行安裝工作。
如果沒有辦法進入，可能需要停用 secure boot。

以下安裝過程皆假設使用 UEFI

### 驗證開機模式

如果你已經啟用 UEFI 模式，Arch ISO 就會被經由 UEFI 啟動，在 UEFI 模式下，會存在目錄 /sys/firmware/efi/efivars ，我們如果想確保目前是以 UEFI 進入系統，便可以列出 efivars 目錄

```shell
ls /sys/firmware/efi/efivars
```

### 設定網路連線

如果能 ping 到 google.com 在絕大多數情況下就成功了

```shell
ping www.google.com
```

如果沒辦法 ping 到，先確保有啟動 dhcpcd

```shell
systemctl start dhcpcd.service
```

如果是有線連線，先確保該 interface 有 up

```shell
ifconfig <interface> up
```

如果要列出包含 down 的 interfaces

```shell
ifconfig -a
```

如果是無線，可以利用 netctl 的 wifi-menu

```shell
wifi-menu
```

然後不管是有線還是無線，確保有透過 DHCP 拿到 ip

```shell
ifconfig <interface>
```

如果沒有，手動設定 ip 和 routing table

```shell
ifconfig <interface> <ip>
route add <subnet> dev <interface>
route add default gw <gateway> dev <interface>
```

如果 DHCP 設定 DNS 太爛或是需要手動設定

```shell
vim /etc/resolv.conf
```

在頂部增加

```
nameserver <dns server>
```

這是暫時的，會被 dhcpcd 覆蓋，如果需要永久設定可以改成加入 ```/etc/resolv.conf.head ```

如果有程式沒有在查詢失敗時常是下一個 server ，加上 ```options rotate```可能會有幫助

### 分割硬碟分區

在開始分割硬碟區前先確認硬碟有正確讀到以及他的代號

```shell
lsblk -a
```

上面這個指令可以列出硬碟代號及大小，如果需要更詳細的資料

```shell
fdisk -l
```

在 linux 中 device nodes 位於 /dev 底下，其中 block devices (儲存裝置們)位於 /dev 或 /dev/block ，在 Arch 為前者，舉例來說透過運行 lsblk 後，得知固態硬碟名稱為 nvme0n1 ，他的 device node 位置便是 /dev/nvme0n1

其中常見 block devices 的命名規則如下

SATA 或 USB: ```sd<x><y>``` ，其中 x 為英文字母，表示第 x 顆硬碟， y 為數字，表示硬碟上的第 y 個分區

IDE 介面: ```hd<x><y>``` ，其中 x 為英文字母，表示第 x 顆硬碟， y 為數字，表示硬碟上的第 y 個分區
 
NVMe 介面: ```nvme<x>n<y>p<z>``` ，其中 x, y, z 為數字， ```<x>n<y>``` 表示硬碟， ```p<z>``` 表示分區
 
MMC: ```mmcblk<x>p<y>``` ，其中 x, y 為數字， x 表示碟， ```p<y>``` 表示分區

以最常見的第一顆 SATA 介面硬碟分區名稱 /dev/sda<y> 為例，並假設硬碟是空的，開始設定分區

```shell
cfdisk /dev/sda
```

* /dev/sda1: /boot
  **空間至少 512MB，類型為 EFI System** (若有其他系統的EFI分區可以直接沿用，且不須格式化)
  <br />

* /dev/sda2: Swap
  **自訂，作者使用 8G，類型為 Linux Swap**
  <br />

* /dev/sda3: /
  **自訂，作者使用全部剩餘空間，類型為 Linux filesystem**
  <br />

通常我們都會在系統上加上 swap 分區，用來儲存部份原本應該在 RAM 上的資訊。當然這個不是必須的，如果你覺得你的 RAM 大小足夠，可能不需要這個分區。順帶一提，當系統建立完成後想要新增 swap 分區，或是基於檔案的 swap 也都是可行的。

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

一般來說我們都是手動挑選各個需要的 package 來安裝，或者也可以選擇使用別人寫好的腳本，例如：https://github.com/danny8376/arch_install_script

以下為手動安裝

### 設定 pacman 的 mirrorlist

調整 pacman 啟用的鏡像站，可以提高下載安裝的速度。

```shell
vim /etc/pacman.d/mirrorlist
```

把非 Taiwan 的註解掉，把 Taiwan 的視所在位置啟用

### 安裝 base 和 base-devel group packages 

如果想要更小的系統你可能不需要```base-devel```

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

### 設定時區

```shell
ln -sf /usr/share/zoneinfo/Asia/Taipei /etc/localtime
```

### 設定語言環境

生成```zh_TW.UTF-8```語系

```shell
echo "zh_TW.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
```

設定預設為```zh_TW.UTF-8```

```shell
echo "LANG=zh_TW.UTF-8" > /etc/locale.conf
```

在tty底下無法直接顯示中文，使用```zh_TW.UTF-8```會出現一堆方塊，如果常直接在 tty 下做事可以用 ```export LC_ALL="C"```暫時修改，也可以只在 xinitrc 設定為```zh_TW.UTF-8```

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

在後面加入一般user之後可以透過```passwd -l root```防止使用root登入，但那會造成無法進入 emergency shell ，鎖定與否自行斟酌，這裡先修改密碼就好

```shell
passwd
```

### 安裝 grub 啟動載入程式

```shell
pacman -Sy grub os-prober efibootmgr
```

os-prober 可以用以偵測其他系統的存在，並在之後加入 grub 選單中，在 grub-mkconfig 內會自動執行，如果只有裝 Arch 就不用安裝

```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg
```

如果之後開機沒有載入grub而是載入了其他系統的bootloader，先檢查 /boot/EFI/Boot/Bootx64.efi 是否與 /boot/grub/grubx64.efi 相同，注意在FAT系列格式下大小寫不拘

### 更新 repo 資料和套件

就算是最新的 ISO 也有能資料不是最新的

```shell
pacman -Syu
```

### 安裝選用網路工具

```shell
pacman -S net-tools wireless_tools wpa_supplicant dialog
```

其中 wireless_tools wpa_supplicant dialog 只有要用 wifi 才需要， dialog 被 netctl 的 wifi-menu 功能需要

net-tools 提供了 ifconfig route 等指令，如果你會用 iproute2 的 ip 指令就不需要

如果連上網路後沒有得到 ip ，執行 ```systemctl enable dhcpcd.service``` 以及 ```systemctl start dhcpcd.service``` 確保 dhcpcd 有在運行

### 建立新使用者

安裝 sudo

```shell
pacman -S sudo
```

設定 sudo 群組

```shell
vi /etc/sudoers
```

找到該行(大約在第 82 行)，並刪除前方的 # 號

```
# %wheel ALL=(ALL) ALL
```

建立新使用者，並加入 sudo 群組

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

進入新系統後的網路設定請參考上方

## 初次進入系統

### 安裝 CPU 微代碼(Microcode)

詳細請參閱：[Microcode](<https://wiki.archlinux.org/index.php/Microcode_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)>)

#### AMD

對於 AMD 處理器，其 Microcode 更新以包在 linux-firmware 中，因此不需要額外動作

#### Intel

對於 Intel 處理器我們需要另外安裝套件，並且在 bootloader 啟用 Microcode 更新

```shell
sudo pacman -S intel-ucode
```

grub-mkconfig 時會自動處理載入 microcode 需要的參數，在安装完 intel-ucode 後，可以手動執行一次確保有被使用

```shell
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### 安裝顯示卡驅動

如果你有顯示卡的話，安裝針對顯示晶片的驅動，效能通常更好

#### NVIDIA

使用 NVIDIA 提供的 nvidia ，如果偏好開源可以跳過，原本預設會使用完全開源的 nouveau

```shell
sudo pacman -S nvidia
```

或者是 nvidia-lts

他們含有 kernel module，重新啟動才會載入

需要時可以透過其提供的 nvidia-settings 圖形界面程式來調整設定

#### AMD

因為筆者們目前沒有 AMD 的顯示卡因此請直接參參閱 https://wiki.archlinux.org/index.php/AMD_Catalyst_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)

### 安裝桌面環境

如果需要桌面環境，但不知道想用哪個可以試試 Gnome

#### Gnome

安裝 gnome 和 gnome-extra packages

```shell
sudo pacman -S gnome gnome-extra
```

使用 systemd 開機啟動 gdm (gnome 預設的 desktop manager )及 networkmanager ( gnome 使用的網路管理工具)

```shell
sudo systemctl enable NetworkManager
sudo systemctl enable gdm
```

### 安裝 aur helper

Arch 使用者軟體倉庫 ( AUR ) 是由社群推動的使用者軟體庫。它包含了 PKGBUILD 等打包時需要的資訊，可以用 makepkg 打包軟體包，並透過 pacman 安裝。透過 AUR 可以在社群間分享、組織新進軟體包，熱門的軟體包有機會被收錄進 community 軟體庫。

如果我們想要使用 aur 上的資源，我們需要確認有 [makepkg](https://wiki.archlinux.org/index.php/Makepkg) 及 git 指令。然後我們可以使用 aur helper 來幫我們打包 aur 上的內容。以下推薦 aurman

其他請參閱[arch aur](<https://wiki.archlinux.org/index.php/Arch_User_Repository_(%E6%AD%A3%E9%AB%94%E4%B8%AD%E6%96%87)>) 以及[aur helper](https://wiki.archlinux.org/index.php/AUR_helpers)頁面。

#### aurman

基於 python 的 aur helper

```
git clone https://aur.archlinux.org/aurman.git
cd aurman
makepkg -si
```

### 安裝中文輸入法 (fcitx)

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

## 安裝字型

```shell
sudo pacman -S noto-fonts noto-fonts-cjk ttf-roboto ttf-roboto-mono
```

noto-fonts 支援大多數 Unicode 的字元

noto-fonts-cjk 為 Google 提供的免費字型的中日韓子集(Chinese Japanese Korean)，建議至少也要安裝這個

ttf-robot 也是 Google 提供的很潮的字型，適合用來設計 UI

安裝過程圖形介面可能會好像當機、沒有反應，純屬正常現象，字型安裝完成就會恢復。

### NTFS 檔案系統讀寫支援

如果需要對 NTFS 有更好的支援，ntfs-3g 提供了以 FUSE 實做的驅動，以及對 NTFS 進行各種操作的指令

實際上如果只需要存取 NTFS 可以嘗試只用位於 linux kernel 內的驅動

參見[NTFS-3G](https://wiki.archlinux.org/index.php/NTFS-3G)
以及[Linux kernel source](https://github.com/torvalds/linux/blob/master/fs/ntfs/Kconfig)

```shell
sudo pacman -S ntfs-3g
```

### 桌面美化工程

接下來我們可以安裝 theme，主流的有：

##### Arc

```shell
sudo pacman -S arc-gtk-theme
```

##### Numix

```shell
sudo pacman -S numix-theme
```

#### 接下來是 icons

##### Arc-icon

```shell
sudo pacman -S arc-icon-theme 
```

##### Numix-icon

```shell
aurman -S numix-circle-icon-theme-git
```

##### Vivacious Colors icon

```shell
aurman -S vivacious-colors-icon-theme
```

其他的可以去 gnome-look 網站找來玩玩看：
https://www.gnome-look.org/browse/cat/135/

## Enjoy your new system

享受你的新系統，盡情客製化它吧！
