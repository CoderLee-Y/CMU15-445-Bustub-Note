---
layout: post
title:  "CMU 15-441 - Lab 2"
date:   2022-03-27 20:35:49 +0800
categories: database
tags: ["database"]
---

# About

- 保证你的环境正确，特别是做代码风格检查的，如果出现权限问题，你需要`chmod`对应文件；让他帮忙训练代码风格很有好处。

# Lab 2

## 零长度数组

数组名是指针，这是cpp的第一课。而一个结构体中的零长度数组其实就是指这个结构体的尾部。那么有什么意义呢？这是因为这个是`hash table bucket page`. 本身会被分配到一个页里面。而这个数组的元数据是被精心组织好的bitset：

```c++
  //  For more on BUCKET_ARRAY_SIZE see storage/page/hash_table_page_defs.h
  char occupied_[(BUCKET_ARRAY_SIZE - 1) / 8 + 1];
  // 0 if tombstone/brand new (never occupied), 1 otherwise.
  char readable_[(BUCKET_ARRAY_SIZE - 1) / 8 + 1];
```

那么array_指针其实就是data指针。而大小呢？别担心，他的宏已经帮你算好了：

```c++
(4 * PAGE_SIZE / (4 * sizeof(MappingType) + 1))
```

而Mapping type就是KV对。这是一个现代c++如何实现哈希表的有用范例。

这个大小是怎么算出来的呢？我们仔细分析一下。假设Mapping Type的size是4+4 = 8. 那么桶大小是(4096 * 4 / (8 * 4 + 1)) = 496(向下取整)。然后元数据的大小是/8向上取整。也即62. 62 * 2 = 124. 那么加起来就是124 + 496 * 8 = 4092. 可以看出，已经是最优解了。

Part 1弄懂这个之后，剩下的部分是位运算和阅读注释，可以参考内置函数`__builtin_popcount`使用他的接口完成。需要提醒的有：

- Readable是有效位，occupied是占据位。Insert可以直接插在墓碑(tombstone)上
- GetValue返回是否找到，Insert只有桶满或者重复才返回False, Remove只有成功才返回true

## 可扩展哈希表

<img src="https://s2.loli.net/2022/03/14/u5bmYeahtIPUQoM.png" alt="image-20220314150325681" style="zoom:50%;" />

参考上面的图写。其实和我们最近MOSPI 2021 Chcore lab的buddy system有点像。对于简单不造成结构变化的，直接写即可，注意两点：Fetch多少页就要Unpin多少页，拿了几把锁（一般而言，要拿两把锁，元数据一把，页一把）就要放掉几把锁。这个Lab真正的难度在于Merge和Insert部分。这部分有一些让人疑惑的点：

- `SplitInsert`的接口是什么？这是给出的一个可选的接口。我疑惑于为什么需要？如果Insert失败了，那么判断IsFull不就好了吗？根本原因是因为是否Spilt对于锁的要求是不一致的；我们可以利用这个函数更加细粒度的控制读写锁。
- Split部分确实让人疑惑，包括应该对哪些页提高Local depth, 应该让哪些页指向新的页。应该自己画图尝试一下，标注各个变量的来龙去脉。这里有一个有用的[参考](https://blog.csdn.net/twentyonepilots/article/details/120868216#:~:text=size_t%20common_bits%20%3D%20kti,dpg%2D%3EIncrLocalDepth(i)%3B%0A%20%20%7D)，我认为代码风格很不错。（当然不要去找完整实现！这不是好的习惯）。此外，可以感受一下我认为比较明显的“簇”的概念，比如LD是2，GD是4的情况下，0001, 0011, 0101, 0111, 1001, 1011, 1101, 1111是同簇的。分裂和合并的时候，他们往往被一起处理。
- 写完Split, merge的一部分（挑选“同源”桶entry）也是类似的逻辑。主要注意的是：你如果合并任何一对页，那么他们所有同源的页也需要全部重定向。

虽然看上去比较简单，但其中细碎的Bug非常多，特别是如果造成了内存泄漏，Gradescope根本不会帮你判分！你需要自己写不同的样例，但是好在只需要在他给的基础上进行一定的修改即可。然后注意在本地运行好内存泄漏检测再上传。(Tip: 样例规模应该从非常大到非常小。有些测试居然就3个Page, 这是最小的可能的Buffer pool数量，因为Split的时候目录和两个Split页就要用到。那么，**在任何时候，保证你足够及时的Unpin, 特别是这里出现了很多递归调用！**)

最后，你需要实现页粒度的读写锁，注意锁是不可重入的即可。Good Luck!

## Debug心得

- 这个Lab将教会你自己写用例，他的用例是插入5个（这没有意义…），但是当你能正确插入5W个之后，你将能得到90+分。此外，设计一些大数据用例测试你的每个函数，最后，优化性能即可。

- `__builtin_popcount`可以用，但是有符号和无符号的转换一定要弄清楚。有符号转有符号会扩充1，所以应该有符号转等大小无符号再转无符号保证这个函数的正确性。但其实，这个二分的优化没啥意义…因为数字不是很大，自行取舍。

- 维护一些函数的不变量，并且到处ASSERT这些不变量。比如Image之间的Local Depth保持一致等。

- GradeScope是可能出现问题的。我花费了10 hours来解决Lab 2无法被评测的问题, `Test Failed: False is not true : build errored`. 但是当我确认了Lab 1的正确性，Formatter的正确性以及Clang的正确性，且内存检查已经通过之后，我仍然无法运行Lab 2评测。这时在Github的Issue区看到了Pavlo的回复，差点崩溃…

  <img src="C:/Users/11796/AppData/Roaming/Typora/typora-user-images/image-20220325104117598.png" alt="image-20220325104117598" style="zoom:50%;" />

  没错，一周前就有人报告了Lab 2有问题，但是他表示需要一点时间，就拖到了今天…当GradeScope修复之后，评测就通过了。最终的速度也排在了LeaderBoard前3页。

<img src="https://s2.loli.net/2022/03/28/wSCMEyUYbGoFADp.png" alt="image-20220328223605433" style="zoom:50%;" />
