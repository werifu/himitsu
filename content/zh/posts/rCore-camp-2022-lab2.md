---
title: rCore Camp 2022 Lab2 记录
description: 实现 mmap 和 mumap，以及对页表机制的笔记
toc: true
authors:
  - werifu
tags:
  - OS
  - RiscV
categories: 
series:
  - rCoreCamp2022
date: 2022-07-18T00:29:54+08:00
lastmod: 2022-08-01T00:29:54+08:00
featuredVideo:
featuredImage:
draft: false
---
# rCoreCamp2022-lab2记录

## Lab2 本体

[lab地址](https://learningos.github.io/rust-based-os-comp2022/chapter4/7exercise.html)

### 1. 重写 sys_get_time

之前 sys_get_time 失效的原因是现在增加了虚拟内存的设定，而获取时间靠的是参数传入指针来赋值，这样 ts 不是真正的物理地址，因此会失效，因此需要新作的工作就是进行地址转换。

```rust
// YOUR JOB: 引入虚地址后重写 sys_get_time
// 核心代码
pub fn sys_get_time(ts: *mut TimeVal, _tz: usize) -> isize {
    let virt_addr = VirtAddr(ts as usize);
    if let Some(phys_addr) = virt2phys_addr(virt_addr) {
        let us = get_time_us();
        let kernel_ts = phys_addr.0 as *mut TimeVal;
        unsafe {
            *kernel_ts = TimeVal {
                sec: us / 1_000_000,
                usec: us % 1_000_000,
            };
        }
        0
    } else {
        -1
    }
}
```

地址转换就是查表而已。

```rust
// 虚拟地址转换成物理地址
fn virt2phys_addr(virt_addr: VirtAddr) -> Option<PhysAddr> {
    let offset = virt_addr.page_offset();
    let vpn = virt_addr.floor();
    let ppn = PageTable::from_token(current_user_token())
        .translate(vpn)
        .map(|entry| entry.ppn());
    if let Some(ppn) = ppn {
        Some(PhysAddr::combine(ppn, offset))
    } else {
        println!("virt2phys_addr() fail");
        None
    }
}
```


### 2. 重写 sys_task_info

这个跟 1 几乎一模一样，额外工作只有添加地址转换

```rust
// YOUR JOB: 引入虚地址后重写 sys_task_info
pub fn sys_task_info(ti: *mut TaskInfo) -> isize {
    if let Some(phys_addr) = virt2phys_addr(VirtAddr(ti as usize)) {
        get_task_info(phys_addr.0 as *mut TaskInfo);
        0
    } else {
        -1
    }
}
```

花了两天解决了一个坑， TaskStatus::Running 在 ci 的时候变成了 TaskStatus::Ready ，github 上有一个[ pr ](https://github.com/LearningOS/rust-based-os-comp2022/pull/77)解决了这个问题，是因为 ci 里的枚举值多了个 UnInit 值，导致解析错误（从整数解析成枚举）

### 3. 实现 mmap

```rust
fn sys_mmap(start: usize, len: usize, port: usize) -> isize
```

一开始直接使用 frame_allocate() 去分配物理页，但是后面一直过不了，瞄了一下别人的代码，发现根本就设计错了，在抽象层面上，代码框架已经实现了 MemorySet，负责管理一个应用所获得的所有内存，因此我们要做的是往 MemorySet 里新加一个 MapArea，表示一段连续的内存。

中间的校验、转换都不太难，要获取到当前任务的 memory_set，然后再往应用插页。

核心方法集成在 TASK_MANAGER 中，通过 insert_framed_area 来插入页面。

```rust
    fn task_map(&self, start: usize, len: usize, port: usize) -> isize {
        if start & (PAGE_SIZE - 1) != 0 {
            println!(
                "expect the start address to be aligned with a page, but get an invalid start: {:#x}",
                start
            );
            return -1;
        }
        // port最低三位[x w r]，其他位必须为0
        if port > 7usize || port == 0 {
            println!("invalid port: {:#b}", port);
            return -1;
        }

        let mut inner = self.inner.exclusive_access();
        let task_id = inner.current_task;
        let current_task = &mut inner.tasks[task_id];
        let memory_set = &mut current_task.memory_set;
  
        // check valid
        let start_vpn = VirtPageNum::from(VirtAddr(start));
        let end_vpn = VirtPageNum::from(VirtAddr(start + len).ceil());
        for vpn in start_vpn.0 .. end_vpn.0 {
            if let Some(pte) = memory_set.translate(VirtPageNum(vpn)) {
                if pte.is_valid() {
                    println!("vpn {} has been occupied!", vpn);
                    return -1;
                }
            }
        }

	// PTE_U 的语义是【用户能否访问该物理帧】
        let permission = MapPermission::from_bits((port as u8) << 1).unwrap() | MapPermission::U;
        memory_set.insert_framed_area(VirtAddr(start), VirtAddr(start+len), permission);
        0
    }
```


### 4. 实现 munmap

```rust
fn sys_munmap(start: usize, len: usize) -> isize
```

跟 3 差不多，注意要 unmap 的条件是页必须在使用，因此如果 pte 是不可用的，应该报错

```rust
    fn task_munmap(&self, start: usize, len: usize) -> isize {
        if start & (PAGE_SIZE - 1) != 0 {
            println!(
                "expect the start address to be aligned with a page, but get an invalid start: {:#x}",
                start
            );
            return -1;
        }
      
        let mut inner = self.inner.exclusive_access();
        let task_id = inner.current_task;
        let current_task = &mut inner.tasks[task_id];
        let memory_set = &mut current_task.memory_set;

        // check valid
        let start_vpn = VirtPageNum::from(VirtAddr(start));
        let end_vpn = VirtPageNum::from(VirtAddr(start + len).ceil());
        for vpn in start_vpn.0 .. end_vpn.0 {
            if let Some(pte) = memory_set.translate(VirtPageNum(vpn)) {
                if !pte.is_valid() {
                    println!("vpn {} is not valid before unmap", vpn);
                    return -1;
                }
            }
        }
      
        let vpn_range = VPNRange::new(start_vpn, end_vpn);
        memory_set.munmap(vpn_range);
        0
    }
```

## 杂项

### 问答题

[WIP]

1. 缺页缺页指的是进程访问页面时页面不在页表中或在页表中无效的现象，此时 MMU 将会返回一个中断， 告知 os 进程内存访问出了问题。os 选择填补页表并重新执行异常指令或者杀死进程。

   * 请问哪些异常可能是缺页导致的？
   * 发生缺页时，描述相关重要寄存器的值，上次实验描述过的可以简略。

   缺页有两个常见的原因，其一是 Lazy 策略，也就是直到内存页面被访问才实际进行页表操作。 比如，一个程序被执行时，进程的代码段理论上需要从磁盘加载到内存。但是 os 并不会马上这样做， 而是会保存 .text 段在磁盘的位置信息，在这些代码第一次被执行时才完成从磁盘的加载操作。

   * 这样做有哪些好处？

   其实，我们的 mmap 也可以采取 Lazy 策略，比如：一个用户进程先后申请了 10G 的内存空间， 然后用了其中 1M 就直接退出了。按照现在的做法，我们显然亏大了，进行了很多没有意义的页表操作。

   * 处理 10G 连续的内存页面，对应的 SV39 页表大致占用多少内存 (估算数量级即可)？
   * 请简单思考如何才能实现 Lazy 策略，缺页时又如何处理？描述合理即可，不需要考虑实现。

   缺页的另一个常见原因是 swap 策略，也就是内存页面可能被换到磁盘上了，导致对应页面失效。

   * 此时页面失效如何表现在页表项(PTE)上？

2. 双页表与单页表  
   为了防范侧信道攻击，我们的 os 使用了双页表。但是传统的设计一直是单页表的，也就是说， 用户线程和对应的内核线程共用同一张页表，只不过内核对应的地址只允许在内核态访问。 (备注：这里的单/双的说法仅为自创的通俗说法，并无这个名词概念，详情见 [KPTI](https://en.wikipedia.org/wiki/Kernel_page-table_isolation) )

   * 在单页表情况下，如何更换页表？
   * 单页表情况下，如何控制用户态无法访问内核页面？（tips:看看上一题最后一问）
   * 单页表有何优势？（回答合理即可）
   * 双页表实现下，何时需要更换页表？假设你写一个单页表操作系统，你会选择何时更换页表（回答合理即可）？

## 第四章-地址空间笔记

### SV39 多级页表

下图是我觉得最能讲清 SV39 多级页表机制的，对我的理解有很大帮助，来自 [2022春OS slides- lec5](https://learningos.github.io/os-lectures/lec5/p3-labs.html)

![sv39](https://s3.bmp.ovh/imgs/2022/08/01/f888893408efc350.png)
* 一个页表有512项，一项的size为usize，在RV64中就是8字节，因此一个页表刚好占一个物理页帧（4K），因此要找到某一个页表，只需要知道其PPN（offset为0即可）

* 蓝色框框代表一个虚拟地址的组成，其中只有低12+9+9+9位排上了用场，高位都是没用的

  * 其中三个9位的分别对应该地址分别在三级页表中的偏移，而前一级页表项中就存着当前这个页表的物理地址
* 一个页表项的组成：

  * 左边保留无用
  * 接着是PPN
  * 还有2位的保留
  * 最后是flags，标志着下一级的各种信息


## 问答作业

1. 请列举 SV39 页表页表项的组成，描述其中的标志位有何作用

   我认为这张图很好地表达了 SV39 多级页表的结构，虚拟地址分成 EXT（没用），在三级页表中的偏移（or 索引），以及偏移地址。  
   第一级页表的基址存在 satp 寄存器里，在切换任务的时候会切换 satp，实现了不同应用的地址隔离，因为不会访问到同一个页表。

   前两级页表分为 PPN 和 FLags 两部分，前者代表下一页表的基址（一个页表有 512 个 PTE ，一个 PTE 的长度为 usize，在 RV64 下为 64bit，因此一个页表刚好占一个物理页，4K ）。

   最终由三级页表的 PPN 跟 VA 的 Offset 组合成了物理地址。

   标志位如上图，言简意赅，需要注意的是 [10: 8] 被 RSW 占据，保留用而已，没有其他作用。

2. 缺页

   缺页指的是进程访问页面时页面不在页表中或在页表中无效的现象，此时 MMU 将会返回一个中断， 告知 os 进程内存访问出了问题。os 选择填补页表并重新执行异常指令或者杀死进程。

   * 请问哪些异常可能是缺页导致的？
   * 发生缺页时，描述相关重要寄存器的值，上次实验描述过的可以简略。

   缺页有两个常见的原因，其一是 Lazy 策略，也就是直到内存页面被访问才实际进行页表操作。 比如，一个程序被执行时，进程的代码段理论上需要从磁盘加载到内存。但是 os 并不会马上这样做， 而是会保存 .text 段在磁盘的位置信息，在这些代码第一次被执行时才完成从磁盘的加载操作。

   * 这样做有哪些好处？

   其实，我们的 mmap 也可以采取 Lazy 策略，比如：一个用户进程先后申请了 10G 的内存空间， 然后用了其中 1M 就直接退出了。按照现在的做法，我们显然亏大了，进行了很多没有意义的页表操作。

   * 处理 10G 连续的内存页面，对应的 SV39 页表大致占用多少内存 (估算数量级即可)？
   * 请简单思考如何才能实现 Lazy 策略，缺页时又如何处理？描述合理即可，不需要考虑实现。

   缺页的另一个常见原因是 swap 策略，也就是内存页面可能被换到磁盘上了，导致对应页面失效。

   * 此时页面失效如何表现在页表项(PTE)上？

3. 双页表与单页表  
   为了防范侧信道攻击，我们的 os 使用了双页表。但是传统的设计一直是单页表的，也就是说， 用户线程和对应的内核线程共用同一张页表，只不过内核对应的地址只允许在内核态访问。 (备注：这里的单/双的说法仅为自创的通俗说法，并无这个名词概念，详情见 [KPTI](https://en.wikipedia.org/wiki/Kernel_page-table_isolation) )

   * 在单页表情况下，如何更换页表？
   * 单页表情况下，如何控制用户态无法访问内核页面？（tips:看看上一题最后一问）
   * 单页表有何优势？（回答合理即可）
   * 双页表实现下，何时需要更换页表？假设你写一个单页表操作系统，你会选择何时更换页表（回答合理即可）？