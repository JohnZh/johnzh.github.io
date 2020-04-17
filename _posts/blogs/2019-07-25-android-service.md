---
layout: post_no_cmt
title:  "Android Service 启动方式，生命周期，参数说明，混合启动"
date:   2019-07-25 00:06:00 +0800
categories: [android]
---

# 两种启动方式

## 生命周期
标准模式 startService()：

1. onCreate()
2. onStartCommand() 
3. Service 被停止或者自主停止
4. onDestroy()
5. Service 关闭

绑定模式 bindService()：

1. onCreate()
2. onBind()
3. Client 调用 unbindService()
4. onUnbind()
5. onDestroy()
6. Service 关闭 

## 使用上的选择
> Each call to startService() results in significant work done by the system to manage service lifecycle surrounding the processing of the intent, which can take multiple milliseconds of CPU time. Due to this cost, startService() should not be used for frequent intent delivery to a service, and only for scheduling significant work. Use bound services for high frequency calls.

每次调用 startService() 会导致系统为了管理 Service 的生命周期而围绕着 Intent 的处理做重要工作，这会花费多个毫秒级的 CPU 时间。由于此开销，startService() 不应该用于 Intent 到 Service 的频繁传递，而只是用于执行重要工作。高频繁的调用使用 Services 绑定。

简单的说，Service 和 Clients 之间存在频繁调用的应该使用绑定模式

# startSerive 的方式
- 无论调用多少次 startService()，stopService()/stopSelf() 只需要一次
- 除了第一次 startService() 会进行 onCreate()，onStartCommand() 回调，后面都只进行 onStartCommand()  回调

## onStartCommand() 

调用 Context#startService 时，onCreate() 只会第一次调用，后续都只会调用 onStartCommand()。这也意味着不论多少次调用 startService，只有一次 stopService 就够了

调用 Context#startService 方式启动 Service 时，如果 Android 系统面临内存匮乏，可能会销毁当前运行的Service，待内存充足时可以重建 Service。Service 被系统强制销毁并再次重建的行为依赖于 Service 的 onStartCommand() 方法的返回值

onStartCommand (Intent intent, int flags, int startId): int

- intent：startService 启动时候传入的 intent，如果是被系统 restart 的情况，intent 可能是 null 的
- flags：启动服务请求的额外数据，值为 0，或者 START_FLAG_REDELIVERY 或START_FLAG_RETRY 的组合
- startId：表示用于启动服务的特定请求的唯一整数。用于 stopSelfResult(int)

### flags 值为 
这个时候 intent 是之前传递进来的 intent，因为 Service#onStartCommand 返回值是 START_REDELIVER_INTENT，且 Service 在调用 stopSelf(int) 之前被杀了，此时的调用属于系统重建 Service

### flags 值为 START_FLAG_RETRY
这个情况是发生在 Service#onStartCommand 方法没有被调用，或者被返回的这种异常情况，这个时候，intent 是一次重试

### startId 作用
同样是 stop service 这个 startid 作用是什么？startid 为最近一次 start 请求，当然这个值也许不一定是最新的，因为有时候一个 start 请求也许还在发生，且并为更新到这个方法。这个时候，调用对应的 stopSelfResult(int) 方法就会更安全，也能避免停止服务，因为这个不是最新的 startId 请求

## onStartCommand() 返回值
### START_STICKY
> if this service's process is killed while it is started (after returning from onStartCommand(Intent, int, int)), then leave it in the started state but don't retain this delivered intent. Later the system will try to re-create the service. Because it is in the started state, it will guarantee to call onStartCommand(Intent, int, int) after creating the new service instance; if there are not any pending start commands to be delivered to the service, it will be called with a null intent object, so you must take care to check for this.

> This mode makes sense for things that will be explicitly started and stopped to run for arbitrary periods of time, such as a service performing background music playback.

如果 Service 的进程在 started（在 onStartCommand(Intent, int, int) 返回之后） 之后被系统杀死，系统会保留 started 状态，但保存之前 onStartCommand 方法传入的 intent。稍后， 系统会尝试重新创建该 Service，因为在 started 状态下，创建这个新的 Service 实例后会保证调用 onStartCommand 方法，如果没有任何 pending start 命令需要传递，那么 onStartCommand 调用的时候，intent 会是一个 null 的对象，因此你必须注意这里。

这种模式适用与那种会被明确启动运行一段时间以及明确关闭一段时间的场景，例如一个Service 用于执行背景音乐的播放

> START_STICKY_COMPATIBILITY 为 START_STICKY 的兼容版本，基本上用不到

### START_NOT_STICKY
> if this service's process is killed while it is started (after returning from onStartCommand(Intent, int, int)), and there are no new start intents to deliver to it, then take the service out of the started state and don't recreate until a future explicit call to Context.startService(Intent). The service will not receive a onStartCommand(Intent, int, int) call with a null Intent because it will not be re-started if there are no pending Intents to deliver.

