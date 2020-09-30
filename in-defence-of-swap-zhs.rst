【译】替 swap 辩护：常见的误解
====================================================================

:slug: in-defence-of-swap
:translation_id: in-defence-of-swap
:lang: zhs
:date: 2020-09-30 13:45
:tags: swap, mm, memory management, translate, swappiness
:issueid: 97

这篇翻译自 Chris Down 的博文
`In defence of swap: common misconceptions <https://chrisdown.name/2018/01/02/in-defence-of-swap.html>`_
。 `原文的协议 <https://github.com/cdown/chrisdown.name/blob/master/LICENSE>`_
是 `CC BY-SA 4.0 <https://creativecommons.org/licenses/by-sa/4.0/>`_
，本文翻译同样也使用 `CC BY-SA 4.0`_ 。其中加入了一些我自己的理解作为旁注，所有译注都在侧边栏中。

翻译这篇文章是因为经常看到朋友们（包括有经验的程序员和 Linux 管理员）对 swap 和 swappiness
有诸多误解，而这篇文章正好澄清了这些误解，也讲清楚了 Linux 中这两者的本质。值得一提的是本文讨论的
swap 针对 Linux 内核，在别的系统包括 macOS/WinNT 或者 Unix 系统中的交换文件可能有不同一样的行为，
需要不同的调优方式。比如在 `FreeBSD handbook <https://www.freebsd.org/doc/handbook/bsdinstall-partitioning.html#configtuning-initial>`_
中明确建议了 swap 分区通常应该是两倍物理内存大小，这一点建议对 FreeBSD 系内核的内存管理可能非常合理，
而不一定适合 Linux 内核，FreeBSD 和 Linux 有不同的内存管理方式尤其是 swap 和 page cache 和
buffer cache 的处理方式有诸多不同。

经常有朋友看到系统卡顿之后看系统内存使用状况观察到大量 swap 占用，于是觉得卡顿是来源于 swap
。就像文中所述，相关不蕴含因果，产生内存颠簸之后的确会造成大量 swap 占用，也会造成系统卡顿，
但是 swap 不是导致卡顿的原因，关掉 swap 或者调低 swappiness 并不能阻止卡顿，只会将 swap
造成的 I/O 转化为加载文件缓存造成的 I/O 。

以下是原文翻译：

.. contents:: 目录

------------

这篇文章也有 `日文 <https://chrisdown.name/ja/2018/01/02/in-defence-of-swap.html>`_
和 `俄文 <https://softdroid.net/v-zashchitu-svopa-rasprostranennye-zabluzhdeniya>`_
翻译。

.. translate-collapse::

   tl;dr:

   #. Having swap is a reasonably important part of a well functioning system.
      Without it, sane memory management becomes harder to achieve.
   #. Swap is not generally about getting emergency memory, it's about making
      memory reclamation egalitarian and efficient. In fact, using it as
      "emergency memory" is generally actively harmful.
   #. Disabling swap does not prevent disk I/O from becoming a problem
      under memory contention, it simply shifts the disk I/O thrashing from
      anonymous pages to file pages. Not only may this be less efficient,
      as we have a smaller pool of pages to select from for reclaim, but it
      may also contribute to getting into this high contention state in
      the first place.
   #. The swapper on kernels before 4.0 has a lot of pitfalls,
      and has contributed to a lot of people's negative perceptions about
      swap due to its overeagerness to swap out pages. On kernels >4.0,
      the situation is significantly better.
   #. On SSDs, swapping out anonymous pages and reclaiming file pages are
      essentially equivalent in terms of performance/latency.
      On older spinning disks, swap reads are slower due to random reads,
      so a lower :code:`vm.swappiness` setting makes sense there
      (read on for more about :code:`vm.swappiness`).
   #. Disabling swap doesn't prevent pathological behaviour at near-OOM,
      although it's true that having swap may prolong it. Whether the
      system global OOM killer is invoked with or without swap, or was invoked
      sooner or later, the result is the same: you are left with a system in an
      unpredictable state. Having no swap doesn't avoid this.
   #. You can achieve better swap behaviour under memory pressure and prevent
      thrashing using :code:`memory.low` and friends in cgroup v2.


太长不看：

#. 对维持系统的正常功能而言，有 swap 是相对挺重要的一部分。没有它的话会更难做到合理的内存管理。
#. swap 的目的通常并不是用作紧急内存，它的目的在于让内存回收能更平等和高效。
   事实上把它当作「紧急内存」来用的想法通常是有害的。
#. 禁用 swap 在内存压力下并不能避免磁盘I/O造成的性能问题，这么做只是让磁盘I/O颠簸的范围从
   匿名页面转化到文件页面。这不仅更低效，因为系统能回收的页面的选择范围更有限了，
   而且这种做法还可能是加重了内存压力的原因之一。
#. 内核 4.0 版本之前的交换进程（swapper）有一些问题，导致很多人对 swap 有负面印象，
   因为它太急于（overeagerness）把页面交换出去。在 4.0 之后的内核上这种情况已经改善了很多。
#. 在 SSD 上，交换出匿名页面的开销和回收文件页面的开销基本上在性能/延迟方面没有区别。
   在老式的磁盘上，读取交换文件因为属于随机访问读取所以会更慢，于是设置较低的 :code:`vm.swappiness`
   可能比较合理（继续读下面关于 :code:`vm.swappiness` 的描述）。
#. 禁用 swap 并不能避免在接近 OOM 状态下最终表现出的症状，尽管的确有 swap
   的情况下这种症状持续的时间可能会延长。在系统调用 OOM 杀手的时候无论有没有启用 swap
   ，或者更早/更晚开始调用 OOM 杀手，结果都是一样的：整个系统留在了一种不可预知的状态下。
   有 swap 也不能避免这一点。
#. 可以用 cgroup v2 的 :code:`memory.low` 相关机制来改善内存压力下 swap 的行为并且
   避免发生颠簸。

------------


