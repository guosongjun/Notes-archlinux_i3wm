# Archlinux + i3wm 笔记

[TOC]

## 一、安装Arch

> 资料来源： 以官方Wiki的方式安装的Archlinux

### 01. 安装前检查

刻录好系统启动盘(rufus、Etcher)并进入BIOS选择以USB设备启动成功进入系统之后。

- 网络连接

  若ping没有网络时可能是没有启动网络服务：

  - 有线网络：使用ip a命令查看网卡端口，然后再用dhcpcd启动那个网口。（eg: dhcpcd enp0s3）
  - 无线网路：使用wifi-menu命令链接。

- 主板引导方式

  一般新的主板都是efi引导（进入BIOS会有efi字样），确认的方法不统一可以看看查看 /sys/firmware/efi/efivars是否有数据。或者使用fdisk查看是否有efi分区。

### 02. 系统磁盘

推荐使用fdisk（也可以使用cfdisk），也可以使用DiskGenius提前将分区分好（windows）

- 查看磁盘信息

  ````bash
  fdisk -l
  ````

- 选择需要分区的磁盘

   ````bash
   fdisk /dev/sda
   ````

- （如果是全新的硬盘需要新建分区表，如果是BIOS/MBR就输入o，如果是EFI/GPT就输入g）

- 输入n，进入创建分区：

   - Partition type（分区类型）—— 默认，primary（主要分区）

   - Partition number（分区编号）——默认

   - First sector（从哪里开始）——默认

   - Last sector（从哪里结束）——根据自己的需要可以定义空间大小（eg：+5G），默认就是全部空间

   > 题外话：建议创建一个挂载home的磁盘存储个人配置：以后可以多个系统共用一个home可以不用复制、重装系统也方便。

- 输入w，退出命令

   不小心建错了，可输入d 删除分区

- 格式化磁盘

   如果有efi引导磁盘，就是用fat32格式化它：

   ````bash
   mkfs.fat -F32 /dev/sda1
   ````

   安装磁盘，可以根据自己的需要格式为ext4（其实可以不用格式，目前arch默认格式为ext4）

   ````bash
   mkfs.ext4 /dev/sda2
   ````

- 挂载分区

   ```bash
   # 挂在临时安装节点
   mount /dev/sda2 /mnt
   # 有EFI就要挂载上
   mkdir /mnt/boot
   mount /dev/sda1 /mnt/boot
   ```

### 03. 选择镜像源

- 编辑mirrorlist文件

   ````bash
   vim /etc/pacman.d/mirrorlist
   ````

- 搜索163（网易）、cqu（重庆大学）、tsinghua（清华，记得改成https）、zju（浙江大学）

   实测网易最快，清华网速最慢。可以都复制到文件首（越前面优先级越强）

### 04. 安装系统基础包

> 2019-11-01 Arch的base包被精简，内核包linux可以任选依赖 —— 来源Archlinux贴吧大佬

```bash
pacstrap /mnt base base-devel linux dhcp
```

如果安装时出现：Errors occured, no packages were upgraded. ⇒ ERROR: Failed to install packages to new root.

> 百度之后各种方法都不能解决（全是各种贴吧大手子的回复，今天突然注意到是有人回答正确方法的，pacman里面有解决办法，没有看到。。），心灰意冷还是上了Google，第二页找到解决办法，当然此方法可能并不通用，也可贴吧回复的方法只是针对我没用
> 解决办法如下，执行命令
>
> ````bash
> pacman-key --refresh-keys
> ````
> ————————————————
> 版权声明：本文为CSDN博主「不应有的淡定」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
> 原文链接：https://blog.csdn.net/shouldnotappearcalm/article/details/78823682

### 05. 配置Fstab（自动挂载）

生成自动挂载分区的fstab文件，命令如下：

````bash
genfstab -L /mnt >> /mnt/etc/fstab
````

这个挺重要的，可以使用cat查看一下分区是否都正确挂载了。

```bash
cat /mnt/etc/fstab
```

### 06. 切换系统执行位置

注意chroot是创建一个虚拟环境，以为我们在这个环境的操作只作用于我们执行那个系统根目录，不会影响到arch CD。这个特性十分方便我们系统挂掉之后的抢救工作以及备份。

```bash
arch-chroot /mnt
```

### 07. 设置时区

