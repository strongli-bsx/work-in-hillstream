

### str_to_int

```c
static uint32_t
str_to_int(char *p)
{
  uint32_t val = 0, i = 0;
  if(p[i] == '0') {
    i++;
    if(p[i] == '\0') {
      return 0;
    } else if(p[i] == 'x') {
      i++;
    }
    /* test write mode */
  } else if(p[i] == '%') {
    return (uint32_t)p[i];
  }

  while(p[i] != '\0') {
    if(p[i] >= '0' && p[i] <= '9') {
      val = (val << 4) | (p[i] - '0');
    } else if(p[i] >= 'A' && p[i] <= 'F') {
      val = (val << 4) | (10 + p[i] - 'A');
    } else if(p[i] >= 'a' && p[i] <= 'f') {
      val = (val << 4) | (10 + p[i] - 'a');
    } else {
      return p[i];
    }
    i++;
  }

  return val;
}
```

---

### log

```c
#include <stdio.h>

#define LOG(levelstr, ...)            do{   \
    LOG_OUTPUT("%-4s", levelstr);            \
    LOG_OUTPUT(__VA_ARGS__);                 \
} while(0)

#define LOG_OUTPUT(...)                printf(__VA_ARGS__)
#define LOG_INFO(...)                  LOG("[INFO]: ",__VA_ARGS__)
#define LOG_ERR(...)                   LOG("[ERR ]: ", __VA_ARGS__)

int main(void){


    LOG_OUTPUT("test-st\n");
    LOG_INFO("info-st\n");
    LOG_ERR("err-st\n");
    return 0;
}
```

---

### mem-r/w/l/s

