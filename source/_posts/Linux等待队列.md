---


title: Linux等待队列
date: 2021-02-26 09:39:26
categories: 
- Linux系统
tags:
- Linux
---

在软件开发中任务经常由于某种条件没有得到满足而不得不进入睡眠状态，然后等待条件得到满足的时候再继续运行，进入运行状态。这种需求需要等待队列机制的支持。Linux中提供了等待队列的机制，该机制在内核中应用很广泛。

它是以双循环链表为基础数据结构，与进程的休眠唤醒机制紧密相联，是实现异步事件通知、跨进程通信、同步资源访问等技术的底层技术支撑。

它有两种数据结构：等待队列头 （wait_queue_head_t）和等待队列项（wait_queue_t）。等待队列头和等待队列项中都包含一个list_head类型的域作为”连接件”。它通过一个双链表和把等待task的头，和等待的进程列表链接起来。

## 数据结构

等待队列结构如下，因为每个等待队列都可以再中断时被修改，因此，在操作等待队列之前必须获得一个自旋锁。

```c
struct __wait_queue_head {
   spinlock_t      lock;
   struct list_head    task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;
```

可通过宏`DECLARE_WAIT_QUEUE_HEAD(name)`来创建类型为wait_queue_head_t的等待队列头name。

```c
#define DECLARE_WAIT_QUEUE_HEAD(name) \
    struct wait_queue_head name = __WAIT_QUEUE_HEAD_INITIALIZER(name)
    
#define __WAIT_QUEUE_HEAD_INITIALIZER(name) {                    \
    .lock        = __SPIN_LOCK_UNLOCKED(name.lock),            \
    .head        = { &(name).head, &(name).head } }
```

等待队列是通过task_list双链表来实现，其数据成员是如下：

```c
struct __wait_queue {
	unsigned int		flags;
	void			*private;  //指向等待队列的进程task_struct
	wait_queue_func_t	func;
	struct list_head	task_list;
};
typedef struct __wait_queue wait_queue_t;
```

可通过宏`DECLARE_WAITQUEUE(name, tsk) `来创建类型为wait_queue_t的等待队列项name，并将tsk赋值给成员变量private， default_wake_function赋值给成员变量func。

```c
#define DECLARE_WAITQUEUE(name, tsk)                        \
    struct wait_queue_entry name = __WAITQUEUE_INITIALIZER(name, tsk)

#define __WAITQUEUE_INITIALIZER(name, tsk) {                    \
    .private    = tsk,                            \
    .func        = default_wake_function,                \
    .entry        = { NULL, NULL } }
```

示意图：



![](Linux等待队列/wait_queue.png)

## 初始化等待队列元素

1. 静态初始化

   ```
   #define DEFINE_WAIT_FUNC(name, function)				\
   	wait_queue_t name = {						\
   		.private	= current,				\
   		.func		= function,				\
   		.task_list	= LIST_HEAD_INIT((name).task_list),	\
   	}
   #define DEFINE_WAIT(name) DEFINE_WAIT_FUNC(name, autoremove_wake_function)
   ```

2. 动态初始化

   ```c
   static inline void init_waitqueue_entry(wait_queue_t *q, struct task_struct *p)
   {
   	q->flags	= 0;
   	q->private	= p;
   	q->func		= default_wake_function;
   }
   ```

   

## 增加删除

### add_wait_queue

```c
void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)
{
    unsigned long flags;
    wait->flags &= ~WQ_FLAG_EXCLUSIVE;
    spin_lock_irqsave(&q->lock, flags);
    __add_wait_queue(q, wait);  //挂到队列头
    spin_unlock_irqrestore(&q->lock, flags);
}

static inline void __add_wait_queue(wait_queue_head_t *head, wait_queue_t *new)
{
    list_add(&new->task_list, &head->task_list);
}
```

该方法的功能是将wait等待队列项 挂到等待队列头q中。

### remove_wait_queue

```c
void remove_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)
{
    unsigned long flags;
    spin_lock_irqsave(&q->lock, flags);
    __remove_wait_queue(q, wait);
    spin_unlock_irqrestore(&q->lock, flags);
}

static inline void __remove_wait_queue(wait_queue_head_t *head, wait_queue_t *old)
{
    list_del(&old->task_list);
}
```