设置为上海时区并生成相关文件：

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
```

### 08. 需要安装的软件

```bash
pacman -S vim dialog wpa_supplicant ntfs-3g networkmanager
```

dialog、wpa_supplicant(与无线网络相关)

ntfs-3g （文件系统）

> 要支持制作fat文件系统，安装dosfstools，默认内核只能读取ntfs，要支持ntfs读写，安装ntfs-3g。
>
> Form ：  https://www.cnblogs.com/bluestorm/p/5929172.html

networkmanager （网络管理）

### 09. 设置Locale

编辑/etc/locale.gen文件，并取消掉`zh_CH.UTF-8 UTF-8`、`zh_HK.UTF-8 UTF-8`、`zh_TW.UTF-8 UTF-8`、`en_US.UTF-8 UTF-8`前面的#号（当然也可以只根据你的需要来）

```bash
vim /etc/locale.gen
```

然后更新一下你选的语言包：

```bash
locale-gen
```

然后创建/etc/locale.conf文件，在第一行输入：

```bash
LANG=zh_CH.UTF-8
```

### 10. 设置主机名（略）

### 11. 更改root密码

````bash
passwd root
````

arch中命令提示符一般#代表root用户，$代表普通用户。

### 12. 安装Intel-ucode（非Intel CPU跳过）

```bash
pacman -S intel-ucode
```

### 13. 安装引导

有比较多的方案：Grub2、rEFInd、甚至直接用主板efi启动。这里使用成熟的Grub2。

建议安装os-prober ntfs-3g这个两个包，方便Grub检测存在的系统（这里点名windows）

```bash
pacman -S os-prober ntfs-3g
```

#### （一）、BIOS/MBR方式引导

- 安装grub包：

   ```bash
   pacman -S grub
   ```

- 部署grub：

   ````bash
   grub-install --target=i386-pc /dev/sda #注意是整个盘符
   ````

   > `--target=i386-pc`指示`grub-install`是为使用BIOS的系统安装. 推荐一直标明这点以防混淆.
   >
   > From :  https://www.cnblogs.com/studyone/p/5500679.html

- 生成配置文件：

   ```bash
   grub-mkconfig -o /boot/grub/grub.cfg
   ```

#### （二）、EFI/GPT方式引导

- 安装grub、efibootmgr包：

   ```bash
   pacman -S grub efibootmgr
   ```

- 部署grub：

   ````bash
   grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
   ````

- 生成配置文件：

   ```bash
   grub-mkconfig -o /boot/grub/grub.cfg
   ```



多系统话最好记得检查一下grub.cfg是否都包含了。

### 14. 再次开机无网络

很简单没有启动网络服务，幸好我们前面是下载一个NetworkManager。可以把他加入开机启动项中：

```bash
systemctl start NetworkManager   # 启动服务(注意大小写)
systemctl enable NetworkManager  # 加入开机启动
#systemctl disable NetworkManager # 取消开机启动
#systemctl stop NetworkManager    # 停止服务
```



## 二、安装i3wm

> 资料来源： ArchLinux安装（三）-i3wm安装及配置

### 01. 新建用户（可选）

- 还是要有个健康的使用习惯：

  ```bash
  useradd -m -G wheel 用户名
  ```

- 记得加个密码：

  ```bash
  passwd 用户名
  ```

- 配置sudo

  sudo本身也是一个软件包，需要修改让他支持wheel组的用户可以提升权限：

  ```bash
  vi /etc/sudoers #或者 visudo
  ```

  要么找到`# %wheel ALL=(ALL)ALL`取消前面的#注释，要么就在`root ALL=(ALL)ALL`的下一行加入自己用户的语句。

### 02. 显卡驱动

- 查看显卡类型：

  ```bash
  lspci | grep -e VGA -e 3D
  ```

- 然后搜索对应的显卡驱动：

  ```bash
  pacman -Ss xf86-video
  ```
  
- 常见的开源显卡驱动
  
  | 类型                 | 软件包名           |
  | -------------------- | ------------------ |
  | 通用(不支持硬件加速) | xf86-video-vesa    |
  | Intel                | xf86-video-intel   |
  | AMD                  | xf86-video-ati     |
  | AMD                  | xf86-video-amdgpu  |
  | Nvidia               | xf86-video-nouveau |


### 03. 安装Xorg框架

- 下载安装依赖包：

  ```bash
  sudo pacman -S xorg-server xorg-xrandr xorg-xrdb xorg-xinput xf86-input-mouse xf86-input-keyboard
  ```

- 生成Xorg配置：

  ```bash
  X -configure #注意大写X
  ```

- 使用配置：

  ```bash
  cd # 回到用户目录
  mv xorg.conf.new /etc/X11/xorg.conf
  ```

### 04. 安装i3组件

- 随便再安装个基础终端和中文字体：

  ```bash
  sudo pacman -S i3 xterm adobe-source-han-sans-cn-fonts
  ```

