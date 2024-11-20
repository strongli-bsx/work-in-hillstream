

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

### reg.h

```c
#ifndef _REG_H_
#define _REG_H_

#include <types.h>

#ifdef __cplusplus
extern "C" {
#endif

/* low level macros for accessing memory mapped hardware registers */
#define REG64(addr) ((volatile uint64_t *)(addr))
#define REG32(addr) ((volatile uint32_t *)(addr))
#define REG16(addr) ((volatile uint16_t *)(addr))
#define REG8(addr)  ((volatile uint8_t *)(addr))

#define RMWREG64(addr, startbit, width, val) *REG64(addr) = (*REG64(addr) & ~(((1 << (width)) - 1) << (startbit))) | ((val) << (startbit))
#define RMWREG32(addr, startbit, width, val) *REG32(addr) = (*REG32(addr) & ~(((1 << (width)) - 1) << (startbit))) | ((val) << (startbit))
#define RMWREG16(addr, startbit, width, val) *REG16(addr) = (*REG16(addr) & ~(((1 << (width)) - 1) << (startbit))) | ((val) << (startbit))
#define RMWREG8(addr, startbit, width, val) *REG8(addr) = (*REG8(addr) & ~(((1 << (width)) - 1) << (startbit))) | ((val) << (startbit))

#define writel(v, a)   (*REG32(a) = (v))
#define readl(a)       (*REG32(a))

#define writeb(v, a)   (*REG8(a) = (v))
#define readb(a)       (*REG8(a))

#define writehw(v, a)  (*REG16(a) = (v))
#define readhw(a)      (*REG16(a))

#define writew(v, a)   (*REG16(a) = (v))
#define readw(a)       (*REG16(a))

static inline void
__raw_writesl(unsigned long addr, const void *data, int longlen)
{
  uint32_t *buf = (uint32_t *)data;

  while(longlen--) {
    writel(*buf++, addr);
  }
}
static inline void
__raw_readsl(unsigned long addr, void *data, int longlen)
{
  uint32_t *buf = (uint32_t *)data;

  while(longlen--) {
    *buf++ = readl(addr);
  }
}
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

/*------------------------------*/    
    
#ifndef TYPES_H
#define TYPES_H

#include <stdint.h>

typedef uint8_t __u8;
typedef uint16_t __u16;
typedef uint32_t __u32;
typedef uint64_t __u64;

#endif /* TYPES_H */
    
    
```