```c
/* remeber set stack 102400 k size */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <signal.h>
#include <fcntl.h>
#include <ctype.h>
#include <termios.h>
#include <sys/types.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <stdbool.h>
#include <stdint.h>
#include "log.h"

#define FATAL()       do { \
          fprintf(stderr, "Error at line %d, file %s (%d) [%s]\n", \
                  __LINE__, __FILE__, errno, strerror(errno)); \
          exit(1); \
} while(0)
/*---------------------------------------------------------------------------*/
#define DEVMEM          "/dev/mem"
#define FILENAME        "data.dat"
#define FILE_END        ('E' << 16 | 'N' << 8 | 'D')
#define MAP_SIZE        0x1000000
#define MAP_MASK        (MAP_SIZE - 1)
#define CMD_ARR_MAX      16
#define BUF_SIZE        (MAP_SIZE >> 2)
/*---------------------------------------------------------------------------*/
enum cmd_type {
  READ,
  WRITE,
  LOAD_TO_MEM,
  SAVE_TO_FILE,
  CMD_TYPE_MAX,
};
/*---------------------------------------------------------------------------*/
typedef struct {
  uint32_t addr;
  uint32_t buffer[BUF_SIZE];
  uint32_t arg;
}mem_data_t;
/*---------------------------------------------------------------------------*/
typedef void (*memctl_ptr_t)(mem_data_t *data);
/*---------------------------------------------------------------------------*/
typedef struct {
  char *cmd;
  int type;
  memctl_ptr_t func;
}type_cmd;
/*---------------------------------------------------------------------------*/
static void devmem_read(mem_data_t *data);
static void devmem_write(mem_data_t *data);
static void load_to_mem(mem_data_t *datan);
static void save_to_file(mem_data_t *data);
/*---------------------------------------------------------------------------*/
type_cmd cmd_arr[CMD_ARR_MAX] = {
  { "read", READ, devmem_read },
  { "r", READ, devmem_read },
  { "write", WRITE, devmem_write },
  { "w", WRITE, devmem_write },
  { "load", LOAD_TO_MEM, load_to_mem },
  { "l", LOAD_TO_MEM, load_to_mem },
  { "save", SAVE_TO_FILE, save_to_file },
  { "s", SAVE_TO_FILE, save_to_file },
};
/*---------------------------------------------------------------------------*/
void
seek_type(type_cmd *cmd_arr, type_cmd *cmd)
{
  for(int i = 0; i < CMD_ARR_MAX; i++) {
    if(strcmp(cmd_arr[i].cmd, cmd->cmd) == 0) {
      cmd->type = cmd_arr[i].type;
      cmd->func = cmd_arr[i].func;
      break;
    }
  }
}
/*---------------------------------------------------------------------------*/
static void
devmem_ug(void)
{
  LOG_WARN("UG: memctl r/l/s 0x3ed48000 20.\n");
  LOG_WARN("UG: memctl w 0x3ed48000 0x12345678.\n");
  LOG_WARN("UG: represent r-read, w-write, l-load_to_mem, s-save_to_file.\n");
  LOG_WARN("UG: The 0x3ed48000 is the memory physical address.\n");
  LOG_WARN("UG: 20 is 0x20 size, but 0x12345678 is data!\n");
}
/*---------------------------------------------------------------------------*/
static void
flush_buffer(mem_data_t *data)
{
  for(int i = 0; i < (data->arg >> 2); i++) {
    printf("addr = 0x%08x, data = 0x%08x\n", (data->addr + i * 4), data->buffer[i]);
  }
}
/*---------------------------------------------------------------------------*/
static void
input_write_buffer(mem_data_t *data)
{
  if(data->arg != (uint32_t)'%') {
    data->buffer[0] = data->arg;
    data->arg = 4;
  } else {
    data->arg = 0x20 * 4;
    for(int i = 0; i < 0x20; i++) {
      data->buffer[i] = i;
    }
  }
}
/*---------------------------------------------------------------------------*/
static int
read_mem(mem_data_t *data)
{
  uint32_t fd;
  uint32_t length = data->arg;
  off_t addr = data->addr;
  uint32_t offset_len = data->addr;
  uint32_t *buf = data->buffer;
  void *map_base, *virtual_addr;

  if((fd = open(DEVMEM, O_RDWR | O_SYNC)) == -1) {
    LOG_ERR("open /dev/mem failed!\n");
    FATAL();
    return -1;
  }

  /* Map one page */
  map_base = mmap(0, MAP_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, addr & ~MAP_MASK);
  if(map_base == (void *)-1) {
    FATAL();
    close(fd);
    return -1;
  }

  for(uint32_t i = 0; i < (length >> 2); i++) {
    /* if addr == offset_len, then map_base will add MAP_SIZE first */
    if(((offset_len & MAP_MASK) == 0) && (addr != offset_len)) {
      if(munmap(map_base, MAP_SIZE) == -1) {
        FATAL();
        close(fd);
        return -1;
      }
      map_base = mmap(0, MAP_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, offset_len & ~MAP_MASK);
      if(map_base == (void *)-1) {
        FATAL();
        close(fd);
        return -1;
      }
    }
    virtual_addr = map_base + (offset_len & MAP_MASK);
    buf[i] = *((uint32_t *)virtual_addr);
    offset_len += 4;
  }
  if(munmap(map_base, MAP_SIZE) == -1) {
    FATAL();
    close(fd);
    return -1;
  }
  close(fd);
  return 0;
}
/*---------------------------------------------------------------------------*/
static int
write_mem(mem_data_t *data)
{
  uint32_t fd;
  uint32_t len = data->arg;
  uint32_t addr = data->addr;
  uint32_t offset_len = data->addr;
  uint32_t *buf = data->buffer;
  void *map_base, *virtual_addr;

  if((fd = open(DEVMEM, O_RDWR | O_SYNC)) == -1) {
    LOG_ERR("open /dev/mem failed!\n");
    FATAL();
    return -1;
  }
  /* Map one page */
  map_base = mmap(0, MAP_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, addr & ~MAP_MASK);
  if(map_base == (void *)-1) {
    FATAL();
    close(fd);
    return -1;
  }
  for(int i = 0; i < (len >> 2); i++) {
    if(!(offset_len & MAP_MASK) && (addr != offset_len)) {
      if(munmap(map_base, MAP_SIZE) == -1) {
        FATAL();
        close(fd);
        return -1;
      }
      map_base = mmap(0, MAP_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, offset_len & ~MAP_MASK);
      if(map_base == (void *)-1) {
        FATAL();
        close(fd);
        return -1;
      }
    }

    virtual_addr = map_base + (offset_len & MAP_MASK);
    *((uint32_t *)virtual_addr) = buf[i];
    offset_len += 4;
  }
  if(munmap(map_base, MAP_SIZE) == -1) {
    FATAL();
    return -1;
  }

  close(fd);
  return 0;
}
/*---------------------------------------------------------------------------*/
static void
devmem_read(mem_data_t *data)
{
  read_mem(data);
  flush_buffer(data);
}
/*---------------------------------------------------------------------------*/
static void
devmem_write(mem_data_t *data)
{
  input_write_buffer(data);
  write_mem(data);
}
/*---------------------------------------------------------------------------*/
static uint32_t
calculate_log2(uint32_t num)
{
  int result = 0;
  if(num == 1) {
    return 0;
  }
  while(num > 1) {
    num >>= 1;
    result++;
  }
  return result;
}
/*---------------------------------------------------------------------------*/
static void
load_to_mem(mem_data_t *data)
{
  uint32_t fd;
  uint32_t *dst;
  uint32_t len = data->arg;
  uint32_t *buf = data->buffer;
  uint32_t cnt = 1;
  void *map_base;

  if((fd = open(FILENAME, O_RDWR | O_CREAT, 0777)) == -1) {
    LOG_ERR("open file failed!\n");
    FATAL();
    close(fd);
    return;
  }
  if(len > MAP_SIZE) {
    cnt = len >> calculate_log2(MAP_SIZE);
    len = MAP_SIZE;
    data->arg = len;
  }
  for(int j = 0; j < cnt; j++) {
    if(lseek(fd, (len - 1) + (len * j), SEEK_SET) == -1) {
      LOG_ERR("lseek failed\n");
      close(fd);
      return;
    }
    if(write(fd, "\0", 1) == -1) {
      FATAL();
      close(fd);
      return;
    }
    map_base = mmap(0, len, PROT_WRITE | PROT_READ, MAP_SHARED, fd, len * j);

    if(map_base == (void *)-1) {
      FATAL();
      return;
    }
    dst = (uint32_t *)map_base;
    for(int i = 0; i < (len >> 2); i++) {
      buf[i] = *dst++;
    }
    write_mem(data);
    data->addr += len;
  }
  close(fd);
  munmap(map_base, len);
}
/*---------------------------------------------------------------------------*/
static void
save_to_file(mem_data_t *data)
{
  uint32_t fd;
  uint32_t *dst;
  uint32_t len = data->arg;
  uint32_t *buf = data->buffer;
  uint32_t cnt = 1;
  void *map_base;

  if((fd = open(FILENAME, O_RDWR | O_CREAT, 0777)) == -1) {
    LOG_ERR("open file failed!\n");
    FATAL();
    return;
  }

  if(len > MAP_SIZE) {
    cnt = len >> calculate_log2(MAP_SIZE);
    len = MAP_SIZE;
    data->arg = len;
  }
  for(int j = 0; j < cnt; j++) {
    if(lseek(fd, (len - 1) + (len * j), SEEK_SET) == -1) {
      LOG_ERR("lseek failed\n");
      close(fd);
      return;
    }
    if(write(fd, "\0", 1) == -1) {
      LOG_ERR("write failed after lseek!\n");
      close(fd);
      return;
    }
    map_base = mmap(0, len, PROT_WRITE | PROT_READ, MAP_SHARED, fd, len * j);

    if(map_base == (void *)-1) {
      FATAL();
      return;
    }
    dst = (uint32_t *)map_base;

    read_mem(data);
    data->addr += len;
    for(int i = 0; i < (len >> 2); i++) {
      *dst++ = buf[i];
    }
  }
  munmap(map_base, len);
  close(fd);
}
/*---------------------------------------------------------------------------*/
static uint32_t
str_to_int(char *p)
{
  uint32_t val = 0, i = 0;
  if(p[i] == '0') {
    i++;
    if(p[i] == '\0') {
      return 0;
    } else if(p[i] == 'x') {
      i++;
    }
    /* test write mode */
  } else if(p[i] == '%') {
    return (uint32_t)p[i];
  }

  while(p[i] != '\0') {
    if(p[i] >= '0' && p[i] <= '9') {
      val = (val << 4) | (p[i] - '0');
    } else if(p[i] >= 'A' && p[i] <= 'F') {
      val = (val << 4) | (10 + p[i] - 'A');
    } else if(p[i] >= 'a' && p[i] <= 'f') {
      val = (val << 4) | (10 + p[i] - 'a');
    } else {
      return p[i];
    }
    i++;
  }

  return val;
}
/*---------------------------------------------------------------------------*/
int
main(int argc, char **argv)
{
  type_cmd receive_cmd;
  mem_data_t mem_data;

  if(argc != 4) {
    LOG_ERR("The argc is 4!\n");
    devmem_ug();
    return 0;
  }

  receive_cmd.cmd = argv[1];
  mem_data.addr = str_to_int(argv[2]);
  mem_data.arg = str_to_int(argv[3]);
  memset(mem_data.buffer, 0, sizeof(mem_data.buffer));
  seek_type(cmd_arr, &receive_cmd);
  receive_cmd.func(&mem_data);
  return 0;
}
/*---------------------------------------------------------------------------*/

```

