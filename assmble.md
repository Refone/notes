# 汇编

[How to Use Inline Assembly Language in C Code](https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html#Using-Assembly-Language-with-C)

[Linux 中 x86 的内联汇编](https://www.ibm.com/developerworks/cn/linux/sdk/assemble/inline/index.html)

## ```__asm __volatile```的用法

标准格式：

```c
__asm__ __volatile__("[汇编指令]"
                     :"[输出]"
                     :"[输入]"
                     :"[寄存器约束]");
```

\*```__volatile__```告诉编译器不要对汇编代码进行优化，保证原封不动。
\*```__asm__```宏的本质是```asm```，不过多进行了一次判断，判断是否有GNU支持。

* **汇编指令**

    形如：
    
    ```x86asm
        "mov %1, %0\n\t"
        "add $1, %0"
    ```
    或

    ```x86asm
        "movl %1, %%eax;
         movl %%eax, %0;"
    ```
    在一对引号```""```中以分号```;```进行分隔，或是每条指令分别在一对```""```中，以```\n\t```进行格式化处理，```\t```使得生成的二进制代码更具可读性。

* **输入/输出**

    * "r" 是操作数的约束，它指定将变量 "a" 和 "b" 存储在寄存器中。请注意，输出操作数约束应该带有一个约束修饰符 "="，指定它是输出操作数。

    * 如果总共有 n 个操作数（包括输入和输出），那么第一个输入操作数的编号为 0，逐项递增，最后那个输出操作数的编号为 n -1。指令通过数 %0、%1 等来引用 C 表达式（指定为操作数）。编号先编输入再编输出。

    * **r的意义**
    
        ```c
            r   任何可用的通用寄存器
            a   %eax
            b   %ebx
            c   %ecx
            d   %edx
            S   %esi
            D   %edi
        ```
    * **内存操作数约束 (m)** 
    
        任何对它们执行的操作都将在内存位置中直接发生
        ```c
        asm("sidt %0\n" : :"m"(loc));
        ```
    
    * **匹配（数字）约束**
    
        一个变量既要充当输入操作数，也要充当输出操作数.
        这里的 "0" 指定第 0 个输出变量相同的约束。即，它指定 var 的输出实例只应该存储在 %eax 中。
        ```c
        asm ("incl %0" :"=a"(var):"0"(var));
        ```

    * **&约束修饰符**    
        
        要确保输入和输出分配到不同的寄存器中，可以指定 & 约束修饰符。
        ```c
           ("movl %1, %%eax;
            "movl %%eax, %0;"
            :"=&r"(y)
            :"r"(x)
            :"%eax");
        ```
* **寄存器约束**

    ```c
        int a=10, b;
        asm ("movl %1, %%eax; movl %%eax, %0;"
             :"=r"(b)       /* output */    
             :"r"(a)        /* input */
             :"%eax");      /* clobbered register */
    ```
    第三个冒号后的修饰寄存器 %eax 告诉将在 "asm" 中修改 GCC %eax 的值，这样 GCC 就不使用该寄存器存储任何其它的值。

## 例子

* 利用汇编交换变量

```c
void swap(int _a, int _b)
{
    a = _a;
    b = _b;

    asm("movl %1, %%eax;\
         movl %0, %1\
         movl %%eax, %0"
         :"=r"(a),"=r"(b)
         :"0"(a),"1"(b)
         :"%eax");

    pritf("a=%d b=%d\n",a,b);
}
```
