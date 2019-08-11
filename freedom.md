# Moooore freedom (draft)

這篇來介紹如何達到更大程度的自由

Arch Linux 對開源是沒有到很要求的， repo 裡面直接摻雜了閉源軟體，例如最明顯的 nvidia-utils 裡面就充滿了來自 NVIDIA 的 binary blobs ， 
linux-firmware 裡也很多閉源的 firmware ，官方 repo 內還有像 flash player plugin, discord 等明顯閉源的套件

對於這些套件， Arch Linux 並沒有做例如丟進 non-free repo 的隔離措施，而是直接放進一般的 repo ，把 license 欄設為看不出到底開不開源的 custom

如果想要達到完全開源，通常會裝 Parabola GNU/Linux-libre 這個衍生發行版，但大多數人總是還是因為例如某些 device 只有非完全開源的驅動，
只能裝 Arch Linux ，例如目前好像還沒有不用 firmware 的 802.11ac 網卡