---

### shell: loop

```shell
for i in {1..500}
do
	sleep 0.05s
	echo "numer $i"
done
################################
# print logs at custom intervals
################################
sum=0
for i in {1..100}
do
  sum=$(expr $i % 5)
if [ $sum -eq 0 ]
then echo $i
fi
done

```

----

### reg.h

```c
#ifndef _REG_H_
#define _REG_H_

#include <types.h>

#ifdef __cplusplus
extern "C" {
#endif

/* low level macros for accessing memory mapped hardware registers */
#define REG64(addr) 			(*((volatile uint64_t *)(addr)))
#define REG32(addr) 			(*((volatile uint32_t *)(addr)))
#define REG16(addr) 			(*((volatile uint16_t *)(addr)))
#define REG8(addr)  			(*((volatile uint8_t *)(addr)))

#define reg32_write(a, v)      (REG32(a) = (v))
#define reg32_read(a, v)       ((v) = REG32(a))

#define reg16_write(a, v)      (REG16(a) = (v))
#define reg16_read(a, v)       ((v) = REG16(a))
    
#define reg8_write(a, v)       (REG8(a) = (v))
#define reg8_read(a, v)        ((v) = REG8(a))


/**
 * @def SETBIT
 * @brief Sets a bitmask for a bitfield
 *
 * @param[in] val   The bitfield
 * @param[in] bit   Specifies the bits to be set
 *
 * @return The modified bitfield
 */
#define SETBIT(val, bit)    val |= (bit)

/**
 * @def CLRBIT
 * @brief Clears bitmask for a bitfield
 *
 * @param[in] val   The bitfield
 * @param[in] bit   Specifies the bits to be cleared
 *
 * @return The modified bitfield
 */
#define CLRBIT(val, bit)    val &= (~(bit))

#ifndef BIT0
#define BIT0  0x00000001
#define BIT1  0x00000002
#define BIT2  0x00000004
#define BIT3  0x00000008
#define BIT4  0x00000010
#define BIT5  0x00000020
#define BIT6  0x00000040
#define BIT7  0x00000080
#define BIT8  0x00000100
#define BIT9  0x00000200
#define BIT10 0x00000400
#define BIT11 0x00000800
#define BIT12 0x00001000
#define BIT13 0x00002000
#define BIT14 0x00004000
#define BIT15 0x00008000
#endif
#ifndef BIT16
#define BIT16 0x00010000
#define BIT17 0x00020000
#define BIT18 0x00040000
#define BIT19 0x00080000
#define BIT20 0x00100000
#define BIT21 0x00200000
#define BIT22 0x00400000
#define BIT23 0x00800000
#define BIT24 0x01000000
#define BIT25 0x02000000
#define BIT26 0x04000000
#define BIT27 0x08000000
#define BIT28 0x10000000
#define BIT29 0x20000000
#define BIT30 0x40000000
#define BIT31 0x80000000
#endif

#ifdef __cplusplus
extern "C" {
#endif

#endif /* _REG_H_ */
    
```

