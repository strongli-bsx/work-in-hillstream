# petalinux2022

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
   sudo apt-get install gawk  -y
   sudo apt-get install net-tools  -y
   sudo apt-get install xterm  -y
   sudo apt-get install autoconf -y
   sudo apt-get install libtool  -y
   sudo apt-get install texinfo  -y
   sudo apt-get install zlib1g-dev  -y
   sudo apt-get install gcc-multilib  -y
   sudo apt-get install build-essential  -y
   sudo apt-get install zlib1g  -y
   sudo apt-get install libncurses5-dev  -y
   sudo apt-get install zlib1g:i386  -y
   
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
   sudo apt-get install libtinfo5 -y
   sudo apt install libncurses* -y
   
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

Q: 如何添加自启动脚本

A: 在 etc/profile.d/ 文件下面添加脚本，**系统启动的时候会从 该文件夹下面依次 调用**

---

# tftp

1. 打开 tftp.exe，找到 文件 所在文件夹
2. 在 petalinux 命令行中 输入   **tftp -gr 文件名 主机ip**

---

# boot

A53 运行 petalinux

1. 使用 petalinux-build -x mrproper clean 一下工程
2. 使用 petalinux-build，build 整个 project
3. 使用 petalinux-package --boot --fsbl --u-boot --pmufw --force(--fpga user.bit --add r5.elf --cpu=r5-0) 打包生成 BOOT.bin，其中 bit，r5 elf，文件也可以在其他步骤生成
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

# remoteproc

- 在 kernel 中打开 remoteproc 的选项，一般 petalinux 默认是打开的

- 在 rootfs 中增加 remoteproc 的支持 

- 在 device-tree 增加 remoteproc 通信的 node

  ```user.dtsi
  /include/ "system-conf.dtsi"
  / {
  	reserved-memory {
  		#address-cells = <2>;
  		#size-cells = <2>;
  		ranges;
  		rpu0vdev0vring0: rpu0vdev0vring0@3ed40000 {
  			no-map;
  			reg = <0x0 0x3ed40000 0x0 0x4000>;
  		};
  		rpu0vdev0vring1: rpu0vdev0vring1@3ed44000 {
  			no-map;
  			reg = <0x0 0x3ed44000 0x0 0x4000>;
  		};
  		rpu0vdev0buffer: rpu0vdev0buffer@3ed48000 {
  			no-map;
  			reg = <0x0 0x3ed48000 0x0 0x100000>;
  		};
  		rproc_0_reserved: rproc@3ed00000 {
  			no-map;
  			reg = <0x0 0x3ed00000 0x0 0x40000>;
  		};
  	};
  
  
  	tcm_0a@ffe00000 {
  		no-map;
  		reg = <0x0 0xffe00000 0x0 0x10000>;
  		phandle = <0x40>;
  		status = "okay";
  		compatible = "mmio-sram";
  		power-domain = <&zynqmp_firmware 15>;
  	};
  
  	tcm_0b@ffe20000 {
  		no-map;
  		reg = <0x0 0xffe20000 0x0 0x10000>;
  		phandle = <0x41>;
  		status = "okay";
  		compatible = "mmio-sram";
  		power-domain = <&zynqmp_firmware 16>;
  	};
  
  
  	rf5ss@ff9a0000 {
  		compatible = "xlnx,zynqmp-r5-remoteproc";
  		xlnx,cluster-mode = <1>;
  		ranges;
  		reg = <0x0 0xFF9A0000 0x0 0x10000>;
  		#address-cells = <0x2>;
  		#size-cells = <0x2>;
  
  		r5f_0 {
  			compatible = "xilinx,r5f";
  			#address-cells = <2>;
  			#size-cells = <2>;
  			ranges;
  			sram = <0x40 0x41>;
  			memory-region = <&rproc_0_reserved>, <&rpu0vdev0buffer>, <&rpu0vdev0vring0>, <&rpu0vdev0vring1>;
  			power-domain = <&zynqmp_firmware 7>;
  			mboxes = <&ipi_mailbox_rpu0 0>, <&ipi_mailbox_rpu0 1>;
  			mbox-names = "tx", "rx";
  		};
  	};
  
  
  	zynqmp_ipi1 {
  		compatible = "xlnx,zynqmp-ipi-mailbox";
  		interrupt-parent = <&gic>;
  		interrupts = <0 29 4>;
  		xlnx,ipi-id = <7>;
  		#address-cells = <1>;
  		#size-cells = <1>;
  		ranges;
  
  		/* APU<->RPU0 IPI mailbox controller */
  		ipi_mailbox_rpu0: mailbox@ff990600 {
  			reg = <0xff990600 0x20>,
  			      <0xff990620 0x20>,
  			      <0xff9900c0 0x20>,
  			      <0xff9900e0 0x20>;
  			reg-names = "local_request_region",
  				    "local_response_region",
  				    "remote_request_region",
  				    "remote_response_region";
  			#mbox-cells = <1>;
  			xlnx,ipi-id = <1>;
  		};
  	};
  };
  
  
  ```