.. translate-collapse::

   As part of my work improving kernel memory management and cgroup v2,
   I've been talking to a lot of engineers about attitudes towards memory
   management, especially around application behaviour under pressure and
   operating system heuristics used under the hood for memory management.

我的工作的一部分是改善内核中内存管理和 cgroup v2 相关，所以我和很多工程师讨论过看待内存管理的态度，
尤其是在压力下应用程序的行为和操作系统在底层内存管理中用的基于经验的启发式决策逻辑。


.. translate-collapse::

   A repeated topic in these discussions has been swap. 
   Swap is a hotly contested and poorly understood topic, 
   even by those who have been working with Linux for many years. 
   Many see it as useless or actively harmful: a relic of a time where
   memory was scarce, and disks were a necessary evil to provide much-needed
   space for paging. This is a statement that I still see being batted
   around with relative frequency in recent years, and I've had many
   discussions with colleagues, friends, and industry peers to help them
   understand why swap is still a useful concept on modern computers with
   significantly more physical memory available than in the past.

在这种讨论中经常重复的话题是交换区（swap）。交换区的话题是非常有争议而且很少被理解的话题，甚至包括那些在
Linux 上工作过多年的人也是如此。很多人觉得它没什么用甚至是有害的：它是历史遗迹，从内存紧缺而
磁盘读写是必要之恶的时代遗留到现在，为计算机提供在当年很必要的页面交换功能作为内存空间。
最近几年我还经常能以一定频度看到这种论调，然后我和很多同事、朋友、业界同行们讨论过很多次，
帮他们理解为什么在现代计算机系统中交换区仍是有用的概念，即便现在的电脑中物理内存已经远多于过去。

.. translate-collapse::

   There's also a lot of misunderstanding about the purpose of swap –
   many people just see it as a kind of "slow extra memory" for use in emergencies,
   but don't understand how it can contribute during normal load to the healthy
   operation of an operating system as a whole.

围绕交换区的目的还有很多误解——很多人觉得它只是某种为了应对紧急情况的「慢速额外内存」，
但是没能理解在整个操作系统健康运作的时候它也能改善普通负载的性能。

.. translate-collapse::

   Many of us have heard most of the usual tropes about memory:
   " `Linux uses too much memory <https://www.linuxatemyram.com/>`_ ",
   " `swap should be double your physical memory size <https://superuser.com/a/111510>`_
   ", and the like. While these are either trivial to dispel,
   or discussion around them has become more nuanced in recent years,
   the myth of "useless" swap is much more grounded in heuristics and
   arcana rather than something that can be explained by simple analogy,
   and requires somewhat more understanding of memory management to reason about.

我们很多人也听说过描述内存时所用的常见说法： 「 `Linux 用了太多内存 <https://www.linuxatemyram.com/>`_
」，「 `swap 应该设为物理内存的两倍大小 <https://superuser.com/a/111510>`_ 」，或者类似的说法。
虽然这些误解要么很容易化解，或者关于他们的讨论在最近几年已经逐渐变得琐碎，但是关于「无用」交换区
的传言有更深的经验传承的根基，而不是一两个类比就能解释清楚的，并且要探讨这个先得对内存管理有
一些基础认知。

.. translate-collapse::

   This post is mostly aimed at those who administrate Linux systems and
   are interested in hearing the counterpoints to running with
   undersized/no swap or running with vm.swappiness set to 0.

本文主要目标是针对那些管理 Linux 系统并且有兴趣理解「让系统运行于低/无交换区状态」或者「把
:code:`vm.swappiness` 设为 0 」这些做法的反论。

背景
----------------------------------------

.. translate-collapse::

   It's hard to talk about why having swap and swapping out pages are good
   things in normal operation without a shared understanding of some of
   the basic underlying mechanisms at play in Linux memory management,
   so let's make sure we're on the same page.

如果没有基本理解 Linux 内存管理的底层机制是如何运作的，就很难讨论为什么需要交换区以及交换出页面
对正常运行的系统为什么是件好事，所以我们先确保大家有讨论的基础。

内存的类型
++++++++++++++++++++++++++++++++++++++++++++++++


.. translate-collapse::

   There are many different types of memory in Linux, and each type has its
   own properties. Understanding the nuances of these is key to understanding
   why swap is important.

Linux 中内存分为好几种类型，每种都有各自的属性。想理解为什么交换区很重要的关键一点在于理解这些的细微区别。

.. translate-collapse::

   For example, there are **pages ("blocks" of memory, typically 4k)**
   responsible for holding the code for each process being run on your computer.
   There are also pages responsible for caching data and metadata related to
   files accessed by those programs in order to speed up future access.
   These are part of the **page cache** , and I will refer to them as file memory.

比如说，有种 **页面（「整块」的内存，通常 4K）** 是用来存放电脑里每个程序运行时各自的代码的。
也有页面用来保存这些程序所需要读取的文件数据和元数据的缓存，以便加速随后的文件读写。
这些内存页面构成 **页面缓存（page cache）**，后文中我称他们为文件内存。

.. translate-collapse::

   There are also pages which are responsible for the memory allocations
   made inside that code, for example, when new memory that has been allocated
   with :code:`malloc` is written to, or when using :code:`mmap`'s
   :code:`MAP_ANONYMOUS` flag. These are "anonymous" pages –
   so called because they are not backed by anything –
   and I will refer to them as anon memory.

还有一些页面是在代码执行过程中做的内存分配得到的，比如说，当代码调用 :code:`malloc`
能分配到新内存区，或者使用 :code:`mmap` 的 :code:`MAP_ANONYMOUS` 标志分配的内存。
这些是「匿名(anonymous)」页面——之所以这么称呼它们是因为他们没有任何东西作后备——
后文中我称他们为匿名内存。


.. translate-collapse::

   There are other types of memory too –
   shared memory, slab memory, kernel stack memory, buffers, and the like –
   but anonymous memory and file memory are the most well known and
   easy to understand ones, so I will use these in my examples,
   although they apply equally to these types too.

