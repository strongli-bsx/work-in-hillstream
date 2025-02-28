### tips

1. 可以将 version 加入到编译中，在上电运行时可以看到 **version**
1. jf 的代码改了 zynq fsbl 的代码，**记得找供应商问清楚**
1. zynq 的许多外设使用，都有 example，可以先 **import**，在此基础上开发
1. **loopback** 在硬件本身上，会内部打通不会反映在 外部的设备商
1. 当 **指针** 作为参数传递时，传递的是 指针的内容，不是指针的地址，新函数接收时，会创建 *tmp 指向原指针的内容







----

### Q & A

Q: 为什么使用这么多的 宏？

A: 宏本质上是 pattern 替代，c 语言里面基本的功能。使用ta可以实现 **template** 的效果，以及简单的 **逻辑判断** 跟 **代码复用** 等

---

Q: MODULE driver 跟普通 driver 有啥区别？

A: 本质区别不大。在使用上面，可以将不同人开发的不同 模块分层，进行模块化管理

---

Q: UART 传输命令?

A: 使用的中断，**接收完消息中断** 和 **接收消息超时中断**，这样确保了每次都可以重新接收消息，如果不对就丢掉或者 打印出来

---

### linux 全局使用命令工具

1. 将命令所在目录添加到PATH环境变量中：

如将/usr/local/bin添加到PATH中：`export PATH=$PATH:/usr/local/bin` 

2. 利用软链接：

如将/usr/local/go/bin/go命令添加到/usr/local/bin：`ln -s /tools/memctl /usr/local/bin`

3. 修改系统的配置文件：

有些命令的全局使用是通过修改系统的配置文件来实现的

vi /etc/profile（/etc/bashrc）
`export PATH=$PATH:/usr/local/bin`    

`source /etc/profile`

---

### 笔记本安装 ubuntu 18.04.2

1. 使用 rufus 制作 U 盘启动
2. 进去 BIOS，将 security mode 关闭
3. 在 BIOS 中选择 U 盘启动
4. 使用集成 usb 网卡驱动来联网

---

### usb 传输

---

### a系列异常/中断

1. 硬件触发中断，直接进入 asm_vector.S 中的异常中断向量表
2. 在 .S 里面判断异常类型，若是已知，跳转到 vector.c 中
3. 在 vector.c 中，call user 实现的中断函数

---

