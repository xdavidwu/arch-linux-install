# Moooore freedom

這篇來介紹如何達到更大程度的自由

Arch Linux 對開源是沒有到很要求的， repo 裡面直接摻雜了閉源軟體，例如最明顯的 nvidia-utils 裡面就充滿了來自 NVIDIA 的 binary blobs ， 
linux-firmware 裡也很多閉源的 firmware ，官方 repo 內還有像 flash player plugin, discord 等明顯閉源的套件

對於這些套件， Arch Linux 並沒有做例如丟進 non-free repo 的隔離措施，而是直接放進一般的 repo ，把 license 欄設為看不出到底開不開源的 custom

如果想要達到完全開源，通常會裝 Parabola GNU/Linux-libre 這個衍生發行版，但大多數人總是還是因為例如某些 device 只有非完全開源的驅動，
只能裝 Arch Linux ，例如目前好像還沒有不用 firmware 的 802.11ac 網卡

## absolutely-proprietary

如果想要看自己的 Arch Linux 有多少不自由的軟體，可以用 AUR 上的 absolutely-proprietary ，他會把軟體清單跟 Parabola 提供的 blacklist 比對，並列出來，清單內含有不自由的原因，和他自由的替代方案

自由的程度有分類，例如 nonfree, semifree, uses-nonfree 等

## Parabola [libre]

Parabola 的 [libre] 套件庫包含自由化 (例如移除不自由的部份) 的套件，裡面都是完全開源且 license 自由的，雖然我們可能無法直接使用 Parabola ，
但可以從他的 [libre] 來裝一些自由的替代方案，因為 Parabola 是基於 Arch Linux 的，甚至他們的很多套件庫直接是刪除 blacklist 後的 
Arch Linux 相對套件庫，把 [libre] 的東西拿來用不怎麼會出問題

在 pacman.conf 加入 [libre]

```
[libre]
Server = https://mirror.fsf.org/parabola/$repo/os/$arch
```

這裡是使用 FSF 的 mirror ，我的環境用起來是比 Parabola 官方的快

然後小心的 pacman -Syu 一下，不要更新到任何東西，先看看他會動到什麼

例如 linux-libre replaces linux, linux-firmware-libre replaces linux-firmware ，但我因為網卡需要閉源 firmware 才能動，所以我把他們都加進 pacman.conf 的 IgnorePkg

pacman-mirrorlist 在 [libre] 內是 Parabola 的 mirrorlist, 記得 IgnorePkg 掉

確認自己都處理好要留的東西再更新

小心不要裝到 your-freedom 套件，那是刪除 blacklist 套件用的(透過 conflicts )

absolutely-proprietary 不會判斷套件的來源，所以就算你裝了 [libre] 裡面同名的自由版，還是會算成不自由