还有其它类型的内存——共享内存、slab内存、内核栈内存、文件缓冲区（buffers），这种的——
但是匿名内存和文件内存是最知名也最好理解的，所以后面的例子里我会用这两个说明，
虽然后面的说明也同样适用于别的这些内存类型。

可回收/不可回收内存
++++++++++++++++++++++++++++++++++++++++++++++++

.. translate-collapse::

   One of the most fundamental questions when thinking about a particular type
   of memory is whether it is able to be reclaimed or not.
   "Reclaim" here means that the system can, without losing data,
   purge pages of that type from physical memory.

考虑某种内存的类型时，一个非常基础的问题是这种内存是否能被回收。
「回收（Reclaim）」在这里是指系统可以，在不丢失数据的前提下，从物理内存中释放这种内存的页面。


.. translate-collapse::

   For some page types, this is typically fairly trivial. For example,
   in the case of clean (unmodified) page cache memory,
   we're simply caching something that we have on disk for performance,
   so we can drop the page without having to do any special operations.

对一些内存类型而言，是否可回收通常可以直接判断。比如对于那些干净（未修改）的页面缓存内存，
我们只是为了性能在用它们缓存一些磁盘上现有的数据，所以我们可以直接扔掉这些页面，
不需要做什么特殊的操作。


.. translate-collapse::

   For some page types, this is possible, but not trivial. For example,
   in the case of dirty (modified) page cache memory, we can't just drop the page,
   because the disk doesn't have our modifications yet.
   As such we either need to deny reclamation or first get our changes back to
   disk before we can drop this memory.

对有些内存类型而言，回收是可能的，但是不是那么直接。比如对脏（修改过）的页面缓存内存，
我们不能直接扔掉这些页面，因为磁盘上还没有写入我们所做的修改。这种情况下，我们可以选择拒绝回收，
或者选择先等待我们的变更写入磁盘之后才能扔掉这些内存。

.. translate-collapse::

   For some page types, this is not possible. For example,
   in the case of the anonymous pages mentioned previously,
   they only exist in memory and in no other backing store,
   so they have to be kept there.

对还有些内存类型而言，是不能回收的。比如前面提到的匿名页面，它们只存在于内存中，没有任何后备存储，
所以它们必须留在内存里。

说到交换区的本质
----------------------------------------


.. translate-collapse::

   If you look for descriptions of the purpose of swap on Linux,
   you'll inevitably find many people talking about it as if it is merely
   an extension of the physical RAM for use in emergencies. For example,
   here is a random post I got as one of the top results from typing
   "what is swap" in Google:

      Swap is essentially emergency memory; a space set aside for times
      when your system temporarily needs more physical memory than you
      have available in RAM. It's considered "bad" in the sense that
      it's slow and inefficient, and if your system constantly needs
      to use swap then it obviously doesn't have enough memory. […]
      If you have enough RAM to handle all of your needs, and don't
      expect to ever max it out, then you should be perfectly safe
      running without a swap space.

如果你去搜 Linux 上交换区的目的的描述，肯定会找到很多人说交换区只是在紧急时用来扩展物理内存的机制。
比如下面这段是我在 google 中输入「什么是 swap」 从前排结果中随机找到的一篇：

   交换区本质上是紧急内存；是为了应对你的系统临时所需内存多余你现有物理内存时，专门分出一块额外空间。
   大家觉得交换区「不好」是因为它又慢又低效，并且如果你的系统一直需要使用交换区那说明它明显没有足够的内存。
   ［……］如果你有足够内存覆盖所有你需要的情况，而且你觉得肯定不会用满内存，那么完全可以不用交换区
   安全地运行系统。

.. translate-collapse::

   To be clear, I don't blame the poster of this comment at all for the content
   of their post – this is accepted as "common knowledge" by a lot of
   Linux sysadmins and is probably one of the most likely things that you will
   hear from one if you ask them to talk about swap. It is unfortunately also,
   however, a misunderstanding of the purpose and use of swap, especially on
   modern systems.

事先说明，我不想因为这些文章的内容责怪这些文章的作者——这些内容被很多 Linux 系统管理员认为是「常识」，
并且很可能你问他们什么是交换区的时候他们会给你这样的回答。但是也很不幸的是，
这种认识是使用交换区的目的的一种普遍误解，尤其在现代系统上。

.. translate-collapse::

   Above, I talked about reclamation for anonymous pages being "not possible",
   as anonymous pages by their nature have no backing store to fall back to
   when being purged from memory – as such, their reclamation would result in
   complete data loss for those pages. What if we could create such a
   store for these pages, though?


前文中我说过回收匿名页面的内存是「不可能的」，因为匿名内存的特点，把它们从内存中清除掉之后，
没有别的存储区域能作为他们的备份——因此，要回收它们会造成数据丢失。但是，如果我们为这种内存页面创建
一种后备存储呢？

.. translate-collapse::

   Well, this is precisely what swap is for. Swap is a storage area for these
   seemingly "unreclaimable" pages that allows us to page them out to
   a storage device on demand. This means that they can now be considered as
   equally eligible for reclaim as their more trivially reclaimable friends,
   like clean file pages, allowing more efficient use of available physical memory.

嗯，这正是交换区存在的意义。交换区是一块存储空间，用来让这些看起来「不可回收」的内存页面在需要的时候
可以交换到存储设备上。这意味着有了交换区之后，这些匿名页面也和别的那些可回收内存一样，
可以作为内存回收的候选，就像干净文件页面，从而允许更有效地使用物理内存。

.. translate-collapse::

   **Swap is primarily a mechanism for equality of reclamation,**
   **not for emergency "extra memory". Swap is not what makes your application**
   **slow – entering overall memory contention is what makes your application slow.**

**交换区主要是为了平等的回收机制，而不是为了紧急情况的「额外内存」。使用交换区不会让你的程序变慢——**
**进入内存竞争的状态才是让程序变慢的元凶。**

.. translate-collapse::

  So in what situations under this "equality of reclamation"
  scenario would we legitimately choose to reclaim anonymous pages?
  Here are, abstractly, some not uncommon scenarios:

