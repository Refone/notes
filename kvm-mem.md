# KVM内存管理笔记

主机: Intel<sup>@</sup> Core<sup>TM</sup> i7-6700 CPU

系统环境：[linux-4.14.9](http://elixir.free-electrons.com/linux/v4.14.9/source)

本文讲的是基于EPT架构的内存管理（还有一套shadow page table的管理方式）

## memslots — 内存插槽

KVM中内存以“slot”的方式进行管理，一个slot就对应了一个GPA到HPA的映射
```c
// arch/x86/include/asm/kvm_host.h

#define KVM_USER_MEM_SLOTS 509
/* memory slots that are not exposed to userspace */
#define KVM_PRIVATE_MEM_SLOTS 3
#define KVM_MEM_SLOTS_NUM (KVM_USER_MEM_SLOTS + KVM_PRIVATE_MEM_SLOTS)
```

重要的slot相关数据结构↓
```c
// include/linux/kvm_host.h

struct kvm_memory_slot {
    gfn_t base_gfn;
    unsigned long npages;
    unsigned long *dirty_bitmap;
    struct kvm_arch_memory_slot arch;
    unsigned long userspace_addr;
    u32 flags;
    short id;
};

/*
 * Note:
 * memslots are not sorted by id anymore, please use id_to_memslot()
 * to get the memslot by its id.
 */
struct kvm_memslots {
    u64 generation;
    struct kvm_memory_slot memslots[KVM_MEM_SLOTS_NUM];
    /* The mapping table from slot id to the index in memslots[]. */
    short id_to_index[KVM_MEM_SLOTS_NUM];
    atomic_t lru_slot;
    int used_slots;
};

struct kvm {
    spinlock_t mmu_lock;
    struct mutex slots_lock;
    struct mm_struct *mm; /* userspace tied to this vm */
    struct kvm_memslots __rcu *memslots[KVM_ADDRESS_SPACE_NUM];
    struct kvm_vcpu *vcpus[KVM_MAX_VCPUS];

    ...
}
```
从```virt/kvm/kvm_main.c```一个vm的创建过程中我们可以看出，一个Guest的创建其实就是一个kvm结构的填充过程，而其中之一的步骤就是初始化kvm中的memslots。
最后再把创建好的```kvm```添加到```vm_list```链表中
```c
// virt/kvm/kvm_main.c

static struct kvm *kvm_create_vm(unsigned long type)
{
    int r, i;
    struct kvm *kvm = kvm_arch_alloc_vm();

    if (!kvm)
        return ERR_PTR(-ENOMEM);

    spin_lock_init(&kvm->mmu_lock);
    mmgrab(current->mm);
    kvm->mm = current->mm;
    kvm_eventfd_init(kvm);
    mutex_init(&kvm->lock);
    mutex_init(&kvm->irq_lock);
    mutex_init(&kvm->slots_lock);
    refcount_set(&kvm->users_count, 1);
    INIT_LIST_HEAD(&kvm->devices);
    
    ...

    // memslots的初始化过程
    r = -ENOMEM;
    for (i = 0; i < KVM_ADDRESS_SPACE_NUM; i++) {
        struct kvm_memslots *slots = kvm_alloc_memslots();
        if (!slots)
            goto out_err_no_srcu;
        /*
         * Generations must be different for each address space.
         * Init kvm generation close to the maximum to easily test the
         * code of handling generation number wrap-around.
         */
        slots->generation = i * 2 - 150;
        rcu_assign_pointer(kvm->memslots[i], slots);
    }

    ...

    spin_lock(&kvm_lock);
    list_add(&kvm->vm_list, &vm_list);  //创建好的kvm添加到vm_list中
    spin_unlock(&kvm_lock);

    ...

    return kvm;

    ...
}
```

## EPT VIOLATION
发生EPT VIOLATION的意思是GPA到HPA的映射没有在EPT MMU上，需要host来进行处理。

EPT VIOLATIOLN处理流程:
```
arch/x86/kvm/vmx.c: vmx_handle_exit()->
	                        [EXIT_REASON_EPT_VIOLATION] = handle_ept_violation;
                    handle_ept_violation()->
arch/x86/kvm/mmu.c: kvm_mmu_page_fault()->
                    tdp_page_fault()
```

```tdp_page_fault()```在初始化阶段被设置为了host的page fault函数
```c
// arch/x86/kvm/mmu.c

static void init_kvm_tdp_mmu(struct kvm_vcpu *vcpu)
{
    struct kvm_mmu *context = &vcpu->arch.mmu;

    ...
    context->page_fault = tdp_page_fault;
    ...
}

```
所以```tdp_page_fault()```即为EPT VIOLATION最终调用的处理函数。
```c
// arch/x86/kvm/mmu.c

static int tdp_page_fault(struct kvm_vcpu *vcpu, gva_t gpa, u32 error_code,
              bool prefault)
{
    ...
    // 最关键的下面两条语句
    
    // 试着从memslots中找到这个GPA到HPA的映射，如果找到了，那么返回0
    if (try_async_pf(vcpu, prefault, gfn, gpa, &pfn, write, &map_writable))
        return 0;

    // 如果没有在memslots中找到该映射，那么就说明这是一个新申请的页，我们需要在EPT中添加这个映射
    spin_lock(&vcpu->kvm->mmu_lock);
    ...
    r = __direct_map(vcpu, write, map_writable, level, gfn, pfn, prefault);
    spin_unlock(&vcpu->kvm->mmu_lock);
    ...
}
```

### 查找映射```try_async_pf```
尝试异步处理page fault
```c
// arch/x86/kvm/mmu.c

static bool try_async_pf(struct kvm_vcpu *vcpu, bool prefault, gfn_t gfn,
             gva_t gva, kvm_pfn_t *pfn, bool write, bool *writable)
{
    struct kvm_memory_slot *slot;
    bool async;
    
    // 这就是从vcpu找到对应domain对应memslots的函数
    slot = kvm_vcpu_gfn_to_memslot(vcpu, gfn);      
    ...

    // 在对应的memslots中寻找guest frame number到physical frame number的映射
    *pfn = __gfn_to_pfn_memslot(slot, gfn, false, &async, write, writable);
    ...
}
```
```__gfn_to_pfn_memslot```查找gfn到pfn的映射，其中包含了gfn->hva和hva->pfn两个过程
```c
kvm_pfn_t __gfn_to_pfn_memslot(struct kvm_memory_slot *slot, gfn_t gfn,
                   bool atomic, bool *async, bool write_fault,
                   bool *writable)
{
    unsigned long addr = __gfn_to_hva_many(slot, gfn, NULL, write_fault);

    ...

    return hva_to_pfn(addr, atomic, async, write_fault,
              writable);
}
EXPORT_SYMBOL_GPL(__gfn_to_pfn_memslot);
```
gfn->hva最终过程：
```c
static inline unsigned long
__gfn_to_hva_memslot(struct kvm_memory_slot *slot, gfn_t gfn)
{
    return slot->userspace_addr + (gfn - slot->base_gfn) * PAGE_SIZE;
}
```

hva->pfn过程：在```hva_to_pfn```中有
```hva_to_pfn_fast```以及```hva__to_pfn_slow```两个函数，
不是特别清楚他们的区别，
不过最终目的都是将从hva（addr变量）中找到的物理页放到pfn中
```c
// virt/kvm/kvm_main.c

static kvm_pfn_t hva_to_pfn(unsigned long addr, bool atomic, bool *async,
            bool write_fault, bool *writable)
{
    struct vm_area_struct *vma;
    kvm_pfn_t pfn = 0;
    int npages, r;

    /* we can do it either atomically or asynchronously, not both */
    BUG_ON(atomic && async);

    if (hva_to_pfn_fast(addr, atomic, async, write_fault, writable, &pfn))
        return pfn;

    if (atomic)
        return KVM_PFN_ERR_FAULT;

    npages = hva_to_pfn_slow(addr, async, write_fault, writable, &pfn);
    if (npages == 1)
        return pfn;
    
    ...
}
```
最后在```mm/memory.c```文件中，一堆以```vm_```或```_vm_```开头的函数负责处理vm的page walk
```c
// 插入vm页
static int __vm_insert_mixed(struct vm_area_struct *vma, unsigned long addr, pfn_t pfn, bool mkwrite);
// fetch vm页
struct page *_vm_normal_page(struct vm_area_struct *vma, unsigned long addr, pte_t pte, bool with_public_device);

// 下面这些事export出来的接口，全局可以调用
int vm_insert_mixed(struct vm_area_struct *vma, unsigned long addr, pfn_t pfn);
int vm_insert_mixed_mkwrite(struct vm_area_struct *vma, unsigned long addr,  pfn_t pfn);
int vm_insert_page(struct vm_area_struct *vma, unsigned long addr, struct page *page);
int vm_insert_pfn(struct vm_area_struct *vma, unsigned long addr, unsigned long pfn);
int vm_insert_pfn_prot(struct vm_area_struct *vma, unsigned long addr, unsigned long pfn, pgprot_t pgprot);
int vm_iomap_memory(struct vm_area_struct *vma, phys_addr_t start, unsigned long len);

struct page *vm_normal_page_pmd(struct vm_area_struct *vma, unsigned long addr, pmd_t pmd);
```

## 创建EPT映射

如果没有找到该EPT的映射，那么说明是要创建新的EPT映射，由如下函数来完成
```c
// arch/x86/kvm/mmu.c


```

## 其他



## 参考资料

[KVM之内存虚拟化(KVM MMU Virtualization)](http://royluo.org/2016/03/13/kvm-mmu-virtualization/)

[KVM硬件辅助虚拟化之 EPT(Extended Page Table)](http://royluo.org/2014/06/18/KVM-EPT/)

[一个日本小哥的KVM总结](http://rkx1209.hatenablog.com/?page=1457533661)

[KVM地址翻译流程及EPT页表的建立过程](http://blog.csdn.net/lux_veritas/article/details/9284635)

[史上最详细的kvm_mmu_page结构和用法解析](http://ytliu.info/blog/2014/11/24/shi-shang-zui-xiang-xi-de-kvm-mmu-pagejie-gou-he-yong-fa-jie-xi/)

[kvm: virtual x86 mmu setup](http://blog.stgolabs.net/2012/03/kvm-virtual-x86-mmu-setup.html)