---

### module driver

```c
#include <linux/module.h>

#include <linux/fs.h>
#include <linux/errno.h>
#include <linux/miscdevice.h>
#include <linux/kernel.h>
#include <linux/major.h>
#include <linux/mutex.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/stat.h>
#include <linux/init.h>
#include <linux/device.h>
#include <linux/tty.h>
#include <linux/kmod.h>
#include <linux/gfp.h>

/* 1. 确定主设备号 */
static int major = 0;
static char kernel_buf[1024];
static struct class *hello_class;

#define MIN(a, b) (a < b ? a : b)

/* 3. 实现对应的open/read/write等函数，填入file_operations结构体 */

/*
 * @description		: 从设备读取数据
 * @param - file	: 内核中的文件描述符
 * @param - buf		: 要存储读取的数据缓冲区（就是用户空间的内存地址）
 * @param - size	: 要读取的长度
 * @param - offset	: 相对于文件首地址的偏移量（一般读取信息后，指针都会偏移读取信息的长度）
 * @return 			: 返回读取的字节数，如果读取失败则返回-1
 */
static ssize_t hello_drv_read (struct file *file, char __user *buf, size_t size, loff_t *offset)
{
	int err;
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	err = copy_to_user(buf, kernel_buf, MIN(1024, size));
	return MIN(1024, size);
}

/*
 * @description		: 向设备写数据
 * @param - file	: 内核中的文件描述符
 * @param - buf		: 要写给设备驱动的数据缓冲区
 * @param - size	: 要写入的长度
 * @param - offset	: 相对于文件首地址的偏移量
 * @return 			: 返回写入的字节数，如果写入失败则返回-1
 */
static ssize_t hello_drv_write (struct file *file, const char __user *buf, size_t size, loff_t *offset)
{
	int err;
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	err = copy_from_user(kernel_buf, buf, MIN(1024, size));
	return MIN(1024, size);
}

/*
 * @description		: 打开设备
 * @param - node	: 设备节点
 * @param - file	: 文件描述符
 * @return 			: 打开成功返回0，失败返回-1
 */
static int hello_drv_open (struct inode *node, struct file *file)
{
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	return 0;
}

/*
 * @description		: 关闭设备
 * @param - node	: 设备节点
 * @param - file	: 文件描述符
 * @return 			: 关闭成功返回0，失败返回-1
 */
static int hello_drv_close (struct inode *node, struct file *file)
{
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	return 0;
}

/* 2. 定义自己的file_operations结构体                                              */
static struct file_operations hello_drv = {
	.owner	 = THIS_MODULE,
	.open    = hello_drv_open,
	.read    = hello_drv_read,
	.write   = hello_drv_write,
	.release = hello_drv_close,
};

/* 4. 把file_operations结构体告诉内核：注册驱动程序                                */
/* 5. 谁来注册驱动程序啊？得有一个入口函数：安装驱动程序时，就会去调用这个入口函数 		*/
static int __init hello_init(void)
{
	int err;
	
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	major = register_chrdev(0, "hello", &hello_drv);  /* /dev/hello */

	/* 7. 其他完善：提供设备信息，自动创建设备节点                                 */
	hello_class = class_create(THIS_MODULE, "hello_class");
	err = PTR_ERR(hello_class);
	if (IS_ERR(hello_class)) {
		printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
		unregister_chrdev(major, "hello");
		return -1;
	}	
	device_create(hello_class, NULL, MKDEV(major, 0), NULL, "hello"); /* /dev/hello */
	
	return 0;
}

/* 6. 有入口函数就应该有出口函数：卸载驱动程序时，就会去调用这个出口函数           */
static void __exit hello_exit(void)
{
	printk("%s %s line %d\n", __FILE__, __FUNCTION__, __LINE__);
	device_destroy(hello_class, MKDEV(major, 0));
	class_destroy(hello_class);
	unregister_chrdev(major, "hello");
}

/* 指定驱动的入口和出口，以及声明自己的驱动遵循GPL协议（不声明的话无法把驱动加载进内核） */
module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");

```

