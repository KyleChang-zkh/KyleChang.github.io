---
layout:     post
title:      OOM-Killer机制
subtitle:   上次还没被CGroup坑完呢
date:       2018-12-24
author:     Kyle
header-img: img/post-bg-2015.jpg
catalog: 	 true
tags:
    - Linux
    - OOM
    - OOM-Killer
    - CGroup
---

## OOM是什么

首先OOM是out of memory的简称，顾名思义，就是内存不足了。

之所以会有OOM这个问题，是因为Linux内核根据应用程序的要求分配内存的。

通常来说应用程序分配了内存但是并没有实际全部使用，为了提高性能，这部分没用的内存就会先作它用。

这部分内存是属于每个进程的，内核直接回收利用的话比较麻烦，所以内核采用一种过度分配内存（over-commit memory）的办法来间接利用这部分“空闲”的内存，提高整体内存的使用效率。

一般来说这样做是没有问题的，但当大多数应用程序都消耗完自己的内存的时候麻烦就来了，因为这些应用程序的内存需求加起来超出了物理内存（包括swap）的容量，内核就会杀掉一些进程(OOM Killer)来腾出空间保障系统正常运行。

用银行的例子来讲可能更容易懂一些，部分人取钱的时候银行不怕，银行有足够的存款应付。

但当全国人民（或者绝大多数）都在差不多的时候要取钱而且每个人都想把自己钱取完的时候银行的麻烦就来了，银行实际上是没有这么现金给大家取的。

## OOM Killer

每当遇到OOM的情况，OOM Killer就出来了，他会杀掉占用内存过大的进程，来释放内存空间，防止内存耗尽，系统崩溃。

如果你忽然遇到某进程总是无缘无故挂掉/经常死机/任务因等待进程挂掉而经常阻塞并且找不到规律时，就可以检查一下是不是OOM问题了，去`/var/log/messages`看看，没准会有意外的收获。

## OOM Killer的回收选择机制

用一句话来概括的话，OOM Killer的选择机制就是选择那个能释放出最多实际内存（不含swap）的进程，也就是kill掉对回收内存受益最大的进程来救急。

具体的代码在`linux/mm/oom_kill.c`下.

```
/**
 * out_of_memory - kill the "best" process when we run out of memory
 */
void out_of_memory(struct zonelist *zonelist, gfp_t gfp_mask,
        int order, nodemask_t *nodemask, bool force_kill)
{
    // 等待notifier调用链返回，如果有内存了则返回
    blocking_notifier_call_chain(&oom_notify_list, 0, &freed);
    if (freed > 0)
        return;
 
    // 如果进程即将退出，则表明可能会有内存可以使用了，返回
    if (fatal_signal_pending(current) || current->flags & PF_EXITING) {
        set_thread_flag(TIF_MEMDIE);
        return;
    }
 
    // 如果设置了sysctl的panic_on_oom，则内核直接panic
    check_panic_on_oom(constraint, gfp_mask, order, mpol_mask);
 
    // 如果设置了oom_kill_allocating_task
    // 则杀死正在申请内存的process
    if (sysctl_oom_kill_allocating_task && current->mm &&
        !oom_unkillable_task(current, NULL, nodemask) &&
        current->signal->oom_score_adj != OOM_SCORE_ADJ_MIN) {
        get_task_struct(current);
        oom_kill_process(current, gfp_mask, order, 0, totalpages, NULL,
                 nodemask,
                 "Out of memory (oom_kill_allocating_task)");
        goto out;
    }
 
    // 用select_bad_process()选择badness指
    // 数(oom_score)最高的进程
    p = select_bad_process(&points, totalpages, mpol_mask, force_kill);
 
 
    if (!p) {
        dump_header(NULL, gfp_mask, order, NULL, mpol_mask);
        panic("Out of memory and no killable processes...\n");
    }
    if (p != (void *)-1UL) {
        // 查看child process, 是否是要被killed，则直接影响当前这个parent进程 
        oom_kill_process(p, gfp_mask, order, points, totalpages, NULL,
                 nodemask, "Out of memory");
        killed = 1;
    }
out:
 
    if (killed)
        schedule_timeout_killable(1);
}
```

```
/**
 * oom_badness - heuristic function to determine which candidate task to kill
 * 
 */
unsigned long oom_badness(struct task_struct *p, struct mem_cgroup *memcg,
              const nodemask_t *nodemask, unsigned long totalpages)
{
    long points;
    long adj;

    // 内部判断是否是pid为1的initd进程，是否是kthread内核进程，是否是其他cgroup，如果是则跳过
    if (oom_unkillable_task(p, memcg, nodemask))
        return 0;

    p = find_lock_task_mm(p);
    if (!p)
        return 0;

    // 获得/proc/[pid]/oom_adj权值，如果是OOM_SCORE_ADJ_MIN则返回
    adj = (long)p->signal->oom_score_adj;
    if (adj == OOM_SCORE_ADJ_MIN) {
        task_unlock(p);
        return 0;
    }

    // 获得进程RSS和swap内存占用
    points = get_mm_rss(p->mm) + p->mm->nr_ptes +
         get_mm_counter(p->mm, MM_SWAPENTS);
    task_unlock(p);

    // 计算步骤如下，【计算逻辑比较简单，不赘述了】
    if (has_capability_noaudit(p, CAP_SYS_ADMIN))
        adj -= 30;
    adj *= totalpages / 1000;
    points += adj;

    return points > 0 ? points : 1;

}

```

