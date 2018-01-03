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

