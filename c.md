# c 使用总结

- `__func__`:

  ```c
  //_func_ : const char type array, storage the current function name.
  //_FILE_ : string type, storage file name.
  //_TIME_ : string type, storage compile time.
  //_LINE_ : int type, storage line number.
  //_DATE_ : string type, storage complie data
  ```

- `_builtin_clz`“：返回前面引导位 0 的个数，例 0->31, 2->30

- void:

  ```c
  void *p; // 无类型指针，可指向任意类型的数据
  (void) data; // 表示下面未使用该变量
  ```

- .S和.s这两种文件都是汇编代码，其区别在于：.s 格式的汇编文件中， **只能包含纯粹的汇编代码** ，汇编器只对其进行汇编操作，没有预处理操作；.S 格式的汇编文件中， 还可以使用预处理命令 ，汇编器会先进行预处理，然后再进行汇编

- 

- bool类型，返回值用TRUE跟FALSE，不用 0 和 1

  ---

- 内联汇编：

  ```c
  _asm__ __violate__ ("movl %1,%0" : "=r" (result) : "m" (input));
  //`movl %1,%0`是指令模板；
  //`%0`和`%1`代表指令的操作数，称为占位符。
  //内嵌汇编靠它们将 C 语言表达式与指令操作数相对应。
  //指令模板后面用小括号括起来的是C语言表达式，
  //本例中只有两个：`result`和`input`，他们按照出现的顺序分别与指令操作数`%0`，`%1`对应；
  //注意对应顺序：第一个C表达式对应`%0`；第二个表达式对应`%1`，
  //依次类推。操作数至多有10 个，分别用`%0`,`%1`….`%9`表示。
  //在每个操作数前面有一个用引号括起来的字符串，字符串的内容是对该操作数的限制或者说要求。 
  //`result`前面的限制字符串是`=r`，其中`=`表示`result`是输出操作数，`r` 表示需要将`result`与某个通用寄存器相关联，
  //先将操作数的值读入寄存器，然后在指令中使用相应寄存器，而不是`result`本身，
  //当然指令执行完后需要将寄存器中的值存入变量`result`，从表面上看好像是指令直接对`result`进行操作，
  //实际上GCC做了隐式处理，这样我们可以少写一 些指令。`input`前面的`m`表示该表达式需要先放入某个寄存器，然后在指令中使用该寄存器参加运算。
  
  //a,b,c,d,S,D 分别代表 eax,ebx,ecx,edx,esi,edi 寄存器
  //r 上面的寄存器的任意一个（谁闲着就用谁）
  //m 内存
  //i 立即数（常量，只用于输入操作数）
  //g 寄存器、内存、立即数 都行（gcc你看着办）
  ```

  ---

- 位域：

  ```c
  struct Date 
  {
     unsigned nWeekDay  : 3;    // 1..7   (3 bits)
     unsigned nMonthDay : 6;    // 1..31  (6 bits)
     unsigned           : 0;    // Force alignment to next boundary.
     unsigned nMonth    : 5;    // 1..12  (5 bits)
     unsigned nYear     : 8;    // 1..100 (8 bits)
  };
  //分配内存是按照既定好的位数
  //先忽略占位为0的代码
  //short类型为16位，则3+6+5+8>16, 超过16位则最后8占位不能连续存储，需从新的开始xxxxxxxxxx111 1struct Date 2{3   unsigned nWeekDay  : 3;    // 1..7   (3 bits)4   unsigned nMonthDay : 6;    // 1..31  (6 bits)5   unsigned           : 0;    // Force alignment to next boundary.6   unsigned nMonth    : 5;    // 1..12  (5 bits)7   unsigned nYear     : 8;    // 1..100 (8 bits)8};9//分配内存是按照既定好的位数10//先忽略占位为0的代码11//short类型为16位，则3+6+5+8>16, 超过16位则最后8占位不能连续存储，需从新的开始c
  ```

- 


---

- 伪指令意味着它并不是一条真正的指令，而是对其他基本指令使用形式的一种别名，类似于宏
- 
- 以**单下划线**（_）表明是**标准库**的变量，而**双下划线**（__） 开头表明是**编译器**的变量
- U: Unsigned int 的简写

---

函数指针: 无论是否解引用都可以执行