从代码中我们可以看到oom_badness()给每个进程打分，根据points的高低来决定杀哪个进程，这个points可以根据adj调节。

root权限的进程通常被认为很重要，不应该被轻易杀掉，所以打分的时候可以得到3%的优惠（分数越低越不容易被杀掉）。

我们可以在用户空间通过操作每个进程的oom_adj内核参数来决定哪些进程不这么容易被OOM Killer选中杀掉。

比如，如果不想A进程被轻易杀掉的话可以找到A运行的进程号后，调整`/proc/PID/oom_score_adj`为-15（注意 points越小越不容易被杀）防止重要的系统进程触发OOM机制而被杀死。

内核会通过特定的算法给每个进程计算一个分数来决定杀哪个进程，每个进程的oom分数可以在`/proc/PID/oom_score`中找到。

每个进程都有一个oom_score的属性，OOM Killer会杀死oom_score较大的进程，当oom_score为0时禁止内核杀死该进程。

设置/proc/PID/oom_adj可以改变oom_score，oom_adj的范围为【-17，15】，其中15最大-16最小，-17为禁止使用OOM。

至于为什么用-17而不用其他数值（默认值为0），这个是由linux内核定义的，查看内核源码可知：路径为`linux-xxxxx/include/uapi/linux/oom.h`。


## OOM Killer的相关日志

`/var/log/messages`会记录一些系统日志，包括OOM Killer记录。

例如这就是一次OOM Killer的记录。（实际遇到的日志不方便写在外网，这个记录是网上扒的，还请见谅。）

```
Apr 18 16:56:16 v125000100.bja kernel: : [22254383.898423] Out of memory: Kill process 24894 (big_mm) score 277 or sacrifice child
Apr 18 16:56:16 v125000100.bja kernel: : [22254383.899708] Killed process 24894, UID 55120, (big_mm) total-vm:2301932kB, anon-rss:2228452kB, file-rss:24kB
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738942] big_mm invoked oom-killer: gfp_mask=0x280da, order=0, oom_adj=0, oom_score_adj=0
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738947] big_mm cpuset=/ mems_allowed=0
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738950] Pid: 24893, comm: big_mm Not tainted 2.6.32-220.23.2.ali878.el6.x86_64 #1
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738952] Call Trace:
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738961]  [<ffffffff810c35e1>] ? cpuset_print_task_mems_allowed+0x91/0xb0
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738968]  [<ffffffff81114d70>] ? dump_header+0x90/0x1b0
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738973]  [<ffffffff810e1b2e>] ? __delayacct_freepages_end+0x2e/0x30
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738979]  [<ffffffff81213ffc>] ? security_real_capable_noaudit+0x3c/0x70
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738982]  [<ffffffff811151fa>] ? oom_kill_process+0x8a/0x2c0
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738985]  [<ffffffff81115131>] ? select_bad_process+0xe1/0x120
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738989]  [<ffffffff81115650>] ? out_of_memory+0x220/0x3c0
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.738995]  [<ffffffff81125929>] ? __alloc_pages_nodemask+0x899/0x930
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.739001]  [<ffffffff81159c6a>] ? alloc_pages_vma+0x9a/0x150
```

messages还会打印出OOM Killer发生时的score：

```
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.758297] [ pid ]   uid  tgid total_vm      rss cpu oom_adj oom_score_adj name
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.758311] [  399]     0   399     2709      133   2     -17         -1000 udevd
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.758314] [  810]     0   810     2847       43   0       0             0 svscanboot
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.758317] [  824]     0   824     1039       21   0       0             0 svscan
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.758320] [  825]     0   825      993       17   1       0             0 readproctitle
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.758322] [  826]     0   826      996       16   0       0             0 supervise
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.758325] [  827]     0   827      996       17   0       0             0 supervise
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.758327] [  828]     0   828      996       16   0       0             0 supervise
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.758330] [  829]     0   829      996       17   2       0             0 supervise
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.758333] [  830]     0   830     6471      152   0       0             0 run
Apr 18 16:56:18 v125000100.bja kernel: : [22254386.758335] [  831]    99   831     1032       21   0       0             0 multilog
```

## 关于OOM的其他注意事项

不一定只有在设置ULIMIT或者全局内存资源耗尽时才会触发OOM，针对某一些内存设置独立的资源限制也有可能触发OOM Killer。例如CGroup，相关信息可以查看上一篇文章<a href="https://skilechang.github.io/2018/12/16/cgroup/" target="_blank">『被CGroup坑的那些事』</a>。

---

## 参考

<a href="https://blog.csdn.net/gugemichael/article/details/24017515" target="_blank">『Linux -- 内存控制之oom killer机制及代码分析』</a>

<a href="https://www.cnblogs.com/jjmcao/p/9450383.html" target="_blank">『Linux内核OOM机制的详细分析』</a>

<a href="http://senlinzhan.github.io/2017/07/03/oom-killer/">『Linux 的 OOM Killer 机制分析』</a>

<a href="http://linux-mm.org/OOM_Killer">『http://linux-mm.org/OOM_Killer』</a>
