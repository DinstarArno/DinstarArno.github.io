---
layout: post
title: skynet启动流程(一)
date: 2018-12-17
categories: blog
tags: [skynet启动流程]
description: 文章金句。
---
* skynet基础模块关系图
  ![图](https://dinstararno.github.io/img/keynet.png)
* skynet启动
  * 配置初始化  
  	```
		const char * config_file = NULL ;
		if (argc > 1) {
			config_file = argv[1]; // ./skynet config 启动传入的配置文件(lua根式)
		} else {
			fprintf(stderr, "Need a config file. Please read skynet wiki : https://github.com/cloudwu/skynet/wiki/Config\n"
				"usage: skynet configfilename\n");
			return 1;
		}
		//启动lua虚拟机存储配置
		luaS_initshr();
		skynet_globalinit();
		skynet_env_init();
	
		sigign();
	
		struct skynet_config config;
	
		struct lua_State *L = luaL_newstate();
		luaL_openlibs(L);	// link lua lib
	
		int err =  luaL_loadbufferx(L, load_config, strlen(load_config), "=[skynet config]", "t");
		assert(err == LUA_OK);
		lua_pushstring(L, config_file);
	
		err = lua_pcall(L, 1, 1, 0);
		if (err) {
			fprintf(stderr,"%s\n",lua_tostring(L,-1));
			lua_close(L);
			return 1;
		}
		_init_env(L);
		//检查配置
		config.thread =  optint("thread",8); //工作线程数量 默认8
		config.module_path = optstring("cpath","./cservice/?.so"); //c库路径 默认./cservice/?.so
		config.harbor = optint("harbor", 1); //harbor ID 默认1 master/cslave模式下有效
		config.bootstrap = optstring("bootstrap","snlua bootstrap"); //lua引导程序 默认 snlua bootstrap
		config.daemon = optstring("daemon", NULL);
		config.logger = optstring("logger", NULL);
		config.logservice = optstring("logservice", "logger");
		config.profile = optboolean("profile", 1);
  	```
  * 资源初始化
	```
		文件skynet_start.c中：

		skynet_harbor_init(config->harbor); // 生成barbor handle,高八位表示,所以全网最多255个barbor节点
		skynet_handle_init(config->harbor); // 初始化handle
		skynet_mq_init(); // 初始化全局队列
		skynet_module_init(config->module_path);
		skynet_timer_init();
		skynet_socket_init();
		skynet_profile_enable(config->profile);
		...	
		start(config->thread);//启动线程
	```
  * 模块启动
	```	
		文件skynet_start.c中：

		create_thread(&pid[0], thread_monitor, m);  //monitor模块
		create_thread(&pid[1], thread_timer, m);    //timer模块
		create_thread(&pid[2], thread_socket, m);   //网络模块
		//启动工作线程 数量为thread
		for (i=0;i<thread;i++) {
			wp[i].m = m;
			wp[i].id = i;
			if (i < sizeof(weight)/sizeof(weight[0])) {
				wp[i].weight= weight[i];
			} else {
				wp[i].weight = 0;
			}
			create_thread(&pid[i+3], thread_worker, &wp[i]);
		}
	```
* 本文仅仅初步描述skynet启动配置初始化到工作线程创建,其他具体过程后续描述。