---

### version number

- 可以在 build setting 中 pre-build setps 将 main.o 删掉，重新编译一下调用 time 的地方

- 如下面添加脚本

  

```makefile
# makfile

VERSION_STRING := "v0.1"
# COMPILE_TIME = $(shell date +"%Y-%m-%d")
#COMPILE_DATE = `date +"%Y-%m-%d-%H-%M"`
# DATE_STRING = -DCOMPILE_TIME=\"$(COMPILE_DATE)\"
# VER_STRING = -DVERSION=\"$(VERSION_STRING)\"

COMPILE_DATE = `date +"%Y-%m-%d-%H-%M"`
CFLAGS += -DCOMPILE_TIME=\"$(COMPILE_DATE)\"
CFLAGS += -DVERSION=\"$(VERSION_STRING)\"

.PHONY:all
all:compile_time
compile_time:main.o
	gcc $^ -o $@
	@rm main.o
# t:main.c
# mem.o : mem.c
# main.o : main.c
# rffe.o : rffe.c

clean:
	rm -f  main.o compile_time

##-----------------------------------------------##
 printf("version is " VERSION "\n");
 printf("compile time is " COMPILE_TIME "\n");
 
```

```shell
#!/bin/bash

str_front='#define SW_VERSION'
quotation="\""
version_file=version.h

rm $version_file
echo $str_front $quotation$(date '+%Y-%m-%d-%H')$quotation >> $version_file 


```