```shell
echo app.elf > /sys/class/remoteproc/remoteproc0/firmware
echo start > /sys/class/remoteproc/remoteproc0/state
echo stop > /sys/class/remoteproc/remoteproc0/state
```

----

# mem

D: 使用 devmem 只能单字节访问，无法访问连续多个字节地址

R: devmem 不支持

S: 单独实现 dev/mem 的驱动

---

# i2c

D: 在 R5 上面实现 i2c 驱动，直接访问 i2c 地址，linux 挂死

R: petalinux 上默认安装了 i2c 的驱动，在 user 模式下，这个地址是不能访问的；但是我们 debug 模式使用的是 root，所以这个地址是强行访问，所以系统会 挂掉

S: 将 petalinux-config -c kernel 中所有关于 i2c 的项目**全部**取消即可，**包括子菜单**

---

# usb

根据 xlinx 官网给的参考例程，可以实现 串口的 功能 以及 usb MTD 功能，书签里有保存

---

# petalinux2019

1. 需要使用 ubuntu 18.04.2，严格按照 手册上面的，ubunt 20.04 不行

2. devmem 无法使用，petalinux2022 默认可以使用

   ​	D：mmap 无法使用，显示为 Permission Denied

   ​	R: petalinux 2019 为默认 CONFIG_STRICT_DEVMEM=y，导致在 dtsi 里面没有定义的 mem 是无法访问的

   ​	S: 在 linux-kernel 中 的 pln_kernel.cfg 中 添加 CONFIG_STRICT_DEVMEM=n

3. rootfs 无法在 sd 卡启动成功，显示 root=null，读不到 root 设置

   ​	R: config root 没有传到 uboot config 中

   ​	S: 在 system-user.dtsi 中添加

   ```dtsi
      dtsi
         /{
         	chosen{
         		bootargs = "console=ttyPS0,115200 root=/dev/mmcblk0p2 rw earlyprintk";
         	};
         };
     
   ```

   ​    R: 厂家提供的 dtsi 有语法错误，不是 disbale_wp，是 disable-wp 没有生效，非常傻逼

4. remoteproc: 放在 chosen 上面

   ```dtsi
   /include/ "system-conf.dtsi"
   / {
       reserved-memory {
           #address-cells = <2>;
           #size-cells = <2>;
           ranges;
           rproc_0_dma_reserved: rproc@3ed40000{
               no-map;
               compatible = "shared-dma-pool";
               reg = <0x0 0x3ed40000 0x0 0x100000>;
           };
           rproc_0_fw_reserved: rproc@3ed00000 {
               no-map;
               reg = <0x0 0x3ed00000 0x0 0x40000>;
           };
             
           rproc_1_fw_reserved: rproc@3ee00000{
               no-map;
               reg = <0x0 0x3ef00000 0x0 0x40000>;
           };
     
           rproc_1_dma_reserved: rproc@3ee40000 {
               compatible = "shared-dma-pool";
               no-map;
               reg = <0x0 0x3ef40000 0x0 0x100000>;
           };
       };
     
       zynqmp-rpu {
           compatible = "xlnx,zynqmp-r5-remoteproc-1.0";
           #address-cells = <2>;
           #size-cells = <2>;
           ranges;
           core_conf = "split";
             
           r5_0: r5@0 {
               #address-cells = <2>;
               #size-cells = <2>;
               ranges;
               memory-region = <&rproc_0_fw_reserved>, <&rproc_0_dma_reserved>;
               pnode-id = <0x7>;
               mboxes = <&ipi_mailbox_rpu0 0>, <&ipi_mailbox_rpu0 1>;
               mbox-names = "tx", "rx";
                 
               tcm_0_a: tcm_0@0 {
                   reg = <0x0 0xFFE00000 0x0 0x10000>;
                   pnode-id = <0xf>;
               };
               tcm_0_b: tcm_0@1 {
                   reg = <0x0 0xFFE20000 0x0 0x10000>;
                   pnode-id = <0x10>;
               };
           };
             
           r5_1: r5@1 {
               #address-cells = <2>;
               #size-cells = <2>;
               ranges;
               memory-region = <&rproc_1_fw_reserved>, <&rproc_1_dma_reserved>;
               pnode-id = <0x8>;
               mboxes = <&ipi_mailbox_rpu1 0>, <&ipi_mailbox_rpu1 1>;
               mbox-names = "tx", "rx";
               r5_1_tcm_a: tcm@ffe90000 {
                   reg = <0x0 0xFFE90000 0x0 0x10000>;
                   pnode-id = <0x11>;
               };
               r5_1_tcm_b: tcm@ffeb0000 {
                   reg = <0x0 0xFFEB0000 0x0 0x10000>;
                   pnode-id = <0x12>;
               };
           };
     
       };
     
       zynqmp_ipi1 {
           compatible = "xlnx,zynqmp-ipi-mailbox";
           interrupt-parent = <&gic>;
           interrupts = <0 29 4>;
           xlnx,ipi-id = <7>;
           #address-cells = <1>;
           #size-cells = <1>;
           ranges;
     
           /* APU<->RPU0 IPI mailbox controller */
           ipi_mailbox_rpu0: mailbox@ff90600 {
               reg = <0xff990600 0x20>,
                     <0xff990620 0x20>,
                     <0xff9900c0 0x20>,
                     <0xff9900e0 0x20>;
               reg-names = "local_request_region",
                       "local_response_region",
                       "remote_request_region",
                       "remote_response_region";
               #mbox-cells = <1>;
               xlnx,ipi-id = <1>;
           };
       };
     
       zynqmp_ipi2 {
           compatible = "xlnx,zynqmp-ipi-mailbox";
           interrupt-parent = <&gic>;
           interrupts = <0 30 4>;
           xlnx,ipi-id = <8>;
           #address-cells = <1>;
           #size-cells = <1>;
           ranges;
           /* APU<->RPU1 IPI mailbox controller */
           ipi_mailbox_rpu1: mailbox@ff3f0b00 {
               reg = <0xff3f0b00 0x20>,
                     <0xff3f0b20 0x20>,
                     <0xff3f0940 0x20>,
                     <0xff3f0960 0x20>;
               reg-names = "local_request_region",
                       "local_response_region",
                       "remote_request_region",
                       "remote_response_region";
               #mbox-cells = <1>;
               xlnx,ipi-id = <2>;
           };
       };
         
   };
   ```

   

