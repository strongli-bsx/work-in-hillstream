# petalinux

AMD 提供的 petalinux-v2022.2-10141622-installer.run 不是真实的 OS，是一个 linux 环境下的 sdk，用来生成真实的 OS。下面说一下 sdk 的安装使用（虚拟机环境）

1. 下载安装 ubuntu，我这里使用的是 ubuntu-20.04，一切按提示安装即可

   ```shell
   # ubuntu 安装后 backspace 不能使用怎么办？
   sudo vi /etc/vi/vimrc.tiny
   set nocompatiable
   set backspace=2
   
   # ubuntu 更换源
   sudo cp /etc/apt/source.list /etc/apt/source.list.backup
   sudo vi /etc/apt/source.list
   # 网上有源的资源，建议要更换，会影响 build
   
   # ubuntu 如何与外面的主机怎么互传文件？
   # 虚拟机设置管理中有个共享文件，打开即可
   vmware-hgfsclien
   # 如果没有反应
   sudo apt-get install open-vm-tools
   sudo vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other
   
   # 查看 ubuntu 版本号
   lsb_release -a
   ```

2. 将下载好的 petalinux-v2022.2-10141622-installer.run 放到共享目录下，注意 petalinux 版本要与 vivado 版本一致

   ```shell
   # 先安装支持包
   sudo apt-get install gawk
   sudo apt-get install net-tools 
   sudo apt-get install xterm
   sudo apt-get install autoconf 
   sudo apt-get install libtool
   sudo apt-get install texinfo 
   sudo apt-get install zlib1g-dev 
   sudo apt-get install gcc-multilib 
   sudo apt-get install build-essential 
   sudo apt-get install zlib1g
   sudo apt-get install libncurses5-dev
   sudo apt-get install zlib1g:i386
   
   sudo apt-get install iproute2 gawk python3 python build-essential gcc git make net-tools libncurses5-dev tftpd zlib1g-dev libssl-dev flex bison libselinux1 gnupg wget git-core diffstat chrpath socat xterm autoconf libtool tar unzip texinfo zlib1g-dev gcc-multilib automake zlib1g:i386 screen pax gzip cpio python3-pip python3-pexpect xz-utils debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev pylint3
   
   # 安装 tftp
   sudo apt-get install tftpd-hpa
   
   # 可以将 petalinux-xxx-installer.run 拷贝到相应目录下，然后执行
   petalinux-v2022.2-10141622-installer.run --dir petalinux
   
   # 等待安装完毕即可
   # 若 WARNING: This is not a supported OS，OS 版本不对，重新下载
   ```

3. petalinux 安装完毕后，此时用 bsp 先来检验下 petalinux 是否可用（先不要使用 xsa 文件生成 project，无法判断是 xsa 还是 peta 的原因）

   ```shell
   # 先将 ubuntu 的 dash 换为 bash，选择 否 即可
   sudo dpkg-reconfigure dash
   
   # 或者去 AMD 官网 petalinux 对应的版本下面 下载 对应的 .bsp 文件
   petalinux-create -t project -n test_bsp -s petalinux_gf.bsp
   
   # 若出现 ERROR: Failed to Kconfig project，则
   sudo apt-get install libtinfo5
   
   petalinux-build
   
   petalinux-config
   ```
   
4. 等待 build 后的结果

   - 可能会遇到 error，有可能是镜像网站不稳定导致 编译失败；我选择的是挂了梯子，也可以改用离线编译
   
   ```shell
   # 将 state-aarch64-2022，downloads-2022 下载下来通过 sftp 传到 ubuntu 上
   tar zxvf state-xxx downloads-xxx
   
   petalinux-config
   
   # 进入最后一行 yocto setting
   # Add pre-mirror url
   file:///home/xx/xlinx/downloads  # downloads的路径，前面加上 file://
   
   # Local sstate feeds settings
   /home/xx/xlinx/aarch64
   
   #不要退出yocoto settings界面
   #打开工程文件根目录/project-spec/meta-user/conf/petalinuxbsp.conf文件，在文件末尾添加
   DL_DIR="/home/xx/xlinx/downloads"
   #然后继续添加SSTATE_DIR=""，在双引号中间填上sstate文件夹的绝对路径，
   SSTATE_DIR="/home/xx/xlinx/aarch64"
   
   # 回到Yocoto Settings界面
   # 在Enable BB no network前面打上星号(按Y) ，然后保存并退出
   ```
   
   

---

使用：