那么在这种「平等的可回收机遇」的情况下，让我们选择回收匿名页面的行为在何种场景中更合理呢？
抽象地说，比如在下述不算罕见的场景中：

.. translate-collapse::

   #. During initialisation, a long-running program may allocate and
      use many pages. These pages may also be used as part of shutdown/cleanup,
      but are not needed once the program is "started" (in an
      application-specific sense). This is fairly common for daemons which
      have significant dependencies to initialise.
   #. During the program's normal operation, we may allocate memory which is
      only used rarely. It may make more sense for overall system performance
      to require a **major fault** to page these in from disk on demand,
      instead using the memory for something else that's more important.

#. 程序初始化的时候，那些长期运行的程序可能要分配和使用很多页面。这些页面可能在最后的关闭/清理的
   时候还需要使用，但是在程序「启动」之后（以具体的程序相关的方式）持续运行的时候不需要访问。
   对后台服务程序来说，很多后台程序要初始化不少依赖库，这种情况很常见。
#. 在程序的正常运行过程中，我们可能分配一些很少使用的内存。对整体系统性能而言可能比起让这些内存页
   一直留在内存中，只有在用到的时候才按需把它们用 **缺页异常（major fault）** 换入内存，
   可以空出更多内存留给更重要的东西。


.. panel-default::
    :title: `cgroupv2: Linux's new unified control group hierarchy (QCON London 2017) <https://www.youtube.com/watch?v=ikZ8_mRotT4>`_

    .. youtube:: ikZ8_mRotT4

考察有无交换区时会发生什么
----------------------------------------

.. translate-collapse::

   Let's look at typical situations, and how they perform with and without
   swap present. I talk about metrics around "memory contention" in my 
   `talk on cgroup v2 <https://www.youtube.com/watch?v=ikZ8_mRotT4>`_ .

我们来看一些在常见场景中，有无交换区时分别会如何运行。
在我的 `关于 cgroup v2 的演讲 <https://www.youtube.com/watch?v=ikZ8_mRotT4>`_
中探讨过「内存竞争」的指标。

在无/低内存竞争的状态下
++++++++++++++++++++++++++++++++++++++++++++++++

.. translate-collapse::

   - **With swap:** We can choose to swap out rarely-used anonymous memory that
     may only be used during a small part of the process lifecycle,
     allowing us to use this memory to improve cache hit rate,
     or do other optimisations.
   - **Without swap** We cannot swap out rarely-used anonymous memory,
     as it's locked in memory. While this may not immediately
     present as a problem, on some workloads this may represent
     a non-trivial drop in performance due to stale,
     anonymous pages taking space away from more important use.

- **有交换区:** 我们可以选择换出那些只有在进程生存期内很小一部分时间会访问的匿名内存，
  这允许我们空出更多内存空间用来提升缓存命中率，或者做别的优化。
- **无交换区:** 我们不能换出那些很少使用的匿名内存，因为它们被锁在了内存中。虽然这通常不会直接表现出问题，
  但是在一些工作条件下这可能造成卡顿导致不平凡的性能下降，因为匿名内存占着空间不能给
  更重要的需求使用。

.. panel-default::
    :title: 译注：关于 **内存热度** 和 **内存颠簸（thrash）**

    讨论内核中内存管理的时候经常会说到内存页的 **冷热** 程度。这里冷热是指历史上内存页被访问到的频度，
    内存管理的经验在说，历史上在近期频繁访问的 **热** 内存，在未来也可能被频繁访问，
    从而应该留在物理内存里；反之历史上不那么频繁访问的 **冷** 内存，在未来也可能很少被用到，
    从而可以考虑交换到磁盘或者丢弃文件缓存。


    **颠簸（thrash）** 这个词在文中出现多次但是似乎没有详细介绍，实际计算机科学专业的课程中应该有讲过。
    一段时间内，让程序继续运行所需的热内存总量被称作程序的工作集（workset），估算工作集大小，
    换句话说判断进程分配的内存页中哪些属于 **热** 内存哪些属于 **冷** 内存，是内核中
    内存管理的最重要的工作。当分配给程序的内存大于工作集的时候，程序可以不需要等待I/O全速运行；
    而当分配给程序的内存不足以放下整个工作集的时候，意味着程序每执行一小段就需要等待换页或者等待
    磁盘缓存读入所需内存页，产生这种情况的时候，从用户角度来看可以观察到程序肉眼可见的「卡顿」。
    当系统中所有程序都内存不足的时候，整个系统都处于颠簸的状态下，响应速度直接降至磁盘I/O的带宽。
    如本文所说，禁用交换区并不能防止颠簸，只是从等待换页变成了等待文件缓存，
    给程序分配超过工作集大小的内存才能防止颠簸。

在中/高内存竞争的状态下
++++++++++++++++++++++++++++++++++++++++++++++++


.. translate-collapse::

   - **With swap:** All memory types have an equal possibility of being reclaimed. 
     This means we have more chance of being able to reclaim pages
     successfully – that is, we can reclaim pages that are not quickly
     faulted back in again (thrashing).
   - **Without swap** Anonymous pages are locked into memory as they have nowhere to go.
     The chance of successful long-term page reclamation is lower,
     as we have only some types of memory eligible to be reclaimed
     at all. The risk of page thrashing is higher. The casual
     reader might think that this would still be better as it might
     avoid having to do disk I/O, but this isn't true –
     we simply transfer the disk I/O of swapping to dropping
     hot page caches and dropping code segments we need soon.

- **有交换区:** 所有内存类型都有平等的被回收的可能性。这意味着我们回收页面有更高的成功率——
  成功回收的意思是说被回收的那些页面不会在近期内被缺页异常换回内存中（颠簸）。
- **无交换区:** 匿名内存因为无处可去所以被锁在内存中。长期内存回收的成功率变低了，因为我们成体上
  能回收的页面总量少了。发生缺页颠簸的危险性更高了。缺乏经验的读者可能觉得这某时也是好事，
  因为这能避免进行磁盘I/O，但是实际上不是如此——我们只是把交换页面造成的磁盘I/O变成了扔掉热缓存页
  和扔掉代码段，这些页面很可能马上又要从文件中读回来。