```
set str_front=#define SW_VERSION
set quotation= "
set version_file=version.h
set real_time=%date:~2,2%-%date:~5,2%-%date:~8,2%-%time:~0,2%
@REM @echo %real_time%

del %version_file%
echo %str_front%   %quotation%%real_time%%quotation% > %version_file% 
pause
```

---

### read file

```c
int
read_data_from_file(uint32_t size)
{
  /* read bin */
  FILE *p = fopen("data.bin", "rb");

  /* receive data buff */
  uint32_t buf[size] = { 0 };
  /* uint32_t* target_addr = (uint32_t *)0x40000000; */

  printf("read start\n");
  /* $1: dst_addr */
  /* $2: read basic len */
  /* $3: read num of basic len */
  /* $4: file *p */
  fread(buf, sizeof(uint8_t), 10, p);
  printf("read end\n");
  return 0;
}
```

---

### format txt

```bat
@echo off
set TheStart=0x
set TheEnd=,
(for /f "delims=" %%i in ('type 1.txt') do (
    echo %TheStart%%%i%TheEnd%
))>2.txt
```

---

### uart_lite

```c
#include "xparameters.h"
#include "xuartps.h"
#include "xscugic.h"
#include <string.h>
#include <stdio.h>
#include "uart_lite.h"
#include "cmd_func.h"


/*------------------------------------------------------------------*/
/*------------------------------------------------------------------*/
#define INTC            XScuGic
#define UART_DEVICE_ID    XPAR_XUARTPS_0_DEVICE_ID
#define INTC_DEVICE_ID    XPAR_SCUGIC_SINGLE_DEVICE_ID
#define UART_INT_IRQ_ID   XPAR_XUARTPS_0_INTR
/*------------------------------------------------------------------*/
XUartPs uart;
XUartPs_Config *uart_conf_ptr;
INTC InterruptController;
/*------------------------------------------------------------------*/
static uint8_t uart_buffer[BUFFER_SIZE];
static uint8_t token[64][64];
/*------------------------------------------------------------------*/
/*------------------------------------------------------------------*/
static void uart_cmd_spilt(const char *cmd, char **token)
{
  const char *delim = ",";
  uint32_t cnt = 0;

  token[cnt] = strtok((char *)cmd, delim);
  /* printf("first token: %s\n\r", token[cnt]); */

  while(token[cnt++] != NULL) {
    token[cnt] = strtok(NULL, delim);
    /* printf("%s\n\r", token[cnt]); */
  }
}
/*------------------------------------------------------------------*/
/* re-receive data each time  */
static void uart_irq(void *CallBackRef, u32 Event, unsigned int EventData)
{
  if((Event == XUARTPS_EVENT_RECV_DATA) || (Event == XUARTPS_EVENT_RECV_TOUT))
  {
    uart_cmd_spilt(uart_buffer, token);
    cmd_parse(token);
    /* test loop: send and receive data correct with diff len */
    /* uart_send(uart_buffer, EventData); */
    uart_receive(uart_buffer, BUFFER_SIZE);
  }
}
/*------------------------------------------------------------------*/
static void uart_set_baudrate(uint32_t baudrate)
{
  XUartPs_SetBaudRate(&uart, baudrate);
}
/*------------------------------------------------------------------*/
static int SetupInterruptSystem(INTC *IntcInstancePtr,
                                XUartPs *UartInstancePtr,
                                u16 UartIntrId)
{
  XScuGic_Config *IntcConfig;

  IntcConfig = XScuGic_LookupConfig(INTC_DEVICE_ID);
  XScuGic_CfgInitialize(IntcInstancePtr,
                        IntcConfig,
                        IntcConfig->CpuBaseAddress);
  Xil_ExceptionRegisterHandler(XIL_EXCEPTION_ID_INT,
                               (Xil_ExceptionHandler)XScuGic_InterruptHandler,
                               IntcInstancePtr);
  XScuGic_Connect(IntcInstancePtr, UartIntrId,
                  (Xil_ExceptionHandler)XUartPs_InterruptHandler,
                  (void *)UartInstancePtr);

  XScuGic_Enable(IntcInstancePtr, UartIntrId);

  Xil_ExceptionEnable();

  return XST_SUCCESS;
}
/*------------------------------------------------------------------*/
static int uart_interrupt_init(void)
{
  SetupInterruptSystem(&InterruptController, &uart, UART_INT_IRQ_ID);
  XUartPs_SetHandler(&uart, (XUartPs_Handler)uart_irq, &uart);

  uint32_t IntrMask =
    XUARTPS_IXR_TOUT | XUARTPS_IXR_PARITY | XUARTPS_IXR_FRAMING |
    XUARTPS_IXR_OVER | XUARTPS_IXR_TXEMPTY | XUARTPS_IXR_RXFULL |
    XUARTPS_IXR_RXOVR;
  /* IntrMask |= XUARTPS_IXR_RBRK; */

  XUartPs_SetInterruptMask(&uart, IntrMask);
  XUartPs_SetRecvTimeout(&uart, 8);
  return 0;
}
/*------------------------------------------------------------------*/
int uart_init(void)
{
  uart_conf_ptr = XUartPs_LookupConfig(DEVICE_ID);
  XUartPs_CfgInitialize(&uart, uart_conf_ptr, uart_conf_ptr->BaseAddress);
  uart_set_baudrate(UART_BAUD);
  uart_interrupt_init();
  return 0;
}
/*------------------------------------------------------------------*/
int uart_send(uint8_t *buffer, uint32_t len)
{
  XUartPs_Send(&uart, buffer, len);
  return 0;
}
/*------------------------------------------------------------------*/
int uart_receive(uint8_t *buffer, uint32_t len)
{
  XUartPs_Recv(&uart, buffer, len);
  return 0;
}
/*------------------------------------------------------------------*/

```

