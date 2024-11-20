Q: BSP 和现在使用 vitis 的区别？

A: 目前的 vitis 更像是一个语法糖，将所有的 BSP 的功能一键生成。所有的地址，时钟等都不需要配置。



Q: 能实现原来那种方式加载吗？

A: 需要再 Linux 环境下，使用 makefile 进行编译出 .bin 文件，通过 jlink 加载到相关地址才行



Q: FreeRTOS 原生版本 和 vitis 上的有什么区别？

A: FreeRTOS 原生版本是在 Linux 环境下编译使用的，vitis 是在 xlinx 环境下，能在 win 上使用的

---