在临时内存占用高峰时
++++++++++++++++++++++++++++++++++++++++++++++++

.. translate-collapse::

   - **With swap:** We're more resilient to temporary spikes, but in cases of
     severe memory starvation, the period from memory thrashing beginning
     to the OOM killer may be prolonged. We have more visibility into the
     instigators of memory pressure and can act on them more reasonably,
     and can perform a controlled intervention.
   - **Without swap** The OOM killer is triggered more quickly as anonymous
     pages are locked into memory and cannot be reclaimed. We're more likely to
     thrash on memory, but the time between thrashing and OOMing is reduced.
     Depending on your application, this may be better or worse. For example,
     a queue-based application may desire this quick transfer from thrashing
     to killing. That said, this is still too late to be really useful –
     the OOM killer is only invoked at moments of severe starvation,
     and relying on this method for such behaviour would be better replaced
     with more opportunistic killing of processes as memory contention
     is reached in the first place.

- **有交换区:** 我们对内存使用激增的情况更有抵抗力，但是在严重的内存不足的情况下，
  从开始发生内存颠簸到 OOM 杀手开始工作的时间会被延长。内存压力造成的问题更容易观察到，
  从而可能更有效地应对，或者更有机会可控地干预。
- **无交换区:** 因为匿名内存被锁在内存中了不能被回收，所以 OOM 杀手会被更早触发。
  发生内存颠簸的可能性更大，但是发生颠簸之后到 OOM 解决问题的时间间隔被缩短了。
  基于你的程序，这可能更好或是更糟。比如说，基于队列的程序可能更希望这种从颠簸到杀进程的转换更快发生。
  即便如此，发生 OOM 的时机通常还是太迟于是没什么帮助——只有在系统极度内存紧缺的情况下才会请出
  OOM 杀手，如果想依赖这种行为模式，不如换成更早杀进程的方案，因为在这之前已经发生内存竞争了。

好吧，所以我需要系统交换区，但是我该怎么为每个程序微调它的行为？
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

.. translate-collapse::

   You didn't think you'd get through this entire post without me plugging cgroup v2, did you? ;-)

你肯定想到了我写这篇文章一定会在哪儿插点 cgroup v2 的安利吧？ ;-)

.. translate-collapse::

   Obviously, it's hard for a generic heuristic algorithm to be right all the time,
   so it's important for you to be able to give guidance to the kernel.
   Historically the only tuning you could do was at the system level,
   using :code:`vm.swappiness` . This has two problems: :code:`vm.swappiness`
   is incredibly hard to reason about because it only feeds in as
   a small part of a larger heuristic system, and it also is system-wide
   instead of being granular to a smaller set of processes.

显然，要设计一种对所有情况都有效的启发算法会非常难，所以给内核提一些指引就很重要。
历史上我们只能在整个系统层面做这方面微调，通过 :code:`vm.swappiness` 。这有两方面问题：
:code:`vm.swappiness` 的行为很难准确控制，因为它只是传递给一个更大的启发式算法中的一个小参数；
并且它是一个系统级别的设置，没法针对一小部分进程微调。

.. translate-collapse::

   You can also use :code:`mlock` to lock pages into memory, but this requires
   either modifying program code, fun with :code:`LD_PRELOAD` , or doing
   horrible things with a debugger at runtime.
   In VM-based languages this also doesn't work very well, since you
   generally have no control over allocation and end up having to
   :code:`mlockall` , which has no precision towards the pages
   you actually care about.

你可以用 :code:`mlock` 把页面锁在内存里，但是这要么必须改程序代码，或者折腾
:code:`LD_PRELOAD` ，或者在运行期用调试器做一些魔法操作。对基于虚拟机的语言来说这种方案也不能
很好工作，因为通常你没法控制内存分配，最后得用上 :code:`mlockall`
，而这个没有办法精确指定你实际上想锁住的页面。


.. translate-collapse::

   cgroup v2 has a tunable per-cgroup in the form of :code:`memory.low`
   , which allows us to tell the kernel to prefer other applications for
   reclaim below a certain threshold of memory used. This allows us to not
   prevent the kernel from swapping out parts of our application,
   but prefer to reclaim from other applications under memory contention.
   Under normal conditions, the kernel's swap logic is generally pretty good,
   and allowing it to swap out pages opportunistically generally increases
   system performance. Swap thrash under heavy memory contention is not ideal,
   but it's more a property of simply running out of memory entirely than
   a problem with the swapper. In these situations, you typically want to
   fail fast by self-killing non-critical processes when memory pressure
   starts to build up.

cgroup v2 提供了一套可以每个 cgroup 微调的 :code:`memory.low`
，允许我们告诉内核说当使用的内存低于一定阈值之后优先回收别的程序的内存。这可以让我们不强硬禁止内核
换出程序的一部分内存，但是当发生内存竞争的时候让内核优先回收别的程序的内存。在正常条件下，
内核的交换逻辑通常还是不错的，允许它有条件地换出一部分页面通常可以改善系统性能。在内存竞争的时候
发生交换颠簸虽然不理想，但是这更多地是单纯因为整体内存不够了，而不是因为交换进程（swapper）导致的问题。
在这种场景下，你通常希望在内存压力开始积攒的时候通过自杀一些非关键的进程的方式来快速退出（fail fast）。

.. translate-collapse::

   You can not simply rely on the OOM killer for this. The OOM killer is
   only invoked in situations of dire failure when we've already entered
   a state where the system is severely unhealthy and may well have been
   so for a while. You need to opportunistically handle the situation yourself
   before ever thinking about the OOM killer.

你不能依赖 OOM 杀手达成这个。 OOM 杀手只有在非常急迫的情况下才会出动，那时系统已经处于极度不健康的
状态了，而且很可能在这种状态下保持了一阵子了。需要在开始考虑 OOM 杀手之前，积极地自己处理这种情况。

