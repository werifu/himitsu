---
title: RCore Camp 2022 Lab3
description:
toc: true
authors:
  - werifu
tags:
  - OS
  - RiscV
categories: 
series:
  - rCoreCamp2022
date: 2022-07-23T00:29:54+08:00
lastmod: 2022-08-01T00:29:54+08:00
featuredVideo:
featuredImage:
draft: false
---
# rCoreCamp2022-lab3记录

## Lab3 本体

[lab 地址](https://learningos.github.io/rust-based-os-comp2022/chapter5/4exercise.html)

### 0. 迁移通过以前的测试

1. get_time： 原本我给 TCB 加了一个 inner，这部分在第五章里官方加了这个结构，因此需要把 syscall_times 和 start_time 给迁移进去
2. get_task_info： 和 get_time 同理，不过需要注意的是代码更改了逻辑，原本由 TASK_MANAGER 负责管理的任务调度和任务执行部分，分成了 manager 和 processor 的工作，前者负责管理任务调度，后者负责操控当前任务 + 当前任务的切换，这二者都迁移进了 Processor 里
3. mmap & munmap：没什么变化，就是直接搬运

### 1. 实现 spawn

```rust
fn sys_spawn(path: *const u8) -> isize
```

平时我们会使用 fork() + exec() 来实现创建一个新的进程，但为什么要先复制状态机再重置状态机（状态机的理论见绿导师的南大 OS 课）？当然可以直接创建一个新的状态！

man page 提供了 [spawn 相关的说明](https://man7.org/linux/man-pages/man3/posix_spawn.3.html)

实现不太难，就是在 fork 和 exec 上偷偷而已，具体代码就不放了

```rust
impl TaskControlBlock {
    /// Spawn a new process without fork + exec
    pub fn spawn(self: &Arc<TaskControlBlock>, elf_data: &[u8]) -> Arc<TaskControlBlock> {
	// 1. 解析 elf 文件，得到 memory_set, user_sp, entry_point, 从 memory_set 里算出物理 trap_cx_ppn
	// 2. 新建 TCB，数据是新的（类似 exec 里的逻辑）
	// 3. 将新的 TCB 挂到当前 TCB 的 children 里
	// 4. 修改新的 TCB 的 trap_cx 的值
    }
}
```

### 2. 实现 stride

stride 意思是步伐，计算方式是 `stride = BIG_STRIDE / priority`，每次执行一个任务要 `pass += stride`，然后在调度时选择 pass 最小的执行。因为 priority 当了分母，所以越高的优先级的 stride 越小，每次就越早被调度。

比如 priority = 5 的进程和 priority = 10 的进程，每次优先级为5的增加的 stride 是 10 的两倍，所以在相同时间里，次数大约会是优先级 10 的一半。

下面的实现参考了助教 xushanpu123 的笔记， 维护一个 pass 单调递增的队列，每次从头取就行了。  
另外一种做法是队列不一定单调，但是每次取时靠遍历来找最小值

```rust
impl TaskManager {
    pub fn add(&mut self, task: Arc<TaskControlBlock>) {
        // insert the new task into a proper position
        let inner = task.inner_exclusive_access();
        let pass = inner.pass;
        // let prio = inner.priority;
        // drop the ownership of inner
        drop(inner);

        let len = self.ready_queue.len();
        for idx in 0..len {
            let queue_task = self.ready_queue.get_mut(idx).unwrap();
            let pass1 = queue_task.inner_exclusive_access().pass;
            // keep the queue head owns the smallest pass
            if pass < pass1 {
                // println!("new task priority: {}, pass: {}, inserted before idx {}", prio, pass, idx);
                self.ready_queue.insert(idx, task);
                return
            }
        }
        self.ready_queue.push_back(task);
    }
}
```

### 踩坑

1. 会有 `already borrowed: BorrowMutError` 的 panic 报错，是在测试结束后调用最后一个 exit 时发生的

   * 定位：在函数前面加 `#[trace_caller]` 就能显示文件+行数
   * 原因：在测试的时候，initproc 会被替换成各章的 ch_usertest ,所以是不会像普通的运行那样进入 initproc 然后运行 shell 的， usertest 退出的时候所有权会被借走，但是后面又借回来了导致错误（其实就是在 exit_current_and_run_next_task 里，将要退出的和下一个任务是同一个，而我们需要同时可变借用这两个）
   * 解决方案：在拿到 current 后判断 pid，如果是 0（表示初进程）就调用 sbi 里的 shutdown() 直接关机

2. stride test 过不了，调度不符合公平性

   * 没找到原因，调度的过程应该是没错的，打印出来的统计数据没有错
   * 解决：给 ci-user 里的 ch5_stridex 加了println!，使得执行速度大幅降低，然后居然就正常了…

<br />

>  2022.07.26更新：  
>  已经找到原因，是分时程序切换那里做的设置定时时钟中断出问题了，是sbi的故障，夏令营仓库在commit 70ae28ab2280f3e57d14b7a631e7508fe5b4bbaf 后就修复了，在sbi.rs中调用 ecall 前执行 **"li x16, 0"**


## 杂项

### 问答题

stride 算法深入

stride 算法原理非常简单，但是有一个比较大的问题。例如两个 stride = 10 的进程，使用 8bit 无符号整形储存 pass， p1.pass = 255, p2.pass = 250，在 p2 执行一个时间片后，理论上下一次应该 p1 执行。

* 实际情况是轮到 p1 执行吗？为什么？

**Answer**: 并不，因为整型溢出了导致p2.pass更小


我们之前要求进程优先级 >= 2 其实就是为了解决这个问题。可以证明， **在不考虑溢出的情况下** , 在进程优先级全部 >= 2 的情况下，如果严格按照算法执行，那么 PASS_MAX – PASS_MIN <= BigStride / 2。

* 为什么？尝试简单说明（不要求严格证明）。

**Answer**: （没搞懂）

* 已知以上结论， **考虑溢出的情况下** ，可以为 pass 设计特别的比较器，让 BinaryHeap<Pass> 的 pop 方法能返回真正最小的 Pass。补全下列代码中的 `partial_cmp` 函数，假设两个 Pass 永远不会相等。

```rust
use core::cmp::Ordering;

struct Pass(u64);

impl PartialOrdforPass{
	fn partial_cmp(&self, other: &Self)-> Option<Ordering>{
		// 口胡的代码
	}
}

impl PartialEqforPass{
	fn eq(&self, other: &Self)-> bool {
		false
	}
}
```

* TIPS: 使用 8 bits 存储 pass, BigStride = 255, 则: (125 < 255) == false, (129 < 255) == true


## 第五章-进程管理笔记

[WIP]

## 感想

1. 感觉这章的 lab 任务不太难，spawn 其实就是把 fork 和 exec 缝合一下，不难实现
2. stride 算法的原理也很简单，但是实现了最基本的公平性调度，对我来说感觉还是挺新奇的
3. 因为 sbi 的 bug 而导致卡了几天，很难受，幸好微信群有人解答，再次感受到了有人陪你一起写 lab 的重要性，一个人捣鼓的话遇到这种情况很可能要放弃了。感谢群友