1. 使用 bsp 来构建整个 project，其中 .bsp 是 gf 给的
2. xsa 选择自己 export 的
3. 更换 xsa 文件后，如果出错则 clean 一下，petalinux-build -x mrproper

---

Q: 如何创建裸机程序？

A: 直接使用 vitis，选择 standalone，fsbl 选择自动生成即可。最后将 boot.bin 拷贝到 sd 卡即可。

   注意：若需要打包其他文件，则右键点击 create image，可以将 R5 一块打包进 boot.bin，此时实现两个裸机同时启动。

Q: 如何同时启动 A53 和 R5？

A: 裸机的情况下，在 A53 裸机的 app 上右键 create image，其他选择默认，将 R5 生成的裸机 elf 加入，打包成 boot.bin，

​	 可以发现两个都启动了

---

# tftp

1. 打开 tftp.exe，找到 文件 所在文件夹
2. 在 petalinux 命令行中 输入   **tftp -gr 文件名 主机ip**

---

# boot

A53 运行 petalinux

1. 使用 petalinux-build -x mrproper clean 一下工程
2. 使用 petalinux-build，build 整个 project
3. 使用 petalinux-package --boot --fsbl --u-boot (--fpga user.bit --add r5.elf --cpu=r5-0) 打包生成 BOOT.bin，其中 bit，r5 elf，文件也可以在其他步骤生成
4. 将 BOOT.bin image.ub boot.scr 拷贝到 sd 卡，将 板卡设为 sd 卡启动即可



R5 运行 freertos

- 只需要在 package 加入 --add r5.elf --cpu=r5-0 即可一起生成 BOOT.bin

----

# vitis

如何使用 vitis 生成 BOOT.bin

- 将 petalinux 生成的 zynqmp_fsbl.elf，bl31.elf，pmufw.elf，system.dtb，u-boot.elf 文件 拷贝出来
- 右键 creat-image，将所有文件 按照 petalinux/image/linux/boogen.bif 的格式都添加进去即可 



vitis 开发 linux app

- A53 创建的步骤与 裸机一致，只不过将 OS 选择 linux 即可，此时在上面开发的 app.elf 通过 tftp 传入即可



如何不取出 sd 卡替换 BOOT.bin

- 将生成的 BOOT.bin 拷贝到 /mnt/boot/ 下面，reboot 即可



注意：

1. 若 R5 勾选了 自动生成 boot fsbl，则 petalinux-package 会失败，需要使用 petalinux 生的 fsbl 和 pmufw elf 文件，
2. freertos 中自动生成的文件是无法直接在 uart 输出的，打印方式 换成直接使用 printf 即可，具体原因为：____
3. petalinux 生成的 fsbl 只能用于 A53

---

# rootfs

- 将 sd 卡分为两个分区，自行百度
  - 将 boot.bin，image.ub 都复制到 boot 分区下

- 将 rootfs.tar.gz 解压到 sd 卡 rootfs 分区

  ```shell
  sudo tar xvf rootfs.tar.gz -C /media/xx/rootfs
  ```

- 更改权限，此时默认的权限是无法再 sd 卡读取启动的

  ```shell
  # 先将 sbin 的权限设为 777
  
  
  cd /media/xx/rootfs
  sudo chmod 755 usr
  sudo chmod 4755 usr/bin/sudo 
  ```

---

#  apps

- 创建用户app命令

  ```shell
  petalinux-create -t apps -n user-app --enable
  ```

- 删除代码

  ```shell
  #删除配置信息 有两处，如下
  a. user-project/project-spec/meta-user/conf/user-rootfsconfig
  b. user-project/project-spec/configs/rootfs_config
  #根据实际情况，没有添加–enable，则删除a中的user-app对应的配置信息即可；若是添加了–enable需要两处配置信息都要删除。
  

- 重新生成配置文件 & 重新编译

  ```shell
  petalinux-config -c rootfs
  petalinux-build
  
  ```

---

# devicetree

Q: 访问 IIC 地址会报错

A: kernel 中自带了很多驱动，比如 I2C，贸然访问 I2C 地址会报错；解决办法是在内核中将所有的 I2C 的选项都取消

---

## remoteproc

- 在 kernel 中打开 remoteproc 的选项，一般 petalinux 默认是打开的
- 在 rootfs 中增加 remoteproc 的支持
- 在 device-tree 增加 remoteproc 通信的 node
- 



```shell
echo app.elf > /sys/class/remoteproc/remoteproc0/firmware
echo start > /sys/class/remoteproc/remoteproc0/state
echo stop> /sys/class/remoteproc/remoteproc0/state
```

----
