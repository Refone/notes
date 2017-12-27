# Cuckoo Migration 工作进展

## Milestone

### Part 1：单机部分
 - [ ] 为guest设置两张EPT，并且能够用VMFUNC进行切换

  > 每次启动Guest都能自动分配两张EPT，并且在启动阶段自动在EPT-1上加载一个kernel module(K1)。

  > 在正常内核起一个kernel module，能利用实现好的VMFUNC接口访问到EPT-1上K1的static数据。

 - [ ] 实现内核态稳定的网络传输模块

  > 将K1改造为网络传输模块，并export网络传输接口，让其他kernel module能够正常调用。
  
  > 能够通过调用接口更改网络传输对象，定好一个特定的端口号，以后迁移模块就用这个端口号。

  > 在正常内核中启动一个kernel module，能够调用K1的网络传输接口。

 - [ ] 成功调用VE Handler

  > 【工具】实现GPA访问kernel module（GPAK），用于访问（读/写）特定GPA。

  > 在正常内核中利用GPAK，模拟访问一个不在当前EPT中的GPA，看是否能执行到EPT-1中的VE Handler代码中。

 - [ ] 利用VE做好内存访问记录（触发过VE得地址不会触发第二次）。
 
  > 设计一个bitmap，放在EPT-1里面，能够记录GPA的访问情况。

  > 为正常内核提供接口，可以重置这个bitmap。

  > 为正常内核提供接口，可以查看某一GPA是否已被访问过。

### Part 2：联机部分

 - [ ] 走通一个remote page fetching流程

  > 在source VM的正常内核里起一个kernel module（Ka），Ka在一个特殊地址addr上alloc一块数据。

  > 在target VM的正常内核里起一个kernel module（Kb），用Kb访问特殊地址addr，这时候肯定会触发VE，看看VE能否结合网络传输模块拿到source VM中addr处的数据。

### *Part 3：迁移部分

 - [ ] 【问题】在target host上如何起“空壳虚拟机”？上面有些什么，没有什么？
 - [ ] 【设想】可不可以基于进程迁移？在vcpu切换环境的时候判断某一个进程的所有数据是否已经存在于target上，如果没有，拷过来。这样甚至内存布局都可以不一样，只要保证所有进程都迁过来就行了。
 - [ ] 【研究】一个进程的所有信息，数据在内存哪里？cpu状态记在哪里？应该全在mm_struct里面记载，好好研究这个模块。
 - [ ] 【问题】如何冻结EPT-0？
   