5. 

---

# patch-rt

1. 在 petalinux 工程中，project-spec/meta-usr/conf/petalinuxbsp.conf 中添加下列代码，将源码保存下来

   RM_WORK_EXCLUDE += “linux-xlnx”

   RM_WORK_EXCLUDE += “u-boot-xlnx”

   并复制到其他地方，petalinux 在编译完会自动删除

2. 使用 uname -r 查看 linux 系统版本，下载对应版本的 rt patch，一定要与版本号对应，release 版本不一定能用

3. patch -p1 < patch.linux 到 kernel-source 中，即保存到的 其他位置

4. petalinux-config 中 linux component selection 中选择 linux-xlnx，选择 ext-local，输入 source-kernel 的位置

5. petalinux-config -c kernel 中，选择 preempt，选中 RT

6. build 即可

---

# ipi

```c
 #include <stdio.h>
 #include "platform.h"
 #include "sleep.h"
 
 #include "xparameters.h"
 #include "xscugic.h"
 #include "xil_printf.h"
 #include "xil_exception.h"
 #include "xil_mmu.h"
 
 /*------------------------------------------------------------------------------------------*/
 #define INTC_DEVICE_ID          XPAR_SCUGIC_SINGLE_DEVICE_ID
 #define SHARE_BASE              0xfffc0000
 /*------------------------------------------------------------------------------------------*/
 #define CPU0_ID                 XSCUGIC_SPI_CPU0_MASK
 #define CPU1_ID                 XSCUGIC_SPI_CPU1_MASK
 #define CPU2_ID                 XSCUGIC_SPI_CPU2_MASK
 #define CPU3_ID                 XSCUGIC_SPI_CPU3_MASK
 
 #define SOFT_INTR_ID_TO_CPU0    0
 #define SOFT_INTR_ID_TO_CPU1    1
 #define SOFT_INTR_ID_TO_CPU1    2
 #define SOFT_INTR_ID_TO_CPU1    3
/*------------------------------------------------------------------------------------------*/
XScuGic Intc;
void cpu0_intr_init(XScuGic *intc_ptr);
void soft_intr_handler(void *CallbackRef);
/*------------------------------------------------------------------------------------------*/
int

main(void)
{
  init_platform();
  cpu_intr_init(&Intc);
  while(1) {
    print("cpu 0 is running\r\n");
    sleep(2);
    XScuGic_SoftwareIntr(&Intc, 0, CPU1_ID);
    sleep(3);
  }
  cleanup_platform();
  return 0;
}
/*------------------------------------------------------------------------------------------*/
void
cpu_intr_init(XScuGic *intc_ptr)
{
  XScuGic_Config *intc_cfg_ptr;
  intc_cfg_ptr = XScuGic_LookupConfig(INTC_DEVICE_ID);
  XScuGic_CfgInitialize(intc_ptr, intc_cfg_ptr, intc_cfg_ptr->CpuBaseAddress);
  Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
                               (Xil_ExceptionHandler)XScuGic_InterruptHandler, intc_ptr);
  Xil_ExceptionEnable();

  XScuGic_Connect(intc_ptr, SOFT_INTR_ID_TO_CPU0,
                  (Xil_ExceptionHandler)soft_intr_handler, (void *)intc_ptr);

  XScuGic_Enable(intc_ptr, SOFT_INTR_ID_TO_CPU0);
}
/*------------------------------------------------------------------------------------------*/
void
soft_intr_handler(void *CallbackRef)
{
  (void)CallbackRef;
  print("intr: cpu0\r\n");
}
```







