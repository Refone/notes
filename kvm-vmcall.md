# KVM中VMCALL的调用

主机: Intel<sup>@</sup> Core<sup>TM</sup> i7-6700 CPU

系统环境：[linux-4.14.9](http://elixir.free-electrons.com/linux/v4.14.9/source)

## HOST中VMCALL响应控制流

1. 在```arch/x86/kvm/vmx.c```中被截获

```c
static int (*const kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
    ...
    [EXIT_REASON_VMCALL]                  = handle_vmcall,
    ...
};
```

2. 之后在```arch/x86/kvm/vmx.c```中由```handle_vmcall```处理

```c
static int handle_vmcall(struct kvm_vcpu *vcpu)
{
    return kvm_emulate_hypercall(vcpu);
}
```

3. 最后调用```arch/x86/kvm/x86.c```中的```kvm_emulate_hypercall```

```c
int kvm_emulate_hypercall(struct kvm_vcpu *vcpu)
{
    ...
    
    //Guest中VMCALL依次把参数放入寄存器，
    //在这里依次从寄存器中读入参数
    nr = kvm_register_read(vcpu, VCPU_REGS_RAX);
    a0 = kvm_register_read(vcpu, VCPU_REGS_RBX);
    a1 = kvm_register_read(vcpu, VCPU_REGS_RCX);
    a2 = kvm_register_read(vcpu, VCPU_REGS_RDX);
    a3 = kvm_register_read(vcpu, VCPU_REGS_RSI);
    
    ...

    switch (nr) {   //nr即为系统调用号
    case KVM_HC_VAPIC_POLL_IRQ:
        ret = 0;
        break;
    case KVM_HC_KICK_CPU:
        kvm_pv_kick_cpu_op(vcpu->kvm, a0, a1);
        ret = 0;
        break;
        
    ...

    default:
        ret = -KVM_ENOSYS;
        break;
    }
    //按照规定，将返回值写入EAX寄存器
    kvm_register_write(vcpu, VCPU_REGS_RAX, ret);
    ...
    return r;
}
EXPORT_SYMBOL_GPL(kvm_emulate_hypercall);
```
## Guest中对VMCALL的调用

1.引用头文件

```c
#include <linux/kvm_para.h>
```

2.调用```arch/x86/include/asm/kvm_para.h```中的函数接口，它们的本质都是调用```vmcall```汇编指令，只是形参个数不同
```c
static inline long kvm_hypercall0(unsigned int nr);
static inline long kvm_hypercall1(unsigned int nr, unsigned long p1);
static inline long kvm_hypercall2(unsigned int nr, unsigned long p1, unsigned long p2);
static inline long kvm_hypercall3(unsigned int nr, unsigned long p1, unsigned long p2, unsigned long p3)
static inline long kvm_hypercall4(unsigned int nr, unsigned long p1, unsigned long p2, unsigned long p3, unsigned long p4)
```

## 自定义KVM Hypercall

1.定义自己的调用号，不要与其他的函数调用号重复。代码位于```include/uapi/linux/kvm_para.h```
```c
...
#define KVM_HC_VAPIC_POLL_IRQ		1
#define KVM_HC_MMU_OP			2
#define KVM_HC_FEATURES			3
#define KVM_HC_PPC_MAP_MAGIC_PAGE	4
#define KVM_HC_KICK_CPU			5
#define KVM_HC_MIPS_GET_CLOCK_FREQ	6
#define KVM_HC_MIPS_EXIT_VM		7
#define KVM_HC_MIPS_CONSOLE_OUTPUT	8
#define KVM_HC_CLOCK_PAIRING		9

#define KVM_HC_MY_VMCALL    20  //自己的函数调用号
...
```

2.编写自己的handler程序。代码位于```arch/x86/kvm/x86.c```
```c
int kvm_emulate_hypercall(struct kvm_vcpu *vcpu)
{
    ...
    switch (nr) {
    //自己的处理函数
    case KVM_HC_MY_VMCALL:
        do_something();
        break
	case KVM_HC_VAPIC_POLL_IRQ:
		ret = 0;
		break;
	case KVM_HC_KICK_CPU:
		kvm_pv_kick_cpu_op(vcpu->kvm, a0, a1);
		ret = 0;
		break;
    ...
    }
    ...
}
```
> 接下来不重启更新KVM

3.编译KVM，进入linux代码主目录下
```bash
make
sudo modules_install SUBDIRS=arch/x86/kvm
```

4.卸载原来的KVM
* 首先看看有哪些模块与KVM相关
```bash
sudo lsmod | grep kvm
```
得到
```bash
[0] % sudo lsmod | grep kvm
kvm_intel             233472  10            #有10个进程在依赖kvm_intel,需要关闭所有虚拟机
kvm                   688128  1 kvm_intel   #kvm_intel在依赖kvm
irqbypass              16384  1 kvm         #kvm在依赖irqbypass
```
* 卸载响应模块，注意顺序，卸载的永远是没有其他模块会依赖的模块
```bash
sudo rmmod kvm_intel
sudo rmmod kvm
```

5. 安装新KVM
* 安装kvm模块，装的顺序依然要注意。
```bash
sudo insmod arch/x86/kvm/kvm.ko
sudo insmod arch/x86/kvm/kvm-intel.ko
```
或者
```bash
sudo modprobe kvm_intel
```

## 参考资料

[KVM VMCALL](http://royluo.org/2014/07/27/hyper-call/)

