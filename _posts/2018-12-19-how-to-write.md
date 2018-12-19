---
layout: post
title: skynet启动流程(二)
date: 2018-12-19
categories: blog
tags: [skynet启动流程]
description: 文章金句。
---
* monitor模块
	```
	//skynet_monitor.c文件
	thread_monitor(void *p) {
		struct monitor * m = p;
		int i;
		int n = m->count;
		skynet_initthread(THREAD_MONITOR); //线程私有变量赋值
		for (;;) {
			CHECK_ABORT // 当服务节点为0退出
			for (i=0;i<n;i++) {
				skynet_monitor_check(m->m[i]);
			}
			for (i=0;i<5;i++) {
				CHECK_ABORT
				sleep(1);
			}
		}
		return NULL;
	}
	//具体工作_skynet_monitor_check_
	skynet_monitor_check(struct skynet_monitor *sm) {
		if (sm->version == sm->check_version) {
			if (sm->destination) {
				skynet_context_endless(sm->destination);
				skynet_error(NULL, "A message from [ :%08x ] to [ :%08x ] maybe in an endless loop (version = %d)", sm->source , sm->destination, sm->version);
			}
		} else {
			sm->check_version = sm->version;
		}
	}
	```

　　当检查版本号和当前版本号一至且目标地址存在(服务之间的消息可以没源地址但是一定有目标地址),则认为消息没有处理完毕打印告警,可能存在死循环,还可以通过上述代码可以知道检查频率为1秒一次。所以看到这个告警需要检查是否死循环。
CHECK_ABORT 是检查到服务节点数为0退出，但是启动线程前，服务器已经启动了日志服务和snlua服务，所以通常不可能为0，那么只有再skynet退出才可能为0(后续线程这个函数使用也应该是这个原因:检查安全退出skynet)，不过没有找到安全退出相关代码，通过云风大大的博客猜测应该没有合入.<br>
　　skynet_context_endless(sm->destination) 检查目标服务引用是否为0，准备回收资源。
这里并不修正目标地址不存或者失效的情况，那么说明应该在其他对这种情况做了保证。  

* timer模块
　　Skynet定时器模块是以计数器的方式实现的usleep(2500) 定时唤醒，更新计数器，然后唤醒所有work线程,周而复始。需要说明的就是定时器的时间区域划分：<br>
　　1.超时近点事件哈希表near[TIME_NEAR]，以超时期望时间的高24位和当前时间的高24位相比较，相等说明超时时间快到了，这部分超时事件以低8位为key值保存在near[key]哈希表中。<br>
　　2.超时远点多级哈希表t[4][TIME_LEVEL],level分别是以高18，12，6位相比较对应二维数组0，1，2，3其中3表示都不相等,key值从第8位开始每6位取值。比如高18位相等的超时期望时间t[0]>[key],key是9位到15位的值。这样每次超时事件定位的复杂度都是O(1),只需要通过滴答计数器的低8位就可以定位near[TIME_NEAR]触发消息。然后远点超时需要通过滴答计数刚好跨level进位，将对应的时间level重新分组。

	```
	//skynet_timer.c文件
	skynet_updatetime(void) { // 时差计数
		uint64_t cp = gettime();
		if(cp < TI->current_point) { // 修复时间反转
			skynet_error(NULL, "time diff error: change from %lld to %lld", cp, TI->current_point);
			TI->current_point = cp;
		} else if (cp != TI->current_point) {
			uint32_t diff = (uint32_t)(cp - TI->current_point);
			TI->current_point = cp;
			TI->current += diff;
			int i;
			for (i=0;i<diff;i++) { // 每百分之一秒更新定时任务
				timer_update(TI);
			}
		}
	}
	timer_update(struct timer *T) {
		SPIN_LOCK(T);
		// try to dispatch timeout 0 (rare condition)
		timer_execute(T); // 检查执行
		// shift time first, and then dispatch timer message
		timer_shift(T); // 检查滴答计数并且检查跨level时间
		timer_execute(T);// 再次检查执行，可以快速执行上面重新分组的level时间。
		SPIN_UNLOCK(T);
	}
	static void
	timer_shift(struct timer *T) {
		int mask = TIME_NEAR;
		uint32_t ct = ++T->time;
		if (ct == 0) { 计数溢出处理
			move_list(T, 3, 0);
		} else {
			uint32_t time = ct >> TIME_NEAR_SHIFT;
			int i=0;
			while ((ct & (mask-1))==0) {
				int idx=time & TIME_LEVEL_MASK;
				if (idx!=0) {
					move_list(T, i, idx);
					break;				
				}
				mask <<= TIME_LEVEL_SHIFT;
				time >>= TIME_LEVEL_SHIFT;
				++i;
			}
		}
	}
	move_list(struct timer *T, int level, int idx) {
		struct timer_node *current = link_clear(&T->t[level][idx]);
		while (current) {
			struct timer_node *temp=current->next;
			add_node(T,current);
			current=temp;
		}
	}
	```

　　需要注意的是timer_shift(T)中，++T->time可能会溢出归零的情况，然后这种情况下为什么需要level 3中key值为0的超时事件全部重新分组呢。<br>
　　1.从参数类型可知超时期望溢出从零开始后高位必然和当前时间不等,所以level错位定位为3。<br>
　　2.如果实际超时期望不是level3的话，那么其key必然为0所以只有level-3 key-0可能存在错误分组每次溢出重新对该组重新分组就好了。<br>

* socket网络模块
	> 由于这个模块并不是由纯c语言控制，涉及到lua层, 所以后续文章讲解了skynet从c到lua层的控制权转移后再补充，会更清晰。
  