.. translate-collapse::

   Determination of memory pressure is somewhat difficult using traditional
   Linux memory counters, though. We have some things which seem somewhat related,
   but are merely tangential – memory usage, page scans, etc – and from these
   metrics alone it's very hard to tell an efficient memory configuration
   from one that's trending towards memory contention. There is a group of us
   at Facebook, spearheaded by `Johannes <https://patchwork.kernel.org/project/LKML/list/?submitter=45>`_
   , working on developing new metrics that expose memory pressure more easily
   that should help with this in future. If you're interested in hearing more
   about this, 
   `I go into detail about one metric being considered in my talk on cgroup v2 <https://youtu.be/ikZ8_mRotT4?t=2145>`_.

不过，用传统的 Linux 内存统计数据还是挺难判断内存压力的。我们有一些看起来相关的系统指标，但是都
只是支离破碎的——内存用量、页面扫描，这些——单纯从这些指标很难判断系统是处于高效的内存利用率还是
在滑向内存竞争状态。我们在 Facebook 有个团队，由
`Johannes`_
牵头，努力开发一些能评价内存压力的新指标，希望能在今后改善目前的现状。
如果你对这方面感兴趣， `在我的 cgroup v2 的演讲中介绍到一个被提议的指标 <https://youtu.be/ikZ8_mRotT4?t=2145>`_
。

调优
----------------------------------------

那么，我需要多少交换空间？
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++


.. translate-collapse::

   In general, the minimum amount of swap space required for optimal
   memory management depends on the number of anonymous pages pinned into
   memory that are rarely reaccessed by an application, and the value of
   reclaiming those anonymous pages. The latter is mostly a question of
   which pages are no longer purged to make way for these infrequently
   accessed anonymous pages.

通常而言，最优内存管理所需的最小交换空间取决于程序固定在内存中而又很少访问到的匿名页面的数量，
以及回收这些匿名页面换来的价值。后者大体上来说是问哪些页面不再会因为要保留这些很少访问的匿名页面而
被回收掉腾出空间。

.. translate-collapse::

   If you have a bunch of disk space and a recent (4.0+) kernel,
   more swap is almost always better than less. In older kernels :code:`kswapd`,
   one of the kernel processes responsible for managing swap, was historically
   very overeager to swap out memory aggressively the more swap you had.
   In recent times, swapping behaviour when a large amount of swap space is
   available has been significantly improved. If you're running kernel 4.0+,
   having a larger swap on a modern kernel should not result in overzealous
   swapping. As such, if you have the space, having a swap size of a few GB
   keeps your options open on modern kernels.

如果你有足够大的磁盘空间和比较新的内核版本（4.0+），越大的交换空间基本上总是越好的。
更老的内核上 :code:`kswapd` ， 内核中负责管理交换区的内核线程，在历史上倾向于有越多交换空间就
急于交换越多内存出去。在最近一段时间，可用交换空间很大的时候的交换行为已经改善了很多。
如果在运行 4.0+ 以后的内核，即便有很大的交换区在现代内核上也不会很激进地做交换。因此，
如果你有足够的容量，现代内核上有个几个 GB 的交换空间大小能让你有更多选择。

.. translate-collapse::

   If you're more constrained with disk space, then the answer really
   depends on the tradeoffs you have to make, and the nature of the environment.
   Ideally you should have enough swap to make your system operate optimally
   at normal and peak (memory) load. What I'd recommend is setting up a few
   testing systems with 2-3GB of swap or more, and monitoring what happens
   over the course of a week or so under varying (memory) load conditions.
   As long as you haven't encountered severe memory starvation during that week
   – in which case the test will not have been very useful – you will probably
   end up with some number of MB of swap occupied. As such, it's probably worth
   having at least that much swap available, in addition to a little buffer for
   changing workloads. :code:`atop` in logging mode can also show you which applications
   are having their pages swapped out in the :code:`SWAPSZ` column, so if you don't
   already use it on your servers to log historic server state you probably
   want to set it up on these test machines with logging mode as part of this
   experiment. This also tells you when your application started swapping out
   pages, which you can tie to log events or other key data.

如果你的磁盘空间有限，那么答案更多取决于你愿意做的取舍，以及运行的环境。理想上应该有足够的交换空间
能高效应对正常负载和高峰（内存）负载。我建议先用 2-3GB 或者更多的交换空间搭个测试环境，
然后监视在不同（内存）负载条件下持续一周左右的情况。只要在那一周里没有发生过严重的内存不足——
发生了的话说明测试结果没什么用——在测试结束的时候大概会留有多少 MB 交换区占用。
作为结果说明你至少应该有那么多可用的交换空间，再多加一些以应对负载变化。用日志模式跑 :code:`atop`
可以在 :code:`SWAPSZ` 栏显示程序的页面被交换出去的情况，所以如果你还没用它记录服务器历史日志的话
，这次测试中可以试试在测试机上用它记录日志。这也会告诉你什么时候你的程序开始换出页面，你可以用这个
对照事件日志或者别的关键数据。

.. translate-collapse::

   Another thing worth considering is the nature of the swap medium.
   Swap reads tend to be highly random, since we can't reliably predict
   which pages will be refaulted and when. On an SSD this doesn't matter much,
   but on spinning disks, random I/O is extremely expensive since it requires
   physical movement to achieve. On the other hand, refaulting of file pages
   is likely less random, since files related to the operation of a single
   application at runtime tend to be less fragmented. This might mean that on
   a spinning disk you may want to bias more towards reclaiming file pages
   instead of swapping out anonymous pages, but again, you need to test and
   evaluate how this balances out for your workload.

另一点值得考虑的是交换空间所在存储设备的媒介。读取交换区倾向于很随机，因为我们不能可靠预测什么时候
什么页面会被再次访问。在 SSD 上这不是什么问题，但是在传统磁盘上，随机 I/O 操作会很昂贵，
因为需要物理动作寻道。另一方面，重新加载文件缓存可能不那么随机，因为单一程序在运行期的文件读操作
一般不会太碎片化。这可能意味着在传统磁盘上你想更多地回收文件页面而不是换出匿名页面，但仍就，
你需要做测试评估在你的工作负载下如何取得平衡。