该方法主要功能是将wait等待队列项 从等待队列头q中移除。

## 休眠唤醒

### wait_event （进程休眠）

```
//condition ： a C expression for the event to wait for
#define wait_event(wq, condition)                    \
do {                                    \
    if (condition)                            \
        break;                            \
    __wait_event(wq, condition);                    \
} while (0)

#define __wait_event(wq, condition)                    \
    (void)___wait_event(wq, condition, TASK_UNINTERRUPTIBLE, 0, 0, schedule())
```

####___wait_event

```c
___wait_event(wq, condition, state, exclusive, ret, cmd){  
    wait_queue_t __wait;                    
    INIT_LIST_HEAD(&__wait.task_list);                
    for (;;) {
        //当检测进程是否有待处理信号则返回值__int不为0,见下面解析
        long __int = prepare_to_wait_event(&wq, &__wait, state);
        if (condition)  //当满足条件，则跳出循环                    
            break;                        
                                    
        //当有待处理信号且进程处于可中断状态(TASK_INTERRUPTIBLE或TASK_KILLABLE))，则跳出循环
        if (___wait_is_interruptible(state) && __int) {        
            __ret = __int;                    
            break;                      
        }                            
        cmd; //schedule()，进入睡眠，从进程就绪队列选择一个高优先级进程来代替当前进程运行                       
    }                                
    finish_wait(&wq, &__wait);  //调用函数finish_wait()将进程状态设置为TASK_RUNNING，并从等待队列的链表中移除对应的成员             
}
```

#### prepare_to_wait_event

```c
long prepare_to_wait_event(wait_queue_head_t *q, wait_queue_t *wait, int state)
{
    unsigned long flags;
    if (signal_pending_state(state, current)) //信号检测
        return -ERESTARTSYS;

    wait->private = current;
    wait->func = autoremove_wake_function; //设置func唤醒函数

    spin_lock_irqsave(&q->lock, flags);
    if (list_empty(&wait->task_list)) {  //当wait不在队列q，则加入其中，防止无法唤醒
        if (wait->flags & WQ_FLAG_EXCLUSIVE)
            __add_wait_queue_tail(q, wait);
        else
            __add_wait_queue(q, wait);
    }
    set_current_state(state);  //设置进程状态
    spin_unlock_irqrestore(&q->lock, flags);

    return 0;
}
```

```c
int autoremove_wake_function(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
    int ret = default_wake_function(wait, mode, sync, key); //唤醒函数
    if (ret)
        list_del_init(&wait->task_list); //从列表中移除wait
    return ret;
}
```

wait_event(wq, condition)：进入睡眠状态直到condition为true，在等待期进程状态为TASK_UNINTERRUPTIBLE。对应的唤醒方法是wake_up()，当等待队列wq被唤醒时会执行如下两个检测：

- 检查condition是否为true，满足条件，则跳出循环。
- 检测该进程task的成员thread_info->flags是否被设置TIF_SIGPENDING，被设置则说明有待处理的信号，则跳出循环。

### wake_up （进程唤醒）

```c
#define wake_up(x)            __wake_up(x, TASK_NORMAL, 1, NULL)


void __wake_up(wait_queue_head_t *q, unsigned int mode,
            int nr_exclusive, void *key)
{
    unsigned long flags;
    spin_lock_irqsave(&q->lock, flags);
    //核心方法
    __wake_up_common(q, mode, nr_exclusive, 0, key);
    spin_unlock_irqrestore(&q->lock, flags);
}


static void __wake_up_common(wait_queue_head_t *q, unsigned int mode,
            int nr_exclusive, int wake_flags, void *key)
{
    wait_queue_t *curr, *next;

    list_for_each_entry_safe(curr, next, &q->task_list, task_list) {
        unsigned flags = curr->flags;
        //调用prepare_to_wait_event中设置的唤醒函数
        if (curr->func(curr, mode, wake_flags, key) &&
                (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
            break;
    }
}
```

其中：q是等待队列，mode指定进程的状态，用于控制唤醒进程的条件，nr_exclusive表示将要唤醒的设置了WQ_FLAG_EXCLUSIVE标志的进程的数目。 

