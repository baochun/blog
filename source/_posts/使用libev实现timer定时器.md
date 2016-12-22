
---
title: 使用libev实现timer定时器
date: 2016-12-22 12:36:55
tags: [c,linux,libev,timer]
categories: [代码开发]
comments: true
---


libev是轻量级、高性能事件循环/事件模型的网络库，和他很相似的开源有libevent，libev开源代码很晦涩，较难读懂，里面可谓是把宏用到极致，代码很简练。libev提供非常全的功能，包含

```
ev_io
ev_timer
ev_periodic
ev_signal
ev_child
ev_stat
ev_idle
ev_prepare and ev_check
ev_embed
ev_fork
ev_cleanup
ev_async
```

本文主要写一下关于timer如何使用，以及对他进行一个简单封装，这是开发软件常用的方法，原型开发时，能采用开源尽量利用开源资源，但正式开发是否会使用开源或者自己实现一个库是要进行评估的。所以开源相关接口都可以考虑封装，到时候使用其它库时，替换工作量较小。


下面是实现代码：主要调用libev中
```
ev_loop_new
ev_loop
ev_timer_init
ev_timer_stop
ev_timer_start
```

这些接口，来完成定时器功能的封装。


## 头文件

```c
#ifndef _dims_timer_h_
#define _dims_timer_h_

#include <pthread.h>

#include "dims_base.h"
#include "ev.h"
							

typedef void (*dims_timer_cb)(void *arg);

typedef struct dims_watcher_tag
{
	struct	dims_watcher_tag 	*next;
	dims_uint32 	type; 
	dims_timer_cb   timer_cb;
	void			*pvData;
	ev_timer		timer_watcher; 
}dims_watcher_t;

typedef struct dims_timer_tag
{
	dims_uint32 		bInited;
	dims_timer_cb		init_cb;
	void				*pvData;
	dims_watcher_t		*watcher_list;
	pthread_t			thread_id;
	pthread_mutex_t		lock;
	struct ev_loop 		*loop;
}dims_timer_t;

extern dims_uint32 dims_timer_create(dims_timer_t *timer, dims_timer_cb init_cb, void *pvInitData);
extern void dims_timer_init(dims_timer_t *timer, 
		dims_timer_cb	  timer_cb,
		dims_uint32       type,
		void              *pvData,
		double            after,
		double            repeat);
extern void dims_timer_start(dims_timer_t *timer, dims_uint32 type);
extern void dims_timer_stop(dims_timer_t *timer, dims_uint32 type);

#endif /* _dims_timer_h_ */

```

##源文件