```c
//函数指针
int(*p)(int, int); // p 的数据类型是 int (*)(int, int)	
p = func;         //p-->func(int, int)


//函数指针数组
int (*p[2])(int, int);  //p[0]-->f1(int, int), p[1]-->f2(int ,int)
int f1(int, int);
int f2(int, int);

//指针数组和数组指针
int*p[4];			   //p是一个指针数组，每一个指向一个int型的
int (*q)[4]   		   //q是一个指针，指向int[4]的数组
    
void(*)(void)  //--表示一个返回值为void，没有参数的函数指针
(void(*)(void))//--表示类型转换
//【将func这个函数强转成返回值为void，没有参数的函数】
```

- 参数中有函数指针变量，可以传入函数名，如下：

  ```c
  void (*timer_t)(void *arg, int channel);  // 函数指针定义
  
  void func(void *arg, int channel){}   // 具体函数
  
  void timer_irq(timer_t func){} 		  // 定义函数指针作为参数
  
  void rtimer(){
      timer_irq(func);				 // 调用的时候传入具体函数名称
  }
  ```

- void型指针，可以指向任何类型；如果将`void*`类型指针赋给其他类型指针，则需要强制类型转换且`void*`指针不可解引用（取内容）如下：

  ```c
  int cmp_node_val(void *a, void *b){
      if(*(int*)a == *(int *)b){
  		return 1;
      }
      return 0;
  }
  typedef void (*ptr_t)(void *a, void *b);
  
  void main(void){
      int a = 10;
      void *arg = &a;
      ptr_t p = cmp_node_val;
      int ret = p(&node->val, arg);
      
  }
  
  ```

- 用函数指针实现、维护 函数形式一样，但是功能不同的函数

  ```c
  void isr_svc(syscall_t index, void **p){
    if(index == ...){
    }
     *p = syscall_get_handler(index);
  }
  
  static void *__syscall_get_handler(syscall_t index){
      void *func_ptr = NULL;
  	isr_svc(index, &func_ptr); //实现了将 *p-> *func_ptr
      return func_ptr;
  }
  ```

  

- 

---

- 按位与的结果作为判断条件时：只要不为 0 都作 true 处理

-  typedef: typedef oldName newName

   ```c
   //有typedef时，pSum是一个指针类型，他可以初始化实例
   typedef int (*pSum)(int a,int b);
   
   //无typedef时，pSum仅仅是一个指针
   int(*pSum)(int a,int b); 
   
   /*----------------typedef 的基本用法如下--------------------------*/
   typedef struct Count{
      int x;
      int y;
      int c;
   }Num;    //用Num代替Count
   
   typedef struct Count{
      struct Count *p_Count;  
   }*pC; 	// <=> struct Count *pCount;
   	    // typedef pCount pC;
   
   
   /*----------------------复习一下struct用法------------------------*/
   //定义用法：
   struct strong {
       int age;
       int num;
   };
   
   struct strong st = { .age = 11, .num = 462 }; //也可以省略 .age 和 .num
   struct strong *p_st = &st;
   p_st->age = 123;
   
   //直接指向分配的空间
   struct strong *p_st2 = (struct strong*)malloc(sizeof(struct strong));
   p_st2->num = 1212;
   p_st2->age = 21;
   
   /*------------------------常用方式--------------------------*/
   struct strong {
       int age;
       int num;
   }st, *p_st; //<=>
   
   struct strong st;
   struct strong *p_st;
   
   /*typedef*/
   typedef struct strong{
     int age;
     int num;
   }st, *p_st; //<=>
   
   struct strong s;
   typedef s st;
   //指针
   struct strong* p;
   typedef p p_st;
   
   ```

---

- 数组命名方式：

  ```c
  int arr[3] = {
    [2] = 3;   
  };
  ```

- hbird-sdk中注册函数使用的代码：

  ```c
  int irq_num[3]; 		//定义函数数组
  typedef void (*IRQ_HANDLER)(int , int);  //定义函数指针
  
  void func_isr(int a, int b){        //中断处理函数
    printf("yep!\n");
  }
  
  //先将数组中的数据绑定到每个函数首地址, 即num[0]中保存了地址
  void bind_ID_to_FUNC(unsigned long func){
  	irq_num[0] = func;   
  }
  
  //使用函数指针，调用ISR
  void register_interrupt(){
      //irq_handler指向 num[0] 地址出
  	IRQ_HANDLER irq_handler = (IRQ_HANDLER)(irq_num[0]); 
      irq_handler(1, 1);
  }
  
  
  void main(){
    bind_ID_to_FUNC((unsigned long)func_isr);
    register_interrupt();
  }
  ```