> This mode makes sense for things that want to do some work as a result of being started, but can be stopped when under memory pressure and will explicit start themselves again later to do more work. An example of such a service would be one that polls for data from a server: it could schedule an alarm to poll every N minutes by having the alarm start its service. When its onStartCommand(Intent, int, int) is called from the alarm, it schedules a new alarm for N minutes later, and spawns a thread to do its networking. If its process is killed while doing that check, the service will not be restarted until the alarm goes off.

如果 Service 的进程在 started（在 onStartCommand(Intent, int, int) 返回之后） 之后被系统杀死，并且没有新的启动 intent 传递，那此 Service 会失去 started 状态且不会重建，直到未来明确调用 Context.startService(intent)。这个 Service 不会收到 null intent 的 onStartCommand(Intent, int, int) 调用，因为如果没有 pending intent 它就不会被重启

这种模式适用于想要启动 Service 做一些工作，Service 在内存有压力的情况下被停止，之后被它们自己再次启动做更多的工作的情况。从服务端轮询数据的例子：执行一个 alarm，每 N 分钟通过 startService 轮询一次。当 Service 的 onStartCommand(Intent, int, int) 经由 alarm 唤起，Service 会执行一个新的 alarm 用于 N 分钟之后，同时产生一个线程用于网络任务。如果它的线程在做检查的时候被杀，Service 不会重启，直到 alarm 响起

### START_REDELIVER_INTENT
> if this service's process is killed while it is started (after returning from onStartCommand(Intent, int, int)), then it will be scheduled for a restart and the last delivered Intent re-delivered to it again via onStartCommand(Intent, int, int). This Intent will remain scheduled for redelivery until the service calls stopSelf(int) with the start ID provided to onStartCommand(Intent, int, int). The service will not receive a onStartCommand(Intent, int, int) call with a null Intent because it will will only be re-started if it is not finished processing all Intents sent to it (and any such pending events will be delivered at the point of restart).

如果 Service 的进程在 started（在 onStartCommand(Intent, int, int) 返回之后） 之后被系统杀死，然后就会周期性执行重启任务，最后传入的 intent 会再次通过 onStartCommand(Intent, int, int) 传入。最后传入的 intent 会被保留，直到 Service 使用 onStartCommand(intent, int, int) 提供的 startId 调用 stopSelf(int)。这个 Service 不会收到一个 null intent 的 onStartCommand(Intent, int, int) 回调，因为它只会因还有未完成的处理的 intent 而被重启 (任何这样的 pending 事件都会在 restart 的时刻被传入)

# bindService 的方式
- 多次 bindService() ，Service#onBind 不会被多次执行
- 同一个 Activity 多次 bindService()，除了第一次，后面的 bindSerice 不会触发 onBind()、onServiceConnected()
- 不同 Activity 进行 bindService()，第一次触发 onBind()、onServiceConnected() ，后面的 bindSerice 只触发 onServiceConnected() 
	-  **注意：同一个Activity bindService() 之后 unbindService()，再 bindService() 同样不会再触发 onBind()，只会触发  onServiceConnected()**

- 全部 Clients 从 Service 断开，才会调用 onUnbind()，且 onDestroy()
- Client 如果销毁，会自动解绑 Service ，也可以明确 unbindService()
	- **注意：Activity 里面尽量明确 unbindService()，避免 ServiceConnectionLeaked**

- onServiceDisconnected() 在连接正常关闭的情况下是不会被调用的，只在 Service 被破坏或被杀的时候
- **onUnbind() 返回值：默认返回 false ，Client 先全断开后再绑定的情况，除了无法将 onRebind() 触发外，onUnbind() 方法也只会触发一次**。因此，如果需要 onUnbind() 每次全部解绑都会被调用，这个方法返回值需为 true

# 混合启动
## 先 startService，后 bindService
先 startService()，再 bindService() 关键方法调用顺序：

1. onCreate() 
2. onStartCommand()
3. onBind()
4. Activity: ServiceConnection.onServiceConnected()

解绑和关闭分两种情况：

第一种：

1. 先 unBindService() 触发 onUnbind()
2. 后 stopService() 触发 onDestroy()

第二种：

1. 先 stopService() 无触发
2. 后 unBindService() 触发 onUnbind() -> onDestroy()

## 先 bindService，后 startService
先 bindService()，再 startService() 关键方法调用顺序：

1. onCreate()
2. onBind()
3. Activity: ServiceConnection.onServiceConnected()
4. onStartCommand()

解绑和关闭的情况和之前相同

## 注意点
但凡使用 startService() 启动的 Service，无论是否解绑了全部 Clients，都应该再使用 stopService()/stopSelf() 关闭Clients