# 安装ArchLinux

- [安装ArchLinux](#安装archlinux)
  - [安装步骤](#安装步骤)
  - [意外处理](#意外处理)
    - [因为强拔u盘时导致超级块损坏时的修复方法](#因为强拔u盘时导致超级块损坏时的修复方法)
    - [Linux访问windows磁盘出现&quot;Error mounting /dev/sda2 at/media&quot;](#linux访问windows磁盘出现quoterror-mounting-devsda2-atmediaquot)

***

## 安装步骤

1. 利用 **lsblk** 或 **sudo fdisk -l** 命令查看u盘所在位置，其会挂在在sdb下
2. 格式化u盘

   ```sh
   mkfs.ext3 /dev/sdb
   ```

   注意：

    - /dve/sdb是u盘
    - 格式化可能很慢，不要拔出u盘，否则会造成超级快损坏，修复方式见[意外处理](#因为强拔u盘时导致超级块损坏时的修复方法)

3. 将archlinux镜像刻录进u盘

   ```sh
    dd if=/home/wfb/Downloads/archlinux-2019.11.01-x86_64.iso of=/dev/sdb bs=8M
   ```

   注意：

   - if后跟ios镜像的路径，of后跟u盘路径。bs表示缓冲区大小
   - dd命令可能会长时间阻塞，请耐心等待。强拔u盘会造成超级块损坏，修复方式见[意外处理](#因为强拔u盘时导致超级块损坏时的修复方法)

4. 进入BIOS设置u盘启动，进入安装界面，选择install
5. 验证启动模式：
   输入 **ls /sys/firmware/efi/efivars** 命令，若可以列出该目录，则为UEFI模式；否则为BIOS模式
6. 连接到因特网 （因为archlinux不能离线安装，所以安装前必须先联网）：
   可以通过 **ping www.baidu.com** 查看是否能够连接网络，可以按 **Ctrl + D** 结束 **ping** 命令
      - 有线网络利用 **dhcpcd** 命令获取ip地址
      - 无线网络利用 **wifi-menu** 命令在字符形式的图形化界面下完成配置
7. 更新系统时间（该命令执行成功时不会有任何输出）：

   ```sh
   timedatectl set-ntp true
   ```

8. 分区与格式化：
   1. 利用 **fdisk -l** 查看目前的分区情况
   2. 输入 **cfdisk /dev/nvme0n1** 进行分区调整的字符图形化界面：
   3. 删除不需要的分区
   4. 新建一个512M的分区，设置**type**为**efi system**
   5. 新建一个8G的分区，设置**type**为**swap**
   6. 其他空闲分区设置**type**为**linux filesystem**
   7. 选择**write**，输入**yes**，进行写入
   8. 选择**quit**，退出cfdisk系统
   9. 进行分区的格式化：
      1. 利用 **mkfs.vfat _/dev/nvme0n1p1_** 命令格式化**type**为**efi system**的分区，注意**斜体请改为该分区的路径**
      2. 利用 **mkswap _/dev/nvme0n1p2_** 命令格式化**type**为**swap**的分区，注意**斜体请改为该分区的路径**
      3. 利用 **swapon _/dev/nvme0n1p2_** 命令开启swap，注意**斜体为该分区的路径**
      4. 利用 **mkswap _/dev/nvme0n1p3_** 命令格式化**type**为**linux filesystem**的分区，注意**斜体请改为该分区的路径**
9. 挂载格式化的分区：
   1. 利用 **mount _/dev/nvme0n1p3_ /mnt** 命令将**linux filesystem**挂载在 **/mnt**
   2. 利用 **mkdir /mnt/boot** 命令在**mnt**下创建**boot**目录
   3. 利用 **mount _/dev/nvme0n1p1_ /mnt/boot** 命令将**efi system**挂载在 **/mnt/boot**
10. 将pacman的镜像列表修改为优先走清华源：
    1. 输入 **vim /etc/pacman.d/mirrorlist** 进入vim界面
    2. 输入 **/tuna** 和 **回车** 定位到清华映像源的配置行
    3. 依次输入 **dd gg p**将该行剪切，到第一行，然后黏贴
    4. 输入 **:wq** ，按 **回车** 保存并退出vim
11. 往 **/mnt**中装入**base包组**、**linux包**、**linux-fireware包**

    ```sh
    pacstrap /mnt base linux linux-fireware
    ```

    下载过程中可能会出现error提示，该报错是因为无法从清华源下载该组件，会自动用下一个配置的镜像去下载该组件
12. 生成fstab文件，然后检查该文件是否正确：

    ```sh
    genfstab -U /mnt >> /mnt/etc/fstab
    ```

13. 进入到新安装的系统：

    ```sh
    arch-chroot /mnt
    ```

14. 设置时区，之后运行**hwclock**命令生成**/etc/adjtime**文件：

    ```sh
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
    hwclock --systohc
    ```

15. 安装**vim**：

    ```sh
    pacman -Syy #强制刷新pacman的软件包数据库，相当于apt update
    pacman -S vim #安装vim
    ```

16. 本地化：

    ```sh
    vim /etc/locale.gen
    #将如下三行的注释打开：
    #en_US.UTF-8 UTF-8
    #zh_CN.UTF-8 UTF-8
    #zh_TW.UTF-8 UTF-8
    #保存并退出vim界面
    locale-gen #生成locale讯息
    vim locale.conf #创建locale.conf文件
    <<'COMMENT'
        输入如下一行
        LANG=en_US.UTF-8
        保存并退出vim界面
    COMMENT
    ```

17. 配置网络：
    1. 创建**hostname**文件：

        ```sh
        vim /etc/hostname
        #输入wfbpc
        #保存并退出vim界面
        ```

    2. 添加对应的信息到**hosts**:

        ```sh
        vim /etc/hosts
        <<'COMMENT'
            添加以下三行信息：
            #127.0.0.1  localhost
            #::1        localhost
            #127.0.1.1  wfbpc.localdomain   wfbpc
        COMMENT
        #保存并退出vim界面
        ```

18. 执行 **passwd** ，为root用户创建密码
19. 安装引导程序：

    ```sh
    #pacman -S os-prober ntfs-3g
    #这两个包可以配置grub检测已经存在的系统，并自动设置启动选项。如果只装archlinux这一个系统，可以跳过该步骤
    pacman -S grub #安装grub包
    pacman -S efibootmgr #安装efiboot管理器
    grub -install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub #将grub主目录设为/boot/grub/
    grub -mkconfig -o /boot/grub/grub.cfg
    #使用 grub-mkconfig 工具来生成 /boot/grub/grub.cfg
    ```

20. 创建一个新用户用来登录：
    1. 执行 **useradd -m -G wheel _wfb_** 创建wheel组的wfb用户
    2. 执行 **passwd _wfb_** 为该用户设置密码
    3. 执行 **pacman -S sudo** 安装sudo包，以便普通用户提权
    4. 执行 **visudo** 进入sudo的配置文件，找到 **%wheel ALL=(ALL)** ，取消改行的注释
21. 安装图形化界面：

    ```sh
    pacman -S xorg plasma-meta kde-applications-meta
    #安装xorg、plasma-meta(plasma桌面系统)和kde-applications-meta(常用软件)包，所有的提示都走默认，即直接敲入回车
    systemctl disable nettools(netstat命令在这里面呦) #禁用nettools，如果报未启动或找到的错误，请忽视
    systemctl enable NetworkManager #开启NetworkManager
    systemctl enable sddm #开启sddm
    ```

22. 解决n卡驱动问题：

    ```sh
    pacman -S bumblebee mesa nvidia xf86-video-intel
    gpasswd -a wfb bumblebee #将wfb换成新建的用户名
    systemctl enable bumblebee.service #启动bumblebee
    ```

23. 手动卸载被挂载的分区,退出chroot，然后重启，关机时拔掉u盘:

    ```sh
    umount -R /mnt #卸载分区
    exit #退出chroot
    reboot #重启
    ```

24. 添加清华archlinuxcn源：

    ```sh
    vim /etc/pacman.conf
    #在最后添加如下两行
    #[archlinuxcn]
    #Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
    pacman -S archlinuxcn-keyring
    sudo pacman -Syy #刷新数据库
    ```

25. 安装中文字体：

    ```sh
    pacman -Ss adobe han #查看adobe的开源汉字
    pacman -S community/adobe-source-han-sans-cn-fonts
    pacman -S community/adobe-source-han-sans-tw-fonts
    pacman -S community/adobe-source-han-serif-cn-fonts
    pacman -S community/adobe-source-han-serif-tw-fonts
    ```

26. 安装中文输入法（搜狗拼音）：

    ```sh
    sudo pacman -S fcitx
    sudo pacman -S fcitx-configtool
    sudo pacman -S fcitx-gtk2 fcitx-gtk3 fcitx-qt4 fcitx-qt5
    sudo pacman -S fcitx-sogoupinyin
    sudo pacman -S fcitx-configtool #安装配置工具
    vim ~/.xprofile #创建.xprofile文件
    <<'COMMENT'
        添加如下三行:
        export GTK_IM_MODULE=fcitx
        export QT_IM_MODULE=fcitx
        export XMODIFIERS="@im=fcitx"
    COMMENT
    reboot #重启电脑，使输入法生效
    ```

## 意外处理

### 因为强拔u盘时导致超级块损坏时的修复方法

1. 利用dd命令用全0覆盖u盘代码如下:

    ```sh
    dd if=/dev/zero of=/dev/sdb bs=8M
    ```

1. 利用 **mkfs.ext3 /dev/sdb** 等命令格式化u盘为相应的格式

### Linux访问windows磁盘出现"Error mounting /dev/sda2 at/media"

利用 **sudo pacman -S ntfs-3g** 安装**ntfs-3g**