然后扫描链表，调用func唤醒函数，直至没有更多的进程被唤醒，或者被唤醒的的独占进程数目已经达到规定数目。

从上面可知，真正调用的唤醒函数为default_wake_function

#### default_wake_function

```c
int default_wake_function(wait_queue_t *curr, unsigned mode, int wake_flags,
			  void *key)
{
	return try_to_wake_up(curr->private, mode, wake_flags);
}
```

#### try_to_wake_up

```c
static int try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
{
    unsigned long flags;
    int cpu, src_cpu, success = 0;

    bool freq_notif_allowed = !(wake_flags & WF_NO_NOTIFIER);
    bool check_group = false;
    wake_flags &= ~WF_NO_NOTIFIER;

    smp_mb__before_spinlock();
    raw_spin_lock_irqsave(&p->pi_lock, flags); //关闭本地中断
    src_cpu = cpu = task_cpu(p);

    //如果当前进程状态不属于可唤醒状态集，则无法唤醒该进程
    //wake_up()传递过来的TASK_NORMAL等于(TASK_INTERRUPTIBLE | TASK_UNINTERRUPTIBLE)
    if (!(p->state & state)) 
        goto out;

    success = 1; 
    smp_rmb();
    if (p->on_rq && ttwu_remote(p, wake_flags)) //当前进程已处于rq运行队列，则无需唤醒
        goto stat;
    ...
    
    ttwu_queue(p, cpu); 
stat:
    ttwu_stat(p, cpu, wake_flags);
out:
    raw_spin_unlock_irqrestore(&p->pi_lock, flags); //恢复本地中断
    ...
    return success;
}
```

#### ttwu_queue

```c
static void ttwu_queue(struct task_struct *p, int cpu)
{
	struct rq *rq = cpu_rq(cpu); // 获取当前进程的运行队列
	raw_spin_lock(&rq->lock);
	lockdep_pin_lock(&rq->lock);
	ttwu_do_activate(rq, p, 0); //
	lockdep_unpin_lock(&rq->lock);
	raw_spin_unlock(&rq->lock);
}
//ttwu ： try to wake up 缩写
```

#### ttwu_do_activate

```c
static void ttwu_do_activate(struct rq *rq, struct task_struct *p, int wake_flags)
{
	ttwu_activate(rq, p, ENQUEUE_WAKEUP | ENQUEUE_WAKING);
	ttwu_do_wakeup(rq, p, wake_flags);
}

static inline void ttwu_activate(struct rq *rq, struct task_struct *p, int en_flags)
{
	activate_task(rq, p, en_flags); //将进程task加入rq队列
	p->on_rq = TASK_ON_RQ_QUEUED;

	if (p->flags & PF_WQ_WORKER)
		wq_worker_waking_up(p, cpu_of(rq)); //worker正在唤醒中，则通知工作队列
}

static void ttwu_do_wakeup(struct rq *rq, struct task_struct *p, int wake_flags)
{
	check_preempt_curr(rq, p, wake_flags);
	p->state = TASK_RUNNING; //标记该进程为TASK_RUNNING状态
    ...
}
```



## 总结

**休眠与唤醒流程：**

1. 进程A调用wait_event(wq, condition)就是向等待队列头中添加等待队列项wait_queue_t，该该等待队列项中的成员变量private记录当前进程，其成员变量func记录唤醒回调函数，然后调用schedule()使当前进程进入休眠状态。
2. 进程B调用wake_up(wq)会遍历整个等待列表wq中的每一项wait_queue_t，依次调用每一项的唤醒函数try_to_wake_up()。这个过程会将private记录的进程加入rq运行队列，并设置进程状态为TASK_RUNNING。
3. 进程A被唤醒后只执行如下检测：
   - 检查condition是否为true，满足条件则跳出循环，再把wait_queue_t从wq队列中移除；
   - 检测该进程task的成员thread_info->flags是否被设置TIF_SIGPENDING，被设置则说明有待处理的信号，则跳出循环，再把wait_queue_t从wq队列中移除；
   - 否则，继续调用schedule()再次进入休眠等待状态，如果wait_queue_t不在wq队列，则再次加入wq队列。



参考：

http://gityuan.com/2018/12/02/linux-wait-queue/

https://blog.csdn.net/younger_china/article/details/7176851