---

### cmd_parse

```c
#include "cmd_func.h"
#include <stdio.h>
#include "xil_printf.h"

/* Introduction:
 *
 * CMD: 1.SHOULD NOT HAVE SPACE
 *      2.VALUE IS HEX
 *      3.THE LAST ',' SHOULD NOT FORGET
 *
 * 1). reg,r,[addr],[size],
 * eg. reg,r,0x40000000,1,
 * 2). reg,w,[addr],[val],
 * eg. reg,w,0x40000000,3,
 * FUNC:
 * 1. func,[num],
 * eg. func,1,
 *
 * */

/*------------------------------------------------------------------*/
typedef void (*func_t)(void);
/*------------------------------------------------------------------*/
/*-------------------------add new func here------------------------*/
/*------------------------------------------------------------------*/
void func0(void)
{
  print("there is func0\r\n");
}
void func1(void)
{
  print("there is func1\r\n");
}
void func2(void)
{
  print("there is func2\r\n");
}
void func3(void)
{
  print("there is func3\r\n");
}
/*------------------------------------------------------------------*/
/*-----------------add func name in follow arr----------------------*/
/*------------------------------------------------------------------*/
static func_t cmd_func_arr[] = {
  /* 0 */ func0,
  /* 1 */ func1,
  /* 2 */ func2,
  /* 3 */ func3,
};
/*------------------------------------------------------------------*/
/*------------------------------------------------------------------*/
static uint32_t str_to_int(uint8_t *p)
{
  uint32_t val = 0, i = 0;
  if(p[i] == '0')
  {
    i++;
    if(p[i] == '\0')
    {
      return 0;
    } else if(p[i] == 'x') {
      i++;
    }
    /* test write mode */
  } else if(p[i] == '%') {
    return (uint32_t)p[i];
  }

  while(p[i] != '\0') {
    if(p[i] >= '0' && p[i] <= '9')
    {
      val = (val << 4) | (p[i] - '0');
    } else if(p[i] >= 'A' && p[i] <= 'F') {
      val = (val << 4) | (10 + p[i] - 'A');
    } else if(p[i] >= 'a' && p[i] <= 'f') {
      val = (val << 4) | (10 + p[i] - 'a');
    } else {
      return p[i];
    }
    i++;
  }
  return val;
}
/*------------------------------------------------------------------*/
void func_common(uint8_t **cmd)
{
  uint32_t num = str_to_int(cmd[1]);
  /* printf("num = %d\n", num); */
  uint32_t size = sizeof(cmd_func_arr) / sizeof(func_t);
  /* printf("size = %p\n\r", size); */
  if(num > size)
  {
    return;
  }
  cmd_func_arr[num]();
}
/*------------------------------------------------------------------*/
/*----------------------------reg common----------------------------*/
/*------------------------------------------------------------------*/
#define REG32(a)             (*((volatile uint32_t *)(a)))
#define REG32_READ(a, v)        ((v) = REG32(a))
#define REG32_WRITE(a, v)       (REG32(a) = (v))
/*------------------------------------------------------------------*/
typedef enum {
  REG,
  FUNC,
  CMD_MAX,
}CMD_NAME;
/*------------------------------------------------------------------*/
typedef struct {
  CMD_NAME type;
  uint8_t *cmd;
}cmd_type_t;
/*------------------------------------------------------------------*/
static cmd_type_t cmd_type[CMD_MAX] = {
  { REG, "reg" },
  { FUNC, "func" },
};

/*------------------------------------------------------------------*/
static void reg_common(uint8_t **cmd)
{
  uint8_t *type_of_cmd = cmd[1];
  uint32_t addr = str_to_int(cmd[2]);
  uint32_t val = str_to_int(cmd[3]);

  if(!strcmp(type_of_cmd, "w"))
  {
    REG32_WRITE(addr, val);
    printf("write done!\n\r");
    REG32_READ(addr, val);
    printf("addr = %p, val = %p\n\r", addr, val);
  } else if(!strcmp(type_of_cmd, "r")) {
    uint32_t size = val;
    for(int i = 0; i < size; i++) {
      REG32_READ(addr, val);
      printf("addr = %p, val = %p\n\r", addr, val);
      addr += 4;
    }
  }
}
/*------------------------------------------------------------------*/
static uint32_t seek_cmd_type(uint8_t *cmd)
{
  for(int i = 0; i < CMD_MAX; i++) {
    if(!strcmp(cmd_type[i].cmd, cmd))
    {
      return cmd_type[i].type;
    }
  }
  return -1;
}
/*------------------------------------------------------------------*/
void cmd_parse(uint8_t **cmd)
{
  uint8_t *cmd_head = cmd[0];
  uint32_t type = seek_cmd_type(cmd_head);
  switch(type) {
  case REG:
    reg_common(cmd);
    break;
  case FUNC:
    func_common(cmd);
  default:
    break;
  }
}
/*------------------------------------------------------------------*/

```



---