```c
/*
 * 对libev timer操作进行封装
 */

#include <ev.h>
#include <stdlib.h>
#include "dims_timer.h"

static void timer_watcher_cb(EV_P_ ev_timer *w, int revents)
{
	dims_watcher_t *watcher = (dims_watcher_t*)w->data;
	if(!watcher) return;

	if(watcher->timer_cb)
		watcher->timer_cb(watcher);
}

static void * timer_loop_entry(void *arg)
{
	dims_timer_t *timer = (dims_timer_t*)arg;
	static ev_timer w;
			
	if(!timer) return 0;
	if(timer->bInited) return 0;
	timer->loop = ev_loop_new(0);
	
	timer->bInited = 1;
	if(timer->init_cb) timer->init_cb(arg);

	ev_timer_init(&w, timer_watcher_cb,1 , 0);
	ev_timer_start(timer->loop, &w);

	ev_loop(timer->loop, 0);
	return 0;
}


dims_uint32 dims_timer_create(dims_timer_t *timer, dims_timer_cb init_cb, void *pvData)
{
	if(!timer) return DIMS_ERROR;

	timer->bInited = 0;
	timer->init_cb = init_cb;
	timer->pvData = pvData;
	timer->watcher_list = DIMS_NULL;
	pthread_mutex_init(&timer->lock, 0);
	if(0 != pthread_create(&(timer->thread_id), NULL, timer_loop_entry, (void*)timer))
	{
		return DIMS_ERROR;
	}

	while(!timer->bInited) sleep(0);	

	return DIMS_OK;
}

static void dims_watcher_del(dims_timer_t *timer, dims_uint32 type)
{
	dims_watcher_t *w = timer->watcher_list;
	dims_watcher_t *prev_w = 0;
	while(w)
	{
		if(w->type == type) break;
		prev_w = w;
		w = w->next;
	}

	if(w)
	{
		if(prev_w) prev_w->next = w->next;
		else timer->watcher_list = 0;
		free(w);
	}
}

static dims_watcher_t * dims_watcher_find(dims_timer_t *timer, dims_uint32 type)
{
	dims_watcher_t *w = timer->watcher_list;
	while(w)
	{
		if(w->type == type) break;
		w = w->next;
	}
	return w;
}

static void dims_watcher_ins(dims_timer_t *timer, dims_watcher_t *w)
{
	w->next = timer->watcher_list;
	timer->watcher_list = w;
	
}

void dims_timer_init(
	dims_timer_t 	*timer,
	dims_timer_cb 	timer_cb, 
	dims_uint32		type,
	void			*pvData,
	double 			after,
	double 			repeat
	)
{
	dims_watcher_t *watcher;

	if(!timer) return ;

	pthread_mutex_lock(&timer->lock);
	watcher = dims_watcher_find(timer, type);
	if(watcher)
	{
		ev_timer_stop(timer->loop, &watcher->timer_watcher);
	}
	else
	{
		watcher = (dims_watcher_t*)malloc(sizeof(dims_watcher_t));
		dims_watcher_ins(timer, watcher);
	}
	
	watcher->timer_cb = timer_cb;
	watcher->type = type;
	watcher->timer_watcher.data = (void*)watcher;
	watcher->pvData = pvData;
	ev_timer_init(&watcher->timer_watcher, timer_watcher_cb, after, repeat);
	
	pthread_mutex_unlock(&timer->lock); 
	return;
}

void dims_timer_start(dims_timer_t *timer, dims_uint32 type)
{
	dims_watcher_t *w;
	if(!timer) return ;

	pthread_mutex_lock(&timer->lock);
	w = dims_watcher_find(timer, type);
	if(w)
	{
		ev_timer_start(timer->loop, &w->timer_watcher);
	}
	pthread_mutex_unlock(&timer->lock);
	return;
}

void dims_timer_stop(dims_timer_t *timer, dims_uint32	type)
{
	if(!timer) return;
	dims_watcher_t *watcher;
	
	pthread_mutex_lock(&timer->lock);
	watcher = dims_watcher_find(timer, type);
	if(watcher) 
	{
		ev_timer_stop(timer->loop, &watcher->timer_watcher);
		dims_watcher_del(timer, type);
	}

	pthread_mutex_unlock(&timer->lock);
	return;
}

```



代码中type字段主要是用来区分定时器类型的，例如一个程序里面需要使用多个定时器，那么用户这里可以type来唯一区分，相同的type认为是同一个定时器。用户在callback调用的时候，也可以根据type来进行不同的处理。
定时器可以重新设置，启动，停止。
> dims_timer_init()

函数为初始化一个定时器


##测试代码：

```c
#include <stdio.h>
#include "dims_timer.h"
#include <unistd.h>


dims_timer_t my_timer;
int cb1, cb2;

void my_init_cb(void *arg)
{
	dims_timer_t *tmr = (dims_timer_t*)arg;
	printf("timer init %s\n", (char*)tmr->pvData);
}

void my_cb(void *arg)
{
	dims_watcher_t *w = arg;
	printf("%s cb1 = %d\n", (char*)w->pvData, cb1);
	cb1++;
}

void my_cb_1(void *arg)
{
	dims_watcher_t *w = arg;
	printf("%s cb2 =%d, type =%d\n", (char*)w->pvData, cb2, w->type);
	cb2++;

	if(cb2 > 20) dims_timer_stop(&my_timer, 1);
}

int main(int argc, char**argv)
{
	dims_timer_create(&my_timer, my_init_cb, "OK!!!");
	
	dims_timer_init(&my_timer, my_cb, 0, "call-1 ", 1, 0.1); //1s秒定时超时，后面每隔0.1s定时超时。		
	dims_timer_init(&my_timer, my_cb_1, 1, "call-2 ", 1.0, 0.5);		

	dims_timer_start(&my_timer, 0);
	dims_timer_start(&my_timer, 1);

	sleep(2);

	dims_timer_stop(&my_timer, 0);

	sleep(8);
	dims_timer_stop(&my_timer, 1);

	sleep(20);	
	return 0;
}

```

##参考文档

libev文档介绍：http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#code_ev_timer_code_relative_and_opti

libev官方网站：http://software.schmorp.de/pkg/libev.html
