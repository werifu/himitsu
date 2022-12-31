---
title: Pointer Analysis Context Sensitive
description:
toc: true
authors: []
tags: []
categories: []
series: []
date: 2022-12-19T13:43:36+08:00
lastmod: 2022-12-19T13:43:36+08:00
featuredVideo:
featuredImage:
draft: true
---

## Introduction

### 上下文不敏感指针分析的缺陷

精度丢失
 
![](problem-ctx-insensitive.png)  

![](ctx-sensitive-sample.png)

用上下文敏感提高了精度，对象示例能区分开

Ctx Insensitive 为何丢精度？

* 因为没有考虑方法的多次调用


实际上是多个 context 混在一起了（因为不同调用场景上下文不一样（比如实例里字段的值之类的））

策略：
### call-site sensitivity

对调用栈的抽象，把方法调用串起来

![](cloning-based-ci.png)  

对方法进行克隆成 context_num 份

### Context-Sensitive Heap

![](ctx-sensitive-heap.png)  

给堆抽象也加上下文，一个 call-site 可能会有多个上下文：因为一个 call-site 可能跑多次（控制流）

不给堆加入上下文的话：call-site 也会丢精度，$o_8.f$ 没法准确指向，如下图

![](ctx-sensitive-no-heap.png)  

对象堆也上下文敏感的话（ 3 和 4 是这个程序里的两个上下文），就可以准确区分相同 call-site 的不同实例

![](ctx-sensitive-with-heap.png)  

## Rules

![](domains.png)  

都增加了一个维度 C

为何 filed 不需要上下文？ field 挂靠在 object ，一个上下文的对象 field 是唯一的

![](rules.png)  

只是在上下文不敏感的规则上加上 context

$c^{'}$ 和 $c^{''}$ 可能是同一个上下文

![](rule-new.png)  

![](rule-assign.png)  

![](rule-store.png)  

![](rule-load.png)  

![](rule-call.png)  

call 的规则决定了上下文如何产生

![](select-example.png)  

$c^{t}$, t means target，表示被调用的函数所处的上下文


## Algorithms

![](graph-c.s.png)  

节点带有上下文

![](propagation-call.png)  

传播上下文， $c$ 的变量传进 $c^{'}$，

![](algo-ctx-sensi.png)  

注意 RM 和 CG 的元素是带有上下文的

![](addreachable.png)  

![](add-edge-propagate.png)  

AddEdge 跟 Propagate 跟上下文不敏感的完全一样，并不关心是否有上下文（因为上下文集成在元素当中了）

![](process-call.png)  

select 是一个产生新上下文的过程

typo: 右下倒数第四行 $AddReachable(c:m)$ 应该为 $AddReachable(c^{t}:m)$

## Variants
 
Variants 相当于策略，决定了 select 是什么样的（产生怎样的上下文）

![](variants.png)  

对于上下文不敏感，select 相当于一直返回空 ctx

### Call-Site Sensitivity

上下文由 call-site 组成，ctx = $[l^{'}, ... l^{'''}]$

$$ Select(c, l, c^{'}:o_i, m) = [l^{'}, ... l^{'''}, l ]$$
where 
$$ c = [l^{'}, ... l^{'''}] $$

也叫 call-string sensitivity

![](k-limit-ctx.png)  

限制上下文深度

![](1-call-site-example.png)  

### Object Sensitivity

![](obj-sensitivity.png)  

ctx 由对象 list 组成

![](1-call-site-vs-1-obj.png)  

1-call-site 准

![](1-obj-vs-1-call-site.png)  

1-obj 准

精度是无法直接比较的，需要 scope，不过对 OO 语言来说 obj 会比较好

### Type Sensitivity

![](type-sensitivity.png)  

是对 obj-sensitivity 的更高级抽象，同个类就合并起来（一个类多个实例提供同一个上下文）

精度严格小于等于 obj-sensitivity

![](type-vs-obj.png)  

通常来讲，几种 select 的比较：
* Precision: object > type > call-site
* Efficiency: type > object > call-site