- 使用LightDM启动i3

  ```bash
  sudo pacman -S lightdm lightdm-settings lightdm-gtk-greeter
  sudo systemctl enable lightdm
  sudo systemctl start lightdm
  ```

### 05. i3快捷键

> 链接 [i3wm快捷键]( https://blog.csdn.net/weixin_43833642/article/details/103032095 ) ，建议读一读[i3wm.org]( https://i3wm.org/docs/userguide.html#_opening_terminals_and_moving_around )

| 快捷键                  | 功能                                              |
| ----------------------- | ------------------------------------------------- |
| $mod + Enter            | 启动虚拟终端                                      |
| $mod + A                | 焦点转义到父窗口上                                |
| $mod + S                | 堆叠布局                                          |
| $mod + W                | 标签布局                                          |
| $mod + E                | 默认布局                                          |
| $mod + SpaceBar         | 焦点在平铺式／浮动式转换                          |
| $mod + D                | 启动 dmenu                                        |
| $mod + H                | 水平分割窗口                                      |
| $mod + V                | 垂直分割窗口                                      |
| $mod + J                | 焦点往左窗口移                                    |
| $mod + K                | 焦点往下窗口移                                    |
| $mod + L                | 焦点往上窗口移                                    |
| $mod + ;                | 焦点往右窗口移                                    |
| $mod + Shift + Q        | 杀死当前窗口的进程                                |
| $mod + Shift + E        | 退出 i3                                           |
| $mod + Shift + C        | 当场重新加载 i3config, 无需重启                   |
| $mod + Shift + R        | 重启 i3 （还重新加载了 i3config, 又没有退出过程） |
| $mod + Shift + J        | 窗口左移                                          |
| $mod + Shift + K        | 窗口下移                                          |
| $mod + Shift + L        | 窗口上移                                          |
| $mod + Shift + :        | 窗口右移                                          |
| $mod + Shift + SpaceBar | 窗口在平铺式／浮动式转换                          |

### 06. 常用pcaman指令：

```bash
sudo pacman -S 软件名　# 安装
sudo pacman -R 软件名　# 删除单个软件包，保留其全部已经安装的依赖关系
sudo pacman -Rs 软件名 # 除指定软件包，及其所有没有被其他已安装软件包使用的依赖关系
sudo pacman -Ss 软件名  # 查找软件
sudo pacman -Sc # 清空并且下载新数据
sudo pacman -Syu　# 升级所有软件包
sudo pacman -Qs # 搜索已安装的包
```



## 三、常用配置

### 01. 添加archlinuxcn的软件源

> 链接:  [arch/manjaro - 添加archlinuxcn的软件源]( http://www.360doc.com/content/19/1014/22/11770334_866813856.shtml )

- 在`/etc/pacman.conf`尾新添加：

  ```config
  [archlinuxcn]
  SigLevel = Optional TrustAll
  Server = http://mirrors.ustc.edu.cn/archlinuxcn/$arch
  ```

- 再更新一下yaourt，同时安装cnkey：

  ```bash
  sudo pacman -Syu yaourt
  sudo pacman -Syy
  sudo pacman -S archlinuxcn-keyring
  ```

-  设置aur源(可选) ：

  ```bash
  sudo vim /etc/yaourtrc
  AURURL="https://aur.tuna.tsinghua.edu.cn"
  ```

- 也可以想官方一样同时添加多个源，防止下载过慢

  ```bash
  sudo pacman -S archlinuxcn-mirrorlist-git
  ```

- 修改配置

  ```bash
  sudo vim /etc/pacman.conf
  [archlinuxcn]
  Include = /etc/pacman.d/archlinuxcn-mirrorlist
  ```

- mirrorlist配置

  建议访问[Pacman中国镜像列表生成器]( https://www.archlinux.org/mirrorlist/?country=CN )以获得最新的mirrorlist

  ```bash
  sudo vim /etc/pacman.d/archlinux-mirrorlist
  ##
  ## Arch Linux repository mirrorlist
  ## Generated on 2019-11-20
  ##
  
  ## China
  Server = http://mirrors.163.com/archlinux/$repo/os/$arch
  Server = http://mirrors.cqu.edu.cn/archlinux/$repo/os/$arch
  Server = https://mirrors.cqu.edu.cn/archlinux/$repo/os/$arch
  Server = http://mirror.lzu.edu.cn/archlinux/$repo/os/$arch
  Server = http://mirrors.neusoft.edu.cn/archlinux/$repo/os/$arch
  Server = https://mirrors.neusoft.edu.cn/archlinux/$repo/os/$arch
  Server = https://mirrors.sjtug.sjtu.edu.cn/archlinux/$repo/os/$arch
  Server = http://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
  Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
  Server = http://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
  Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
  Server = https://mirrors.xjtu.edu.cn/archlinux/$repo/os/$arch
  Server = http://mirrors.zju.edu.cn/archlinux/$repo/os/$arch
  ```

  或者直接下网页，手动将#注释去掉：

  ```bash
  wget -O /etc/pacman.d/mirrorlist https://www.archlinux.org/mirrorlist/?country=CN
  ```

### 02. 安装中文输入法

- 安装fcitx管理器和搜狗：

  ```bash
  sudo pacman -S fcitx
  sudo pacman -S fcitx-configtool
  sudo pacman -S fcitx-gtk2 fcitx-gtk3 fcitx-qt4 fcitx-qt5
  sudo pacman -S fcitx-sogoupinyin
  ```

- .xprofile配置：

  ```config
  vim ~/.xprofile
  export GTK_IM_MODULE=fcitx
  export QT_IM_MODULE=fcitx
  export XMODIFIERS="@im=fcitx"
  fcitx & # 开机启动
  ```

- 加入i3自启动项（可选）

  ```bash
  vim ~/.config/i3/config
  exec_always --no-startup-id fcitx
  ```

### 03. 安装字体

- 安装字体包：

  ```bash
  sudo pacman -S ttf-roboto noto-fonts ttf-dejavu ttf-font-awesome nerd-fonts-complete
  sudo pacman -S wqy-bitmapfont wqy-microhei wqy-microhei-lite wqy-zenhei
  sudo pacman -S noto-fonts-cjk adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts
  ```

- 配置字体（可选）：

  将[fonts.conf](.config/fontconfig)复制到.config/fontconfig中去。

  ```bash
  git clone https://github.com/luanshizhimei/Note-Archlinux-i3WM.git
  cp -r Note-Archlinux-i3WM/.config/fontconfig/fonts.conf ~/.config/fontconfig/fonts.conf
  ```

- 查找更多中文字体：

  > 链接 : [arhlinux wiki Fonts]( [https://wiki.archlinux.org/index.php/Fonts_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E4%B8%AD%E6%96%87%E5%AD%97](https://wiki.archlinux.org/index.php/Fonts_(简体中文)#中文字) 

### 04. 终端配置

这里是采用的alacritty + oh my zsh + agnoster的方案

- 安装alacritty

  ```bash
  sudo pacman -S alacritty
  ```

- 修改alacritty配置，修改透明度：

  ```bash
  vim ~/.config/alacritty/alacritty.yml
  background_opacity: 0.7
  ```

- 安装compton

  ```bash
  sudo pacman -S compton
  ```

  加入i3自启动项中：

  ```bash
  vim ~/.config/i3/config
  exec_always --no-startup-id compton
  ```

- 安装zsh

  ```bash
  sudo pacman -S zsh
  ```

- 将zsh改为默认shall

  ```bash
  which zsh # 获得安装目录
  chsh -S /usr/bin/zsh
  ```

- 安装oh my zsh

  ```bash
  sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
  ```

  建议根据官网说明来：

  > 链接 :  [官方安装](https://github.com/ohmyzsh/ohmyzsh#basic-installation )

- 安装Powerline Fonts 字体：

  > 链接：[简单的manjaro安装powerline及vim zsh配置]( https://blog.csdn.net/z924139546/article/details/79815788 )

### 07. 常用软件

| 软件名             | 包名                          |
| ------------------ | ----------------------------- |
| 火狐浏览器         | friefox                       |
| 谷歌浏览器（开源） | chromium                      |
| 自动更换壁纸       | feh + variety                 |
| Vs code            | code / visual-studio-code-bin |
| 装逼利器           | neofetch                      |
| wps                | wps-office                    |
| tim                | deepin-wine-tim               |
| 截屏工具           | shutter                     |
| 资源管理器         | ranger                        |
| 网易云音乐         | netease-cloud-music           |
| 程序启动器         | rofi / dmenu                  |
| 终端               | alacritty                     |
| 渲染器             | compton                       |
| 主题设置管理器     | lxappearance                  |
| 进程查看器         | htop                          |
| 视频播放器         | vlc                           |
| MarkDown编辑软件   | typora                        |
| 修图               | gimp                          |
| 思维导图           | xmind                         |
| steam              | steam                         |
| FPS游戏            | sauerbraten                   |
| 我的世界           | minecraft                     |
| 模仿马里奥卡丁车   | supertuxkart                  |
| 酸酸               | shadowsocks-qt5               |
| Proxychains 代理   | proxychains-ng                |

更多软件推荐：

> 链接 : [Manjaro 常用软件安装]( https://blog.csdn.net/weixin_43968923/article/details/86662256 )