- 项目中所有的中断处理函数写成了函数指针形式，出发中断时调用irq_handle；在这个里面调用非特殊的 isr，如下：

  ```c
  typedef void (*irq_handler_t)(void *arg);
  struct ihander  {
      irq_handler_t func;
      void *arg;
  };
  static struct ihandler handler[6];
  
  void register_int_handler(unsigned num, irq_handler_t func, void arg){
      handler[num].func = func;
      handler[num].arg = arg;
  }
  
  void timer_arch_init(){
  	register_int_hanler(irq_num, timer_irq, arg);
  }
  
  ```

-  

---

- 叶子函数是指在函数内部不分配栈空间，也不调用其它函数，也不存储非易失性寄存器，也不处理异常

- volatile: 精确地说就是，编译器在用到这个变量时必须每次都小心地重新读取这个变量的值，而不是使用保存在寄存器里的备份

-  一个内存地址对应一个字节，即0x80000000~0x80000004中存放了32bit的数据

-  C语言的精髓都在指针上面了，其中不同类型的指针变量区别是读取内存时候访问的数据量不同，例如char类型指针一次读取 1 Byte，而int型指针可以读取 4 Byte

-  ```c
   char *p = 0;   // p = NULL
   ```

---

typeof()关键字：gcc编译，使用隐式的函数指针调用函数，**优势**在于不用给函数指针初始化便可以直接调用。例如：sys_table[74] = get_sys_table，只能使用该方式调用函数

- 语法： `（typeof(func)*）table[n]()`，其中typeof获取的类型（func，包括参数类型）都会赋给后面的指针数组中的元素。
- **注意：**只有当func参数为void的时候，调用的函数是可以传入参数的，且传入的实参可以 < 形参

```c
void* sys_ta[3];
void** sys_get_talbe(void);

void sys_register() {
	printf("there is register\n");
}

void **sys_get_talbe() {
	printf("there is table\n");
	return sys_ta;
}

void sys_test(int a, int b, void *data) {
	printf("test\n");
}

void* sys_ta[3] = {
	sys_register,
	sys_get_talbe,
	sys_test,
};

void* get_sys(int index) {
	return sys_ta[index];
}

typedef void(*f_p) ();

//void ** res = ((typeof(函数名)*)(指针数组元素))() 
int main(void) {
	//两种方式：第一种，通过函数指针强转
	f_p p = (f_p)sys_ta[2];
	p();

	//第二种：通过typeof转换
	void **ret = ((typeof(sys_get_talbe)*) get_sys(1))();
	((typeof(sys_test)*) get_sys(0))();//也可以实现调用
	((typeof(sys_register)*) get_sys(2))(1); //传入的实参个数 < 形参个数

}
```

函数指针值与函数地址值相等。

对于函数名func来说，不管是*func还是func还是&func，编译器都认为他是函数指针，一般情况下你无法得知函数指针的地址。

而对于这个函数指针的地址（或者说是函数指针存放的位置我们无法知道），也只有

借助函数指针数组，你才能知道函数指针的地址。

- `(void (*)(void))func`: 将func这个函数强转成返回值为void，没有参数的函数

  ```c
  //其他代码同上（typeof）
  int main(void) {
  
  	void (*func)(void);   //定义一个函数指针
  	void** t_c = &sys_ta;	//用 t_c 指向 sys_ta 的地址
  	func = (void (*)(void))(*(t_c+2)); //func 指向 sys_ta中的元素
  	func();
  }
  ```


---

- 在C语言中如果要对绝对地址进行数据操作可以使用：

  ```c
  /* 将0x10000000地址的值修改为1234 */
  *(int *)0x10000000 = 1234; //先将 0x10000000 强转成整型指针变量，然后赋值
  
  ```

- 如果要让程序跳转到指定绝对地址去执行，可以通过将绝对地址强转为函数指针的方法：

  ```C
  //程序的跳转是通过寻找函数名(函数指针)指向的地址来完成的，
  //因此可以使用如下代码来实现让程序跳转到0x100000000处执行
  
  *((void (*)())0x100000000)(); //()-->执行  void (*)() 无参数 无返回值 函数指针
  通过typedef更加直观：
  typedef (void (*)()) func;  //返回值为void 参数为空的函数指针
  *((func)0x100000000)();
  
  //数据类型是 void (*)()
  ```

