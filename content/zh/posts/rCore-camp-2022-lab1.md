---
title: rCore Camp 2022 Lab1 记录
description: lab报告及简答题
toc: true
authors:
  - werifu
tags:
  - OS
  - RiscV
categories: 
series:
  - rCoreCamp2022
date: 2022-07-14T01:20:44+08:00
lastmod: 2022-07-14T01:20:44+08:00
featuredVideo:
featuredImage:
draft: false
---
# rCoreCamp2022-lab1记录

## Lab1本体

[lab地址](https://learningos.github.io/rust-based-os-comp2022/chapter3/5exercise.html)

实现以下函数的功能

```rust
fn sys_task_info(ti: *mut TaskInfo) -> isize
struct TaskInfo {
    status: TaskStatus,
    syscall_times: [u32; MAX_SYSCALL_NUM],
    time: usize
}
```

### 分解需求

1. 让 TASK_MANAGER 有获取 task 状态的能力
2. 让 TASK_MANAGER 有更新 task 状态的能力

### 实现

其实就是对 TASK_MANAGER 管理的每个 task 维护一个 TaskInfo 的对象，每次 syscall 的时候进行记录。

因此我们给 Task 的对象加入一个新的 inner 字段，维护该任务开始时间和系统调用情况。

![Inner](https://s3.bmp.ovh/imgs/2022/07/14/c6c23c1967cef82c.png)

之后给 TASK_MANAGER 加 set 和 get 的能力，都挺好懂的

```rust
// src/task/mod.rs
    fn set_syscall_times(&self, syscall_id: usize) {
        let mut inner = self.inner.exclusive_access();
        let current_id = inner.current_task;
        inner.tasks[current_id].task_info_inner.syscall_times[syscall_id] += 1;
    }

    fn get_current_task_info(&self, ti: *mut TaskInfo) {
        let inner = self.inner.exclusive_access();
        let current_id = inner.current_task;
        let TaskInfoInner {syscall_times, start_time} = inner.tasks[current_id].task_info_inner;

        unsafe {
            *ti = TaskInfo {
                status: TaskStatus::Running,
                syscall_times,
                time: get_time_ms() - start_time,
            };
        }
    }
```

然后对外提供接口

```rust
pub fn record_syscall(syscall_id: usize) {
    TASK_MANAGER.set_syscall_times(syscall_id);
}

pub fn get_task_info(ti: *mut TaskInfo) {
    TASK_MANAGER.get_current_task_info(ti);
}
```

每次调用syscall的时候记录一下就行了。

![syscall前](https://s3.bmp.ovh/imgs/2022/07/14/de517ab611e9546e.png)

## 简答作业

1. 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。 请同学们可以自行测试这些内容 (运行 [Rust 三个 bad 测例 (ch2b_bad_*.rs)](https://github.com/LearningOS/rust-based-os-comp2022/tree/main/user/src/bin) ， 注意在编译时至少需要指定 `LOG=ERROR` 才能观察到内核的报错信息) ， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。

2. 深入理解 [trap.S](https://github.com/LearningOS/rust-based-os-comp2022/blob/main/os3-ref/src/trap/trap.S) 中两个函数` __alltraps`和`__restore`的作用，并回答如下问题:

   1. L40：刚进入 `__restore` 时，`a0` 代表了什么值。请指出 `__restore` 的两种使用情景。

      在L37有一行`mv a0, sp`的指令，这是`addi a0, sp, 0`的语法糖，其实就是mov，在`__alltraps`里是给参数赋值，但是在之后a0并没有再次出现过，结合trap_handler的参数和返回值，可以认为又被返回回去了，所以a0直到`__restore`时都是taskContext。

      `__restore`封装在`goto_restore`里，我们ctrl f全局找一下可以看到唯一引用的地方：TASK_MANAGER的初始化

      这个地方把每个应用的上下文（包括32个寄存器、sepc、sstatus值）存起来返回然后从supervisor返回到user

      ![restore应用](https://s3.bmp.ovh/imgs/2022/07/14/6a3f8e6f097a1915.png)

      其实所有从 supervisor => user 都可以使用`__restore`，跟`__alltraps`是反过来的，对偶使用，另一种场景想不出来……。

   2. L46-L51：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。

      ```mipsasm
      ld t0, 32*8(sp)
      ld t1, 33*8(sp)
      ld t2, 2*8(sp)
      csrw sstatus, t0
      csrw sepc, t1
      csrw sscratch, t2
      ```

      * t0给了 sstatus，是 trap 发生前 cpu 特权级的信息（如S/U）

      * t1给了 sepc，是 trap 发生前执行的最后一条指令的地址
      * t2给了 sscratch，是原来 user stack 的 sp

      这仨都是用来恢复状态的（在进入supervisor的时候把原来信息存在这里面了）

   3. L53-L59：为何跳过了 `x2` 和 `x4`？

      问题2和3都在[特权级交换](https://learningos.github.io/rust-based-os-comp2022/chapter2/4trap-handling.html)这章里讲了，x2是 sp(stack pointer) ，x4 是 tp(thread pointer)

      > 跳过x2：用户栈的栈指针保存在 sscratch 中，必须通过 `<span class="pre">csrr</span>` 指令读到通用寄存器中后才能使用，因此我们先考虑保存其它通用寄存器，腾出空间。
      >
      > 跳过x4：非特殊情况不需要用到x4（存疑？）

   4. L63：该指令之后 sp 和 sscratch 中的值分别有什么意义？

      ```mipsasm
      csrrw sp, sscratch, sp
      ```

      `csrrw r1, r2, r3`的意思是把 r2 写进 r1 ，把 r3 写进 r2 ，在上面就是交换 sp 和 sscratch 的意思。  
      sp 原来是用户栈，sscratch 原来是内核栈，交换后 sp 指向 kernel stack， sscratch 指向 user stack

   5. `__restore`: 中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？  
      我猜是`csrw sstatus, t0`，sstatus代表用户状态

   6. L13：该指令之后，`sp` 和 `sscratch` 中的值分别有什么意义？  
      同4，反过来（怀疑上面L63是笔误

   7. 从 U 态进入 S 态是哪一条指令发生的？  
      看不出来……因为现场没有改变 sstatus 的。