---
title: rCore Camp 2022 Lab4 记录
description: 实现 linkat 、 unlinkat，以及对文件系统的笔记
toc: true
authors:
  - werifu
tags:
  - OS
  - RiscV
categories: 
series:
  - rCoreCamp2022
date: 2022-07-30T00:29:54+08:00
lastmod: 2022-08-01T00:29:54+08:00
featuredVideo:
featuredImage:
draft: false
---
# rCoreCamp2022-lab4记录

## Lab4 本体

[lab地址](https://learningos.github.io/rust-based-os-comp2022/chapter5/4exercise.html)

Lab4 的难度感觉比 Lab3 高了一档，因为之前在学校或者自己学 OS 的时候都基本都没学到文件系统，所以这一张属于是真的新学了。在理解上遇到了很大的障碍，代码也很难憋出来。

## 0. 迁移代码

唯一的难点是，这章使用文件系统代替了loader，因此加载用户程序的方式从 get_app_by_name 从loader里取改成了用文件系统的api 去打开文件

下面的代码片段摘自 sys_spawn，使用open_file 去获取 inode 进而创建新的 task

```rust
    if let Some(inode) = open_file(path.as_str(), OpenFlags::RDONLY) {
        let data = inode.read_all();
        let task = current_task().unwrap();
        let new_task = task.spawn(data.as_slice());
        let pid = new_task.pid.0;
        add_task(new_task);
        pid as isize
    }
```

## 1. fstat

因为觉得 fstat 是最好做的所以从它最开始了

获取 fstat 的核心是（ino**,** mode**,** nlink）三个数据，代表inode id，文件的模式（是文件还是目录），有几个引用，这部分实现在Inode 结构中，我们给 File trait 添加一个特征方法叫 fstat()，能够返回三个维度的数据

注意 Stdin 和 Stdout 也是特殊的文件，但是懒得实现就直接在里面 panic 了。

```rust
impl File {
	// ... read, write, readable, writable
	fn fstat(&self) -> (u64, StatMode, u32);
}
```

fstat 里最难的是得到 inode id，因为 Inode 中只提供了 get_inode_id_by_name，我们将该能力分配给 fs，由efs来帮忙实现从块信息中读取 inode 的id

```rust
// impl Inode ==> get_inode_id
fs.get_inode_id(self.block_id, self.block_offset)
```

```rust
impl EasyFileSystem {
    pub fn get_inode_id(&self, block_id: usize, block_offset: usize) -> usize {
        let inode_size = core::mem::size_of::<DiskInode>();
        let inodes_per_block = (BLOCK_SZ / inode_size) as usize;
        // 目标 inode 处在 inode 区第n个
        let nth_inode_block = block_id - self.inode_area_start_block as usize;
        // 目标 inode 所在区前有几个 inode， + 区里排第几个 inode
        return nth_inode_block * inodes_per_block + block_offset / inode_size;
    }
}
```

## 2. linkat

linkat 的功能实现在 Inode 里，代码需要模仿 Inode 中的 create 方法，实际上新建文件的过程也是创建一个硬连接

具体思路就是：

1. 新旧文件名的校验
2. 在 get_block_cache 里，通过从 fs 得到的 block 位置信息，新建一个文件
3. 更新目录表，插入一个新的DirEntry

```rust
    /// like `fn create`
    pub fn linkat(&self, old_name: &str, new_name: &str) -> isize {
        let mut fs = self.fs.lock();
        let old_inode_id = self.read_disk_inode(|disk_inode|
            self.find_inode_id(old_name, disk_inode)
        );
        // old_name should point to a valid file
        if old_inode_id == None {
            return -1;
        }
        // new_name should not point to an existing file
        if self.read_disk_inode(|disk_inode| self.find_inode_id(new_name, disk_inode)).is_some() {
            return -1;
        }
        let (block_id, block_offset) = fs.get_disk_inode_pos(old_inode_id.unwrap());
        get_block_cache(block_id as usize, Arc::clone(&self.block_device))
            .lock()
            .modify(block_offset, |new_inode: &mut DiskInode| {
                new_inode.initialize(DiskInodeType::File)
            });

        // update dir table
        self.modify_disk_inode(|root_inode| {
            // add a new dir entry
            let file_count = (root_inode.size as usize) / DIRENT_SZ;
            let new_size = (file_count + 1) * DIRENT_SZ;
            self.increase_size(new_size as u32, root_inode, &mut fs);

            // write into the new dir entry
            let dirent = DirEntry::new(new_name, old_inode_id.unwrap());
            root_inode.write_at(file_count * DIRENT_SZ, dirent.as_bytes(), &self.block_device);
        });
        0
    }
```


## 3. unlinkat

unlink 跟 link 还是比较像，思路是遍历根目录（使用modify_disk_inode），在 file_num 个文件里找到符合条件的文件，删除其目录项（将其赋为 DirEntry::empty()）。

能这么写是因为 rCore 里的文件系统只有一层，根目录下全是文件

```rust
    /// unlink
    pub fn unlink(&self, name: &str) -> isize {
        self.modify_disk_inode(|root_inode| {
            let file_num = (root_inode.size as usize) / DIRENT_SZ;
            // find the correct entry and modify it, else -1
            for i in 0..file_num {
                let mut dirent = DirEntry::empty();
                let readn = root_inode.read_at(
                    i * DIRENT_SZ,
                    dirent.as_bytes_mut(),
                    &self.block_device
                );
                // read size should == DIRENT_SZ
                if readn != DIRENT_SZ {
                    continue;
                }

                if dirent.name() == name {
                    let dirent = DirEntry::empty();
                    root_inode.write_at(i * DIRENT_SZ, dirent.as_bytes(), &self.block_device);
                    return 0;
                }
            }
            -1
        })
    }
```

## 杂项

### 简答题

1. 在我们的easy-fs中，root inode起着什么作用？如果root inode中的内容损坏了，会发生什么？

**Answer**: ROOT_INODE 代表根目录所对应的 Inode，也是整个文件系统（文件树）的起点，我们管理其他的文件都是在根目录下玩完成的，如果它坏了，那么整个文件系统就无法正常管理文件


## 文件系统笔记

### 一个磁盘文件系统的组织结构

这一章的 MVP slide 我觉得是下图，一个 fs 分成了下面五块

* Super Block：记录了后边几个分别占了多少块（磁盘的单位使用块来表示，类似内存的页帧）
* Inode Bitmap：记录 Inodes 块的使用情况，一个位可以表示一个 Inode 的使用与否，在分配 Inode 时起到重要作用
* Data Bitmap：跟上一块差不多，记录的是数据块的使用情况
* Inodes：是DiskInode（存储于磁盘上的文件管理单元）在内存里的形态，一个Inode代表管理了一个文件，通过访问 Inode 可以访问到数据，而其实这些数据就缓存在 Data Blocks 里，但是上层的抽象不需要感知到 DataBlock
* Data Blocks：真实的数据，通过 Inode 能够找到

![fs 视图](https://s3.bmp.ovh/imgs/2022/08/01/8b27861e801c27fd.png)

下图则代表了目录项，就是上图的右上表，实现从文件名到文件Inode的映射（当然如果是目录，则映射到其下的一堆目录项）

![目录项](https://s3.bmp.ovh/imgs/2022/08/01/bf32bbeb4f78156d.png)

## 感想

1. 文件系统对我来说过于陌生，所以在这花了非常多的时间，整整一周，到现在也不是彻底理解了这套系统，多亏了助教[xushanpu 的笔记](https://github.com/xushanpu123/xsp-daily-work/blob/master/%E6%9A%91%E6%9C%9Frcore%E5%AE%9E%E9%AA%8C%E7%AC%94%E8%AE%B0/chapter%206%EF%BC%9A%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F.md)，用自上而下的视角去理解代码结构感觉比 camp 文档里的更好理解。
2. 对文件系统的理解主要就是第一张图那个视图

   1. 左上是用户视角，文件系统就是一棵文件树
   2. 右上是 OS 视角，一个文件是一个 Inode，中间是文件名到 Inode idx 的映射
   3. 下面的结构是磁盘视角，一个fs分区就是这样的结构

      * 如果我们把一个磁盘分区，那么每个分区都是独立的 fs，因此都会有平行的那个结构