- load_talbe_move_to_target: 其中的 LOAD_TABLE_MAGIC 是使用的数据标志，规定而已，下面使用的 dst[-2]，dst[-1]是因为table中规定了 start-->end-->magic，只有magic是人为定义，用于查找到此位置

---

指针：包含三个内容，分别是 自身地址，存储内容，解引用内容

- **空指针不能当做参数进行传递**

- 当参数为指针类型时，本质上跟普通变量一样：都是先生成一个临时变量，将传参赋给临时变量；所以要改变指针的指向，需要用二级指针

- 指针p指向0x100地址处

  ```c
  int *p = (int *) 0x100;
  ```

- 二级指针：简单理解就是嵌套，`int **a = &b, int *b = &c`

```c
//自身地址：是系统分配的
//存储内容：是初始化时候设置的
//解引用内容：是 存储内容 指向区域 中的内容

int a = 10;
int b = 100;
int* q;
//int **p, 在函数内依然会重新 malloc 一个指针，p 的内容是传进来指针的地址
void func(int** p) {
	printf("------------------func-----------------------\n");
	printf("p = %x, &p = %x, *p = %x, **p= %d\n\n", p, &p, *p, **p);
	*p = &b;
	printf("p = %x, &p = %x, *p = %x, **p = %d\n\n", p, &p, *p, **p);
	printf("------------------end-------------------------\n");
}

int main(void) {


	printf("&a = 0x%x, &b = 0x%x, &q = 0x%x\n\n", &a, &b, &q);
	q = &a;
	printf("&a = %x, &b = %x, &q = %x, q = %x, *q = %d,\n", &a, &b, &q, q, *q);
	func(&q);
	printf("&a = %x, &b = %x, &q = %x, q = %x, *q = %d,\n", &a, &b, &q, q, *q);

	return 0;
}
```



---

- %x的意思是以十六进制显示
- %3x：以十六进制显示，且3位对齐，不够用空格补齐

- %0x数字x跟%数字x的意思差不多，区别在于不够长度补0

---

- 裸函数是编译器自动忽略入口与返回，如要添加返回：`asm volatile("ret")`

- 对于寄存器赋值操作，可以统一使用：`0x0 |= status` 或者 0x1··1 &= status，这样可以只修改 status 的值即可

  原因如下：

  ```c
  reg |= status;
  //假如原来是 0，置 1 -> reg = 0 | 1 = 1 	--> 没有问题
  //原来是 1， 置 1 -> reg = 1 | 1 = 1 	 --> 没有问题
  //但如果原来是 1，置 0 -> reg = 1 | 0 = 1  -->不对
  ```


---

- sprintf（）是把格式化数据输出成（存储到）字符串————将其他格式转化成字符串

  sscanf（）是从字符串中读取格式化的数据————将字符串转化成其他格式

  ```c
  struct S
  {
  	int age;
  	float f;
  	char arr[10];
  };
  
  int main()
  {
  	struct S s = { 10, 5.236f, "abcdefg" };
  	char buf[100] = { 0 };
  	//将格式化的数据转换成字符串存储到buf中
  	sprintf(buf, "%d %f %s\n", s.age, s.f, s.arr);
  	//输出buf
  	printf("%s\n", buf); //buf = 10 5.2236000 abcdefg
  	return 0;
  }
  /*---------------------------------------------------------------------*/
  int main()
  {
  	struct S s = { 10, 5.236f, "abcdefg" };
  	struct S temp = { 0 };
  	char buf[100] = { 0 };
  	//将格式化的数据转换成字符串存储到buf中
  	sprintf(buf, "%d %f %s\n", s.age, s.f, s.arr);
  	//输出buf
  	//printf("%s\n", buf);
   
  	//从buf中读取格式化的数据到temp中
  	sscanf(buf, "%d %f %s\n", &(temp.age), &(temp.f), temp.arr);
      //输出temp
  	printf("%d %f %s\n", temp.age, temp.f, temp.arr);
  	return 0;
  }
  ```

- snprintf(char *，int size, , format, int size): 将其转化为 str
- `%s`相当于连续读取内存，指针的类型决定了按多少字节读取，如int是按 4 字节读取(4 地址)，而 char 类型按 1 字节读取(1 地址)

