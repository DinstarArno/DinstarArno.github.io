---
layout: post
title: skynet中的基础数据结构(一)
date: 2018-12-14
categories: blog
tags: [skynet,数据结构]
description: 文章金句。
---
*skynet网络节点 每个skynet都是全局网络中的一个节点
'''
struct skynet_node {
	int total;		// 节点总数
	int init;      // 初始化标志
	uint32_t monitor_exit; 
	pthread_key_t handle_key;//线程私有变量
	bool profile;	// default is off
};

static struct skynet_node G_NODE;
'''

*环境变量 skynet启动后会启动一个lua虚拟机里把环境变量和配置储存起来。
'''
struct skynet_env {
	struct spinlock lock; // 锁（互斥锁和自旋锁）
	lua_State *L; // lua虚拟机指针
};
配置参数
struct skynet_config {
	int thread;  // 线程数量
	int harbor;  // harbor ID 全网络唯一(master/cslave模式)
	int profile;
	const char * daemon;
	const char * module_path; // C动态库路径
	const char * bootstrap; // 启动程序 默认snlua bootstrap
	const char * logger; // 日志虚拟机名称 默认 logger
	const char * logservice; // 日志服务logger
};
'''
*服务节点管理
'''
struct handle_storage {
	struct rwlock lock; // 锁 读写
	uint32_t harbor; // 节点ID
	uint32_t handle_index; // handle的迭代索引 用于查找下一个可用handle
	int slot_size; // 节点池大小 成倍增长 只增不减
	struct skynet_context ** slot; // 服务节点池哈希表	
	int name_cap;  // 表的大小，成倍增长 只增不减
	int name_count; // 当前name表的大小
	struct handle_name *name; // 服务名称表
};
'''
*全局队列 head,tail分别指向队列的头尾：一级队列 head,tail 头尾坐标 消息入队后头尾相撞说明队列已经满了 扩展队列
'''
消息结构
struct skynet_message {
	uint32_t source; // 源地址
	int session; // 会话id 每次都随机生成
	void * data; // 数据
	size_t sz; // 大小
};
一级队列
struct message_queue {
	struct spinlock lock;  // 锁（互斥锁和自旋锁）
	uint32_t handle; // 服务节点ID
	int cap; // 消息队列长度 动态扩展 只增不减
	int head; // 循环队列头坐标
	int tail; // 循环队列尾坐标
	int release;
	int in_global; // 全局队列标志 用来表示一级队列是否压入了全局队列
	int overload; // 上次队列超阈值的峰值 用于debug提示
	int overload_threshold; // 消息队列长度阈值
	struct skynet_message *queue; // 消息队列
	struct message_queue *next; // 链表后序指针
};
全局队列
struct global_queue {
	struct message_queue *head; // 指向队列的头节点
	struct message_queue *tail; // 指向队列的尾节点
	struct spinlock lock; // 锁（互斥锁和自旋锁）
};
'''
*c库模块
'''
#define MAX_MODULE_TYPE 32

struct skynet_module {
	const char * name; // 名字
	void * module; // 库指针 
	skynet_dl_create create; // create接口
	skynet_dl_init init; // init 接口
	skynet_dl_release release; // release 接口
	skynet_dl_signal signal; // signal 接口
};
struct modules {
	int count; // 加载库的数量
	struct spinlock lock; // 锁（互斥锁和自旋锁）
	const char * path; // 动态库的路径
	struct skynet_module m[MAX_MODULE_TYPE]; // 库表
};
'''
*计数器结构
'''
struct timer {
	struct link_list near[TIME_NEAR]; // 超时近点哈希表
	struct link_list t[4][TIME_LEVEL]; // 超时远点多级哈希表
	struct spinlock lock;  // lock
	uint32_t time; //滴答计数 一滴答百分之一秒
	uint32_t starttime; // 系统启动时间戳单位秒
	uint64_t current; // 系统启动到现场的时间差 百分之一秒
	uint64_t current_point; //上次计数器唤醒与启动时间差 百分之一秒
};
'''
*Socket结构
'''
struct socket_server {
	int recvctrl_fd; // 一对fd,构成管道接收端
	int sendctrl_fd;// 一对fd,构成管道发送端
	int checkctrl; // 管道监控标志
	poll_fd event_fd; // epoll 句柄
	int alloc_id;
	int event_n; // 待处理事件数量
	int event_index; // 事件迭代计数
	struct socket_object_interface soi; 
	struct event ev[MAX_EVENT]; // 监听事件
	struct socket slot[MAX_SOCKET]; // socket槽
	char buffer[MAX_INFO]; // 缓冲区
	uint8_t udpbuffer[MAX_UDP_PACKAGE]; // 缓冲区
	fd_set rfds;//监听套接字集合
};
'''
*监视器结构
'''
struct skynet_monitor {
	int version; // 当前版本号
	int check_version; // 检查的版本号
	uint32_t source; // 消息源地址
	uint32_t destination; // 消息目标地址
};
struct monitor {
	int count; // 线程数量
	struct skynet_monitor ** m; // 监视器数组,对应work线程
	pthread_cond_t cond; // 线程变量
	pthread_mutex_t mutex; // 互斥锁
	int sleep; // 休眠中的线程数量
	int quit; // 退出标志
};
'''










