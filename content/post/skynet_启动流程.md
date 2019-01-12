---
title:      "skynet 学习笔记 - 服务器启动流程"
date:     2019-01-11
categories:
  - skynet
tags:
  - skynet
thumbnailImagePosition: left
thumbnailImage: //d1u9biwaxjngwg.cloudfront.net/highlighted-code-showcase/peak-140.jpg
---

主要介绍下skynet在打包完成后，启动时内部的执行流程是怎样的，从底层C代码到上层lua是如何运转的。首先看Skynet_main.c的代码

<!--more-->
{{< codeblock "Skynet_main.c" "c">}}
int
main(int argc, char *argv[]) {
    // 获取传入的config配置文件
	const char * config_file = NULL ;
	if (argc > 1) {
		config_file = argv[1];
	} else {
		fprintf(stderr, "Need a config file. Please read skynet wiki : https://github.com/cloudwu/skynet/wiki/Config\n"
			"usage: skynet configfilename\n");
		return 1;
	}
	luaS_initshr();
	skynet_globalinit();
	skynet_env_init();
	.
	.//省略部分代码
	.
	lua_close(L);
	// 传入配置参数启动skynet的相关组件和线程
	skynet_start(&config);
	skynet_globalexit();
	luaS_exitshr();
}
{{< /codeblock >}}

上面的init()方法主要是加载配置是数据，以及启动lua的环境和服务，最后会将获取的配置文件传给skynte.start()函数处理，接下来看下该函数
{{< codeblock "Skynet_start.c" "c">}}
    void 
skynet_start(struct skynet_config * config) {
	// register SIGHUP for log file reopen
	struct sigaction sa;
	sa.sa_handler = &handle_hup;
	sa.sa_flags = SA_RESTART;
	sigfillset(&sa.sa_mask);
	sigaction(SIGHUP, &sa, NULL);
	if (config->daemon) {
	    // 初始化守护进程,可在配置文件中配置改参数
		if (daemon_init(config->daemon)) {
			exit(1);
		}
	}
	// 初始化节点，只要在集群中转发节点的消息
	skynet_harbor_init(config->harbor);
	// 初始化句柄，给每个skynet,初始化handler_storage,里面存储了skynet_contxt的指针数组
	skynet_handle_init(config->harbor);
	// 初始化全局消息队列
	skynet_mq_init();
	// 初始化服务动态加载模块，在skynet_module.c中
	skynet_module_init(config->module_path);
	// 初始化定时器模块，在skynet_socket中
	skynet_timer_init();
	// 初始化socket网络模块
	skynet_socket_init();
	skynet_profile_enable(config->profile);
	struct skynet_context *ctx = skynet_context_new(config->logservice, config->logger);
	if (ctx == NULL) {
		fprintf(stderr, "Can't launch %s service\n", config->logservice);
		exit(1);
	}
	skynet_handle_namehandle(skynet_context_handle(ctx), "logger");
    // 加载引导模块，主要在Skynet的配置文件中定义，其默认的是"snlua bootstrap",表示去加载snlua.so库，然后在snlua中会启动
    // bootstrap.lua脚本
	bootstrap(ctx, config->bootstrap);
    // 创建一个montior监视线程
	start(config->thread);

	// harbor_exit may call socket send, so it should exit before socket_free
	skynet_harbor_exit();
	skynet_socket_free();
	if (config->daemon) {
		daemon_exit(config->daemon);
	}
}
{{< /codeblock >}}

在skynet.start()中主要是创建了一些核心模块，module, handle, harbor，网络，以及定时器，消息队列等,

在调用bootstarp时会加载snlua.so模块并创建snlua服务，函数为skyne_context_new(snlua, bootsrap),snlua在初始化时会发送一个消息到到自己模块
对应的skynet_context消息对列中,改消息就是"bootstrap"。回调函数launch_cd处理，在该函数中会把snlua又注销掉,接着执行init_cb()。

{{< codeblock "service_snlua.c" "c">}}
static int
init_cb(struct snlua *l, struct skynet_context *ctx, const char * args, size_t sz) { //args 为 "bootstrap"
	lua_State *L = l->L;
	l->ctx = ctx;
	lua_gc(L, LUA_GCSTOP, 0);
	lua_pushboolean(L, 1);  /* signal for libraries to ignore env. vars. */
	//设置注册表
	lua_setfield(L, LUA_REGISTRYINDEX, "LUA_NOENV");
	luaL_openlibs(L);
	lua_pushlightuserdata(L, ctx);
	//设置元表 {skynet_context}
	lua_setfield(L, LUA_REGISTRYINDEX, "skynet_context");
	
	luaL_requiref(L, "skynet.codecache", codecache , 0);
	lua_pop(L,1);

	const char *path = optstring(ctx, "lua_path","./lualib/?.lua;./lualib/?/init.lua");
	lua_pushstring(L, path);
	//设置lua全局变量
	lua_setglobal(L, "LUA_PATH");
	const char *cpath = optstring(ctx, "lua_cpath","./luaclib/?.so");
	lua_pushstring(L, cpath);
	lua_setglobal(L, "LUA_CPATH");
	const char *service = optstring(ctx, "luaservice", "./service/?.lua");
	lua_pushstring(L, service);
	lua_setglobal(L, "LUA_SERVICE");
	const char *preload = skynet_command(ctx, "GETENV", "preload");
	lua_pushstring(L, preload);
	lua_setglobal(L, "LUA_PRELOAD");

	lua_pushcfunction(L, traceback);
	assert(lua_gettop(L) == 1);
    
	const char * loader = optstring(ctx, "lualoader", "./lualib/loader.lua");
    //启动了loader.lua
	int r = luaL_loadfile(L,loader);
	if (r != LUA_OK) {
		skynet_error(ctx, "Can't load %s : %s", loader, lua_tostring(L, -1));
		report_launcher_error(ctx);
		return 1;
	}
	lua_pushlstring(L, args, sz);
	//将bootstrap传入执行loader
	r = lua_pcall(L,1,0,1);
	if (r != LUA_OK) {
		skynet_error(ctx, "lua loader error : %s", lua_tostring(L, -1));
		report_launcher_error(ctx);
		return 1;
	}
	lua_settop(L,0);
	if (lua_getfield(L, LUA_REGISTRYINDEX, "memlimit") == LUA_TNUMBER) {
		size_t limit = lua_tointeger(L, -1);
		l->mem_limit = limit;
		skynet_error(ctx, "Set memory limit to %.2f M", (float)limit / (1024 * 1024));
		lua_pushnil(L);
		lua_setfield(L, LUA_REGISTRYINDEX, "memlimit");
	}
	lua_pop(L, 1);

	lua_gc(L, LUA_GCRESTART, 0);

	return 0;
}
{{< /codeblock >}}
上面代码就是设置lua虚拟机变量后，就加载执行loader.lua,这时把要加载的文件""Bootstrap"传给他这时候执行流程就转到了lua层了,接下来就可以看
bootstrap.lua的代码了