注意：**返回值** 不包括字符串结尾的空字符 \0，字符串结尾都是有 \0 的

对于 %1，也就是 buff，每次调用 buff，都是从初始化地址传入的，必要时候 buff 需要 +=

```c
#include <stdio.h>

int main(void){
    char buff[128];
    int n = snprintf((char*)buff, 128, "DATA%d", 100);
    printf("n = %d\n", n);		 // n = 7
    printf("buff = %s\n", buff);  //输出 DATA100
    n = snprintf((char*)buff, 128, "DATAsafew", 100);
    printf("n = %d\n", n);		 // n = 9
    printf("buff = %s\n", buff);  //输出DATAsafew，与 100 无关 

    return 0;
}

//怎么即时获得 buff 中的内容
	char *p_info = maclloc(64);
	char *p; 		// p 是用做指针偏移 
	char *k, *v;	// 因为 p 先加上了偏移量，此时获取是为 0
	k = p;
	p += snprintf(p, 128, "%s","hello"); //  p = p + n(snprintf 返回的值)
	printf（"k = %c\n", *k);  	// k = h
	printf（"k = %s\n", k);  	// k = hello，这里不能写成 *k，*k = %c = 'h'
	v = p;
	p += snprintf(p, 128, "%d", 123);
	printf("v = %d\n", *v);  	// v = 49
	printf("v = %s\n", v);  	// v = 123
```

---

- malloc与calloc在内存分配时，前者分配**一整块**，后者分配**n块**，并且后者在分配时会将内存置为0，前者不会，内存里可能是垃圾数据
- _probe = 某个模块进行初始化
- static 定义的全局变量，但是没有给初始化却调用了——传入地址，则该调用就是初始化

---

static 用法：

- 主要用于区分全局变量，限定的是变量，对于（函数）指针指向该变量，或者直接指向 static 的地址，都是可以用的
- 静态全局变量在声明它的整个文件都是可见的，而在文件之外是不可见的
- 即便是 extern 外部声明也不可以
- 其它文件中可以定义相同名字的变量，不会发生冲突
- 未初始化的静态全局变量会自动初始化为 0

代码设计：

- 各个部分的功能使用接口函数，在具体的模块中去 **实现/跳过**，如下：

```c
//cmd_verify
void cmd_verify(const char* arg, void *data, unsigned size){
 //···
	if(sb_auth_image(ID, data, data) >= 0){
	}
}

//RV 要跳过
int sb_auth_image(int ID, void *data, unsigned size){
    return 0;
}
//其他不跳过
int sb_auth_image(int ID, void *data, unsigned size){
    return auth_load_memory_image(ID, data, size);
}

```

---

- 

*alloca*是向栈申请内存,因此无需释放。

*malloc* 分配的内存是位于 堆中 的,并且没有 初始化内存的内容 ,因此基本上malloc之后,调用函数memset来初始化这部分的内存空间,需要用 Free 方式释放空间.

*calloc*则将初始化malloc这部分的内存,设置为0. 

*realloc*则对malloc申请的内存进行大小的调整.

*free*将malloc申请的内存最终需要通过该函数进行释放. 

*sbrk*则是增加数据段的大小 

---

- char *s = "": 表示空字符串，长度为0，但是一个字符串，有地址空间
- char *t = NULL: 表示一个空指针，无地址空间

---

static inline: 用于维护代码的健壮性。对于一个函数是否内联，我们只有建议权，决定权在编译器，所以当编译器否决了内联的建议，那么该函数将以普通函数的形式出现在应用场景，不使用 static 会报错

---

stack: 使用完成后，不用将指针置为 NULL，指针会自动回收。有的地方置为 NULL，是为了 if(p != NULL) 的类似情况

---

list：