.. panel-default::
    :title: 译注：关于休眠到磁盘时的交换空间大小

    原文这里建议交换空间至少是物理内存大小，我觉得实际上不需要。休眠到磁盘的时候内核会写回并丢弃
    所有有文件作后备的可回收页面，交换区只需要能放下那些没有文件后备的页面就可以了。
    如果去掉文件缓存页面之后剩下的已用物理内存总量能完整放入交换区中，就可以正常休眠。
    对于桌面浏览器这种内存大户，通常有很多缓存页可以在休眠的时候丢弃。

.. translate-collapse::

   For laptop/desktop users who want to hibernate to swap, this also needs to
   be taken into account – in this case your swap file should be at least
   your physical RAM size.

对笔记本/桌面用户如果想要休眠到交换区，这也需要考虑——这种情况下你的交换文件应该至少是物理内存大小。

我的 swappiness 应该如何设置？
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

.. translate-collapse::

   First, it's important to understand what :code:`vm.swappiness` does.
   :code:`vm.swappiness` is a sysctl that biases memory reclaim either towards
   reclamation of anonymous pages, or towards file pages. It does this using two
   different attributes: :code:`file_prio` (our willingness to reclaim file pages)
   and :code:`anon_prio` (our willingness to reclaim anonymous pages).
   :code:`vm.swappiness`plays into this, as it becomes the default value for
   :code:`anon_prio`, and it also is subtracted from the default value of 200
   for :code:`file_prio`, which means for a value of :code:`vm.swappiness = 50`,
   the outcome is that :code:`anon_prio` is 50, and :code:`file_prio` is 150
   (the exact numbers don't matter as much as their relative weight compared to the other).

首先很重要的一点是，要理解 :code:`vm.swappiness` 是做什么的。
:code:`vm.swappiness` 是一个 sysctl 用来控制在内存回收的时候，是优先回收匿名页面，
还是优先回收文件页面。内存回收的时候用两个属性： :code:`file_prio` （回收文件页面的倾向）
和 :code:`anon_prio` （回收匿名页面的倾向）。 :code:`vm.swappiness` 控制这两个值，
因为它是 :code:`anon_prio` 的默认值，然后也是默认 200 减去它之后 :code:`file_prio` 的默认值。
意味着如果我们设置 :code:`vm.swappiness = 50` 那么结果是 :code:`anon_prio` 是 50，
:code:`file_prio` 是 150 （这里数值本身不是很重要，重要的是两者之间的权重比）。


.. translate-collapse::

   This means that, in general, :code:`vm.swappiness` **is simply a ratio of how**
   **costly reclaiming and refaulting anonymous memory is compared to file memory**
   **for your hardware and workload**. The lower the value, the more you tell the
   kernel that infrequently accessed anonymous pages are expensive to swap out
   and in on your hardware. The higher the value, the more you tell the kernel
   that the cost of swapping anonymous pages and file pages is similar on your
   hardware. The memory management subsystem will still try to mostly decide
   whether it swaps file or anonymous pages based on how hot the memory is,
   but swappiness tips the cost calculation either more towards swapping or
   more towards dropping filesystem caches when it could go either way.
   On SSDs these are basically as expensive as each other, so setting
   :code:`vm.swappiness = 100` (full equality) may work well.
   On spinning disks, swapping may be significantly more expensive since
   swapping in generally requires random reads, so you may want to
   bias more towards a lower value.

这意味着，通常来说 :code:`vm.swappiness` **只是一个比例，用来衡量在你的硬件和工作负载下，**
**回收和换回匿名内存还是文件内存哪种更昂贵** 。设定的值越低，你就是在告诉内核说换出那些不常访问的
匿名页面在你的硬件上开销越昂贵；设定的值越高，你就是在告诉内核说在你的硬件上交换匿名页和
文件缓存的开销越接近。内存管理子系统仍然还是会根据实际想要回收的内存的访问热度尝试自己决定具体是
交换出文件还是匿名页面，只不过 swappiness 会在两种回收方式皆可的时候，在计算开销权重的过程中左右
是该更多地做交换还是丢弃缓存。在 SSD 上这两种方式基本上是同等开销，所以设成
:code:`vm.swappiness = 100` （同等比重）可能工作得不错。在传统磁盘上，交换页面可能会更昂贵，
因为通常需要随机读取，所以你可能想要设低一些。

.. translate-collapse::

   The reality is that most people don't really have a feeling about which
   their hardware demands, so it's non-trivial to tune this value based on
   instinct alone – this is something that you need to test using different
   values. You can also spend time evaluating the memory composition of your
   system and core applications and their behaviour under mild memory reclamation.

现实是大部分人对他们的硬件需求没有什么感受，所以根据直觉调整这个值可能挺困难的 ——
你需要用不同的值做测试。你也可以花时间评估一下你的系统的内存分配情况和核心应用在大量回收内存的时候的行为表现。

.. translate-collapse::

   When talking about :code:`vm.swappiness` , an extremely important change to
   consider from recent(ish) times is 
   `this change to vmscan by Satoru Moriya in 2012 <https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/patch/?id=fe35004fbf9eaf67482b074a2e032abb9c89b1dd>`_
   , which changes the way that :code:`vm.swappiness = 0` is handled
   quite significantly.

讨论 :code:`vm.swappiness` 的时候，一个极为重要需要考虑的修改是（相对）近期在
`2012 年左右 Satoru Moriya 对 vmscan 行为的修改 <https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/patch/?id=fe35004fbf9eaf67482b074a2e032abb9c89b1dd>`_
，它显著改变了代码对 :code:`vm.swappiness = 0` 这个值的处理方式。

.. translate-collapse::

   Essentially, the patch makes it so that we are extremely biased against
   scanning (and thus reclaiming) any anonymous pages at all with
   :code:`vm.swappiness = 0` , unless we are already encountering severe
   memory contention. As mentioned previously in this post, that's generally
   not what you want, since this prevents equality of reclamation prior to
   extreme memory pressure occurring, which may actually lead to this
   extreme memory pressure in the first place. :code:`vm.swappiness = 1`
   is the lowest you can go without invoking the special casing for
   anonymous page scanning implemented in that patch.

基本上来说这个补丁让我们在 :code:`vm.swappiness = 0` 的时候会极度避免扫描（进而回收）匿名页面，
除非我们已经在经历严重的内存抢占。就如本文前面所属，这种行为基本上不会是你想要的，
因为这种行为会导致在发生内存抢占之前无法保证内存回收的公平性，这甚至可能是最初导致发生内存抢占的原因。
想要避免这个补丁中对扫描匿名页面的特殊行为的话， :code:`vm.swappiness = 1` 是你能设置的最低值。


.. translate-collapse::

   The kernel default here is :code:`vm.swappiness = 60`. This value is
   generally not too bad for most workloads, but it's hard to have a
   general default that suits all workloads. As such, a valuable extension
   to the tuning mentioned in the "how much swap do I need" section above
   would be to test these systems with differing values for :code:`vm.swappiness`
   , and monitor your application and system metrics under heavy (memory) load.
   Some time in the near future, once we have a decent implementation of
   `refault detection <https://youtu.be/ikZ8_mRotT4?t=2145>`_ in the kernel,
   you'll also be able to determine this somewhat workload-agnostically by
   looking at cgroup v2's page refaulting metrics.

内核在这里设置的默认值是 :code:`vm.swappiness = 60` 。这个值对大部分工作负载来说都不会太坏，
但是很难有一个默认值能符合所有种类的工作负载。因此，对上面「 `那么，我需要多少交换空间？`_
」那段讨论的一点重要扩展可以说，在测试系统中可以尝试使用不同的 :code:`vm.swappiness`
，然后监视你的程序和系统在重（内存）负载下的性能指标。在未来某天，如果我们在内核中有了合理的
`缺页检测 <https://youtu.be/ikZ8_mRotT4?t=2145>`_ ，你也将能通过 cgroup v2 的页面缺页
指标来以负载无关的方式决定这个。


.. panel-default::
    :title: `SREcon19 Asia/Pacific - Linux Memory Management at Scale: Under the Hood <https://www.youtube.com/watch?v=beefUhRH5lU>`_

    .. youtube:: beefUhRH5lU

2019年07月更新：内核 4.20+ 中的内存压力指标
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

.. translate-collapse::

   The refault metrics mentioned as in development earlier are now in the
   kernel from 4.20 onwards and can be enabled with :code:`CONFIG_PSI=y`
   . See my talk at SREcon at around the 25:05 mark:

前文中提到的开发中的内存缺页检测指标已经进入 4.20+ 以上版本的内核，可以通过
:code:`CONFIG_PSI=y` 开启。详情参见我在 SREcon 大约 25:05 左右的讨论。

结论
-----------------------------------------------

.. translate-collapse::

   - Swap is a useful tool to allow equality of reclamation of memory pages,
     but its purpose is frequently misunderstood, leading to its negative
     perception across the industry. If you use swap in the spirit intended,
     though – as a method of increasing equality of reclamation – you'll
     find that it's a useful tool instead of a hindrance.
   - Disabling swap does not prevent disk I/O from becoming a problem under
     memory contention, it simply shifts the disk I/O thrashing from anonymous
     pages to file pages. Not only may this be less efficient, as we have
     a smaller pool of pages to select from for reclaim, but it may also
     contribute to getting into this high contention state in the first place.
   - Swap can make a system slower to OOM kill, since it provides another,
     slower source of memory to thrash on in out of memory situations – the
     OOM killer is only used by the kernel as a last resort, after things have
     already become monumentally screwed. The solutions here depend on your system:

     - You can opportunistically change the system workload depending on
       cgroup-local or global memory pressure. This prevents getting into these
       situations in the first place, but solid memory pressure metrics are
       lacking throughout the history of Unix. Hopefully this should be
       better soon with the addition of refault detection.
     - You can bias reclaiming (and thus swapping) away from certain processes
       per-cgroup using memory.low, allowing you to protect critical daemons
       without disabling swap entirely.

- 交换区是允许公平地回收内存的有用工具，但是它的目的经常被人误解，导致它在业内这种负面声誉。如果
  你是按照原本的目的使用交换区的话——作为增加内存回收公平性的方式——你会发现它是很有效的工具而不是阻碍。
- 禁用交换区并不能在内存竞争的时候防止磁盘I/O的问题，它只不过把匿名页面的磁盘I/O变成了文件页面的
  磁盘I/O。这不仅更低效，因为我们回收内存的时候能选择的页面范围更小了，而且它可能是导致高度内存竞争
  状态的元凶。
- 有交换区会导致系统更慢地使用 OOM 杀手，因为在缺少内存的情况下它提供了另一种更慢的内存，
  会持续地内存颠簸——内核调用 OOM 杀手只是最后手段，会晚于所有事情已经被搞得一团糟之后。
  解决方案取决于你的系统：

  - 你可以预先更具每个 cgroup 的或者系统全局的内存压力改变系统负载。这能防止我们最初进入内存竞争
    的状态，但是 Unix 的历史中一直缺乏可靠的内存压力检测方式。希望不久之后在有了
    `缺页检测 <https://youtu.be/ikZ8_mRotT4?t=2145>`_ 这样的性能指标之后能改善这一点。
  - 你可以使用 :code:`memory.low` 让内核不倾向于回收（进而交换）特定一些 cgroup 中的进程，
    允许你在不禁用交换区的前提下保护关键后台服务。

-------------------------

感谢在撰写本文时 `Rahul <https://github.com/rahulg>`_ ，
`Tejun <https://github.com/htejun>`_ 和 
`Johannes <https://patchwork.kernel.org/project/LKML/list/?submitter=45>`_
提供的诸多建议和反馈。