```c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>


#define LIST_CONCAT2(s1, s2) s1##s2
#define LIST_CONCAT(s1, s2) LIST_CONCAT2(s1, s2)

#define LIST(name) \
         static void *LIST_CONCAT(name,_list) = NULL; \
         static list_t name = (list_t)&LIST_CONCAT(name,_list)


typedef void ** list_t;

struct list {
  struct list *next;
};
/*---------------------------------------------------------------------------*/
void *
list_tail(const list_t list)
{
  struct list *l;

  if(*list == NULL) {
    return NULL;
  }

  for(l = *list; l->next != NULL; l = l->next);

  return l;
}
/*---------------------------------------------------------------------------*/
void
list_add(list_t list, void *item)
{
  struct list *l;

  /* Make sure not to add the same element twice */
//   list_remove(list, item);

  ((struct list *)item)->next = NULL;

  l = list_tail(list);

  if(l == NULL) {
    *list = item;
  } else {
    l->next = item;
  }
}

/*---------------------------------------------------------------------------*/
typedef struct bsx_t{
  struct bsx_t *next;
  int num;
}bsx;
/*---------------------------------------------------------------------------*/
void dump_list(list_t list){
  bsx *b = *list;

  if(!list){
    return;
  }
  printf("head->");
  for(; b != NULL; b = b->next){
    printf("%d->", b->num);
  }
  printf("end\n");

}
/*---------------------------------------------------------------------------*/

int main()
{

    LIST(strong);
    bsx b1 = {NULL, 1};
    bsx b2 = {NULL, 2};
    bsx b3 = {NULL, 3};
    bsx b4 = {NULL, 4};
    bsx b5 = {NULL, 5};

    list_add(strong, &b1);
    list_add(strong, &b2);
    list_add(strong, &b3);
    list_add(strong, &b4);
    list_add(strong, &b5);

    dump_list(strong);

    return 0;
}


```

---

# macro

---

- `#pragma`指令的作用是：用于指定计算机或操作系统特定的编译器功能

- `#undef MACRO`: 取消前面的宏定义符号

- **#define**：为了减少代码开销，而函数会有保存现场，恢复现场的开销，用于一些常用且重复量高的功能

  - #用于传入宏参数，一般是字符串形式，例 #define example(s) #s, s->"s"
  - 用##把两个宏参数贴合在一起  a##e##b-->aeb
  - #@x，其实就是给x加上单引号，结果返回是一个const char
  - define a=1; 后面可以接 ';'，是合法的
  - 
  - **注意**：##在宏定义中使用时，它后面的参数如果也是一个宏，那么是**不会展开**的，此时需要在使用中转宏(**默认**)

  | 宏              | 功能                                                         |
  | --------------- | ------------------------------------------------------------ |
  | ’#‘             | 字符串化, #a-> "a"                                           |
  | ‘##’            | 字符连接的功能, a##b  ab                                     |
  | '#@x'           | 返回一个 const char, 'x'                                     |
  | "__VA_ARGS__’   | 这个可变参数的宏是新的C99规范中新增的, 和变参函数中的`...`一致 |
  | ‘##__VA_ARGS__’ | `宏前面加上##的作用在于，当可变参数的个数为0时，这里的##起到把前面多余的","去掉的作用,否则会编译出错` |

- do...while(0): 仅仅是为了让 #define 定义的代码一起执行，无作他用

```c
#define DOSOMETHING(){\

               func1();\

               func2();}
...
if(a>0)
    DOSOMETHING();
else
   ...
···
  
// 展开后就是这个样子:
if(a>0)
{
    func1();

    func2();
};
else
    ...
// 编不过的
// 于是写成
#define DOSOMETHING() \

        do{ \

          func1();\

          func2();\

        }while(0)\
    
```

```
//几种 宏的用法

#define PT_WAIT_UNTIL(pt, condition)	        \
  do {						\
    LC_SET((pt)->lc);				\
    if(!(condition)) {				\
      return PT_WAITING;			\
    }						\
  } while(0)


#define PT_END(pt) LC_END((pt)->lc); PT_YIELD_FLAG = 0; \
                   PT_INIT(pt); return PT_ENDED; }

#define PT_BEGIN(pt) { char PT_YIELD_FLAG = 1; if (PT_YIELD_FLAG) {;} LC_RESUME((pt)->lc)

# define atomic_cas(ptr, cmp, swp) ({ \
  long flags = disable_irqsave(); \
  typeof(*(ptr)) res = *(volatile typeof(*(ptr)) *)(ptr); \
  if (res == (cmp)) *(volatile typeof(ptr))(ptr) = (swp); \
  enable_irqrestore(flags); \
  res; })


#define __RV_CSR_READ_CLEAR(csr, val)                           \
    ({                                                          \
        register rv_csr_t __v = (rv_csr_t)(val);                \
        __ASM volatile("csrrc %0, " STRINGIFY(csr) ", %1"       \
                     : "=r"(__v)                                \
                     : "rK"(__v)                                \
                     : "memory");                               \
        __v;                                                    \
    })
```



