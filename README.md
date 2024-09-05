# libco源码分析

libco与coroutine的异同:

- 协程上下文切换性能更好(采用汇编)
- 协程在IO阻塞时可以自动切换, 包括gethostbyname, mysqlclient等
- 协程可以嵌套创建, 即一个协程内部可以再创建一个协程
- 提供了超时管理, 以及一套类pthread的接口用来协程间通信
- 可选共享栈和独立栈模式, 默认独立栈模式(128K)
- libco利用co_create创建的协程, 需要自行调用co_release释放.
- hook了常见系统调用

libco的特性:

- 非对称协程, 通过`co_resume()`拉起一个协程, 通过`co_yield_ct()`将控制权还给调用方
  - 初始化协程的运行环境时，会隐式的创建一个协程，我们称它为主协程，我们一般会在主协程里运行 epoll 事件循环，调用链头部协程挂起时控制权就会交到主协程手里
- 有栈协程, 两种栈模式
  - 独立栈: 每个协程有自己的栈, 默认128K
    - 对独立栈浪费内存的说法保持质疑，malloc 申请的是虚拟地址空间，只有读写时才会引发缺页中断，从而分配物理内存，因此内存申请不等于内存分配，内存只在实际用到的时候才会被分配
  - 共享栈: 一组协程共享一个运行栈, 栈由运行中的协程持有, 当协程被换下时, 将它使用掉的栈内存保存到一个缓冲区里, 等再次换上它运行时, 再把之前保存下来的栈拷贝到共享的栈空间里
    - 共享栈的缺点: 栈地址不可以跨协程使用;  每次协程切换都要拷贝, 消耗CPU

## 基本数据结构

```c
// 协程控制块 CCB
struct stCoRoutine_t {
  	// 表示协程运行的环境，都会指向当前线程的一个 tls 变量 gCoEnvPerThread
    // 因此可以将它理解为 libco 中代表线程的实体
    stCoRoutineEnv_t *env;
	
    // 入口函数和参数(函数指针)
    pfn_co_routine_t pfn;  
    void *arg;  
    // 上下文, 寄存器,栈指针等
    coctx_t ctx;           

    // 协程状态
    char cStart;
    char cEnd;
    // 是否主协程
    char cIsMain;
    // 是否启用hook
    char cEnableSysHook;
    // 是否启用共享栈
    char cIsShareStack;

    // 保存环境变量
    void *pvEnv;
    // 协程栈
	stStackMem_t *stack_mem;

    // 共享栈使用
    char *stack_sp; // 协程在共享栈中已经使用的栈顶
    unsigned int save_size; // save_buffer的长度
    char *save_buffer;// 协程挂起时, 栈的内容会暂存到save_buffer里
	
    // 协程局部存储数据
    stCoSpec_t aSpec[1024];
};
// 协程属性, 默认不使用共享栈, 独立栈大小128K
struct stCoRoutineAttr_t
{
	int stack_size;
	stShareStack_t*  share_stack;
	stCoRoutineAttr_t()
	{
		stack_size = 128 * 1024;
		share_stack = NULL;
	}
}__attribute__ ((packed));
// 协程运行环境, 类似调度器
// 每一个协程都有一个指向其运行环境的指针
struct stCoRoutineEnv_t {
  	// 维护了一份协程的调用关系链
    // 最后一位是当前运行的协程, 前一位是当前协程的父协程(即resume该协程的协程)
    stCoRoutine_t *pCallStack[128];
    // 当前正在运行的协程下标 + 1, 即当前调用栈长度
    int iCallStackSize;
    // 保存事件循环和定时器相关信息(调度器)
    stCoEpoll_t *pEpoll;
	
    // 共享栈使用
    stCoRoutine_t *pending_co;
    stCoRoutine_t *occupy_co;
};
// 协程(共享)栈结构
struct stStackMem_t
{
    // 当前正在使用该栈的协程
	stCoRoutine_t* occupy_co;
	int stack_size; // 栈大小
	char* stack_bp; //stack_buffer + stack_size 栈底base pointer
	char* stack_buffer; // 栈顶
};
// 共享栈数组
struct stShareStack_t
{
	unsigned int alloc_idx; // 目前正在使用的共享栈的idx
	int stack_size; // 共享栈大小, 即stStackMem_t->stack_size
	int count; // 共享栈的个数
	stStackMem_t** stack_array; // 指针数组,元素为stStackMem_t*
};
```

每个线程都有一份 stCoRoutineEnv_t 对象，在线程第一次创建协程时被自动创建，同时也会创建主协程，并将指向主协程的指针放到 `pCallStack[0]` 里

## 协程接口函数

### 协程创建

```cpp
int co_create( stCoRoutine_t **ppco,const stCoRoutineAttr_t *attr,
              pfn_co_routine_t pfn,void *arg ) 
{
    // 若环境不存在, 初始化线程局部变量gCoEnvPerThread, 其为stCoRoutineEnv_t类型
	if( !co_get_curr_thread_env() ) 
	{
		co_init_curr_thread_env();
	}
	stCoRoutine_t *co = co_create_env( co_get_curr_thread_env(), attr, pfn,arg );
	*ppco = co;
	return 0;
}
struct stCoRoutine_t *co_create_env( stCoRoutineEnv_t * env, 
                                    const stCoRoutineAttr_t* attr,
                                    pfn_co_routine_t pfn,void *arg )
{
	// 初始化属性并赋值
	stCoRoutineAttr_t at;
	if( attr )
	{
		memcpy( &at,attr,sizeof(at) );
	}
	if( at.stack_size <= 0 )
	{
		at.stack_size = 128 * 1024;
	}
	else if( at.stack_size > 1024 * 1024 * 8 )
	{
		at.stack_size = 1024 * 1024 * 8;
	}

	if( at.stack_size & 0xFFF ) 
	{
		at.stack_size &= ~0xFFF;
		at.stack_size += 0x1000;
	}

	stCoRoutine_t *lp = (stCoRoutine_t*)malloc( sizeof(stCoRoutine_t) );
	
	memset( lp,0,(long)(sizeof(stCoRoutine_t))); 


	lp->env = env;
	lp->pfn = pfn;
	lp->arg = arg;
	
	stStackMem_t* stack_mem = NULL;
    // 若为共享栈, 在共享栈中获取内存
	if( at.share_stack )
	{
		stack_mem = co_get_stackmem( at.share_stack);
		at.stack_size = at.share_stack->stack_size;
	}
    // 否则分配内存
	else
	{
		stack_mem = co_alloc_stackmem(at.stack_size);
	}
	lp->stack_mem = stack_mem;
	// 设置上下文
	lp->ctx.ss_sp = stack_mem->stack_buffer;
	lp->ctx.ss_size = at.stack_size;

	lp->cStart = 0;
	lp->cEnd = 0;
	lp->cIsMain = 0;
	lp->cEnableSysHook = 0;
	lp->cIsShareStack = at.share_stack != NULL;

	lp->save_size = 0;
	lp->save_buffer = NULL;

	return lp;
}
// 
static stStackMem_t* co_get_stackmem(stShareStack_t* share_stack)
{
	if (!share_stack)
	{
		return NULL;
	}
    // 轮询
	int idx = share_stack->alloc_idx % share_stack->count;
	share_stack->alloc_idx++;

	return share_stack->stack_array[idx];
}
```

### 协程挂起

```cpp
// 主动将当前运行的协程挂起，并恢复到上一层的协程
void co_yield_env( stCoRoutineEnv_t *env )
{
	// 这里直接取了iCallStackSize - 2，那么万一icallstacksize < 2呢？
	// 所以这里实际上有个约束，就是co_yield之前必须先co_resume, 这样就不会造成这个问题了

	// last就是 找到上次调用co_resume(curr)的协程
	stCoRoutine_t *last = env->pCallStack[ env->iCallStackSize - 2 ];

	// 当前栈
	stCoRoutine_t *curr = env->pCallStack[ env->iCallStackSize - 1 ];

    // 当前栈变成上一次调用的该协程的协程的栈
	env->iCallStackSize--;

	// 把上下文当前的存储到curr中，并切换成last的上下文
	co_swap( curr, last);
}

// 下面两个函数是对co_yield_env的封装
// 挂起当前线程主协程
void co_yield_ct() // ct = current thread
{
	co_yield_env( co_get_curr_thread_env() );
}

// 挂起传入的协程
void co_yield( stCoRoutine_t *co )
{
	co_yield_env( co->env );
}
```

### 协程恢复

```cpp
// 继续运行协程
void co_resume( stCoRoutine_t *co )
{
	stCoRoutineEnv_t *env = co->env;

	// 找到当前运行的协程, 从数组最后一位拿出当前运行的协程，如果目前没有协程，那就是主线程
	stCoRoutine_t *lpCurrRoutine = env->pCallStack[ env->iCallStackSize - 1 ];

	if( !co->cStart )
	{
		// 如果当前协程还没有开始运行，为其构建上下文
		coctx_make( &co->ctx,(coctx_pfn_t)CoRoutineFunc,co, 0 );
		co->cStart = 1;
	}

	// 将指定协程放入线程的协程队列末尾
	env->pCallStack[ env->iCallStackSize++ ] = co;
	
	// 将当前运行的上下文保存到lpCurrRoutine中，同时将协程co的上下文替换进去
	// 执行完这一句，当前的运行环境就被替换为 co 了
	co_swap( lpCurrRoutine, co );
}
```

### 协程上下文切换

```cpp
// 处理协程切换逻辑, 真正切换上下文的是coctx_swap()汇编
// 分为独立栈切换逻辑和共享栈切换逻辑
void co_swap(stCoRoutine_t* curr, stCoRoutine_t* pending_co)
{
 	stCoRoutineEnv_t* env = co_get_curr_thread_env();

	//get curr stack pointer(sp)
	//这里非常重要!!!： 这个c变量的实现，作用是为了找到目前的栈顶，因为c变量是最后一个放入栈中的内容。
	char c;
	curr->stack_sp= &c;

	if (!pending_co->cIsShareStack)
	{  
		// 如果没有采用共享栈，清空pending_co和occupy_co
		env->pending_co = NULL;
		env->occupy_co = NULL;
	}
	else 
	{   
		// 如果采用了共享栈
		env->pending_co = pending_co; 
		
		// get last occupy co on the same stack mem
		// occupy_co指的是，和pending_co共同使用一个共享栈的协程
		// 把它取出来是为了先把occupy_co的内存保存起来
		stCoRoutine_t* occupy_co = pending_co->stack_mem->occupy_co;
		
		// set pending co to occupy thest stack mem;
		// 将该共享栈的占用者改为pending_co
		pending_co->stack_mem->occupy_co = pending_co;

		env->occupy_co = occupy_co;
		
		if (occupy_co && occupy_co != pending_co)
		{  
			// 如果上一个使用协程不为空, 则需要把它的栈内容保存起来。
			save_stack_buffer(occupy_co);
		}
	}

	// swap context asm
	coctx_swap(&(curr->ctx),&(pending_co->ctx) );

	// 这个地方很绕，上一步coctx_swap会进入到pending_co的协程环境中运行
	// 到这一步，已经yield回此协程了，才会执行下面的语句
	// 而yield回此协程之前，env->pending_co会被上一层协程设置为此协程
	// 因此可以顺利执行: 将之前保存起来的栈内容，恢复到运行栈上

	// stack buffer may be overwrite, so get again;
	stCoRoutineEnv_t* curr_env = co_get_curr_thread_env();
	stCoRoutine_t* update_occupy_co =  curr_env->occupy_co;
	stCoRoutine_t* update_pending_co = curr_env->pending_co;
	
	// 将栈的内容恢复，如果不是共享栈的话，每个协程都有自己独立的栈空间，则不用恢复。
	if (update_occupy_co && update_pending_co && update_occupy_co != update_pending_co)
	{
		// resume stack buffer
		if (update_pending_co->save_buffer && update_pending_co->save_size > 0)
		{
			// 将之前保存起来的栈内容，恢复到运行栈上
			memcpy(update_pending_co->stack_sp, update_pending_co->save_buffer, update_pending_co->save_size);
		}
	}
}
```

## 上下文切换性能更好

coctx_swap.S

```c
void coctx_swap(coctx_t *, coctx_t *) asm("coctx_swap");
struct coctx_t
{
#if defined(__i386__)
	void *regs[ 8 ];
#else // x86_64
	void *regs[ 14 ];
#endif
	size_t ss_size;
	char *ss_sp;
};
// 64 bit
// low | regs[0]: r15 |
//    | regs[1]: r14 |
//    | regs[2]: r13 |
//    | regs[3]: r12 |
//    | regs[4]: r9  |
//    | regs[5]: r8  |
//    | regs[6]: rbp |
//    | regs[7]: rdi |
//    | regs[8]: rsi |
//    | regs[9]: ret |  //ret func addr
//    | regs[10]: rdx |
//    | regs[11]: rcx |
//    | regs[12]: rbx |
// hig | regs[13]: rsp |
```



```assembly
coctx_swap:
; 保存当前协程上下文
; 加载栈指针的值到rax
leaq (%rsp),%rax
	;将栈顶地址存储到ss_sp中
    movq %rax, 104(%rdi)
    ;存储寄存器的值
    movq %rbx, 96(%rdi)
    movq %rcx, 88(%rdi)
    movq %rdx, 80(%rdi)
	movq 0(%rax), %rax
	movq %rax, 72(%rdi) 
    movq %rsi, 64(%rdi)
	movq %rdi, 56(%rdi)
    movq %rbp, 48(%rdi)
    movq %r8, 40(%rdi)
    movq %r9, 32(%rdi)
    movq %r12, 24(%rdi)
    movq %r13, 16(%rdi)
    movq %r14, 8(%rdi)
    movq %r15, (%rdi)
    ;清零rax
	xorq %rax, %rax
; 恢复另一个协程的上下文
	; 恢复寄存器的值
    movq 48(%rsi), %rbp
    movq 104(%rsi), %rsp
    movq (%rsi), %r15
    movq 8(%rsi), %r14
    movq 16(%rsi), %r13
    movq 24(%rsi), %r12
    movq 32(%rsi), %r9
    movq 40(%rsi), %r8
    movq 56(%rsi), %rdi
    movq 80(%rsi), %rdx
    movq 88(%rsi), %rcx
    movq 96(%rsi), %rbx
    ; 调整栈指针
	leaq 8(%rsp), %rsp
	; 将恢复的协程的返回地址压入栈中
	pushq 72(%rsi)
	;恢复 %rsi 寄存器的值
    movq 64(%rsi), %rsi
ret
```

64 位的 Linux 中，第一个参数通过 rdi 来传，第二个参数通过 rsi 来传，这里可以看到我们先将当前的各种寄存器装到第一个参数中，再将第二个参数中的内容装到寄存器中

### 协程调度

```cpp
// 自己管理的epoll结构体
struct stCoEpoll_t
{
	int iEpollFd;	// epoll的fd

	static const int _EPOLL_SIZE = 1024 * 10;

	struct stTimeout_t *pTimeout;  // 超时管理器

	struct stTimeoutItemLink_t *pstTimeoutList; // 目前已超时的事件，仅仅作为中转使用，最后会合并到active上

	struct stTimeoutItemLink_t *pstActiveList; // 正在处理的事件

	co_epoll_res *result; 
};
/**
* epoll_wait的结果
*/
struct co_epoll_res
{
	int size;
	struct epoll_event *events;
	struct kevent *eventlist;
};
/*
* 超时链表中的一个项
*/
struct stTimeoutItem_t
{
	enum
	{
		eMaxTimeout = 40 * 1000 //40s
	};

	stTimeoutItem_t *pPrev;	// 前一个元素
	stTimeoutItem_t *pNext; // 后一个元素

	stTimeoutItemLink_t *pLink; // 该链表项所属的链表

	unsigned long long ullExpireTime;

	OnPreparePfn_t pfnPrepare;  // 预处理函数，在eventloop中会被调用
	OnProcessPfn_t pfnProcess;  // 处理函数 在eventloop中会被调用

	void *pArg; // self routine pArg 是pfnPrepare和pfnProcess的参数

	bool bTimeout; // 是否已经超时
};
/*
* 毫秒级的超时管理器
* 使用时间轮实现
* 但是是有限制的，最长超时时间不可以超过iItemSize毫秒
*/
struct stTimeout_t
{
	/*
	   时间轮
	   超时事件数组，总长度为iItemSize,每一项代表1毫秒，为一个链表，代表这个时间所超时的事件。

	   这个数组在使用的过程中，会使用取模的方式，把它当做一个循环数组来使用，虽然并不是用循环链表来实现的
	*/
	stTimeoutItemLink_t *pItems;
	int iItemSize;   // 默认为60*1000

	unsigned long long ullStart; //目前的超时管理器最早的时间
	long long llStartIdx;  //目前最早的时间所对应的pItems上的索引
};

```

```cpp
/**
* 
* 这个函数也极其重要
* 1. 大部分的sys_hook都需要用到这个函数来把事件注册到epoll中
* 2. 这个函数会把poll事件转换为epoll事件
* 
* @param ctx epoll上下文
* @param fds[] fds 要监听的文件描述符 原始poll函数的参数，
* @param nfds  nfds fds的数组长度 原始poll函数的参数
* @param timeout timeout 等待的毫秒数 原始poll函数的参数
* @param pollfunc 原始的poll函数, g_sys_poll_func
*/
int co_poll_inner( stCoEpoll_t *ctx,struct pollfd fds[], nfds_t nfds, int timeout, poll_pfn_t pollfunc)
{
	if( timeout > stTimeoutItem_t::eMaxTimeout )
	{
		timeout = stTimeoutItem_t::eMaxTimeout;
	}

	int epfd = ctx->iEpollFd;

	// 获取当前协程
	stCoRoutine_t* self = co_self();

	//1.struct change
	stPoll_t& arg = *((stPoll_t*)malloc(sizeof(stPoll_t)));
	memset( &arg,0,sizeof(arg) );

	arg.iEpollFd = epfd;
	arg.fds = (pollfd*)calloc(nfds, sizeof(pollfd));
	arg.nfds = nfds;

	stPollItem_t arr[2];
	if( nfds < sizeof(arr) / sizeof(arr[0]) && !self->cIsShareStack)
	{
		// 如果监听的描述符只有1个或者0个， 并且目前的不是共享栈模型
		arg.pPollItems = arr;
	}	
	else
	{
		// 如果监听的描述符在2个以上，或者协程本身采用共享栈
		arg.pPollItems = (stPollItem_t*)malloc( nfds * sizeof( stPollItem_t ) );
	}
	
	memset( arg.pPollItems,0,nfds * sizeof(stPollItem_t) );

	// 当事件到来的时候，就调用这个callback。
	// 这个callback内部做了co_resume的动作
	arg.pfnProcess = OnPollProcessEvent;
	

	// 保存当前协程，便于调用OnPollProcessEvent时恢复协程
	// 这里取到的值不是和co_self一样吗？为什么不用co_self
	arg.pArg = GetCurrCo( co_get_curr_thread_env() );
	
	//2. add epoll
	for(nfds_t i=0;i<nfds;i++)
	{
		// 将事件添加到epoll中
		arg.pPollItems[i].pSelf = arg.fds + i;
		arg.pPollItems[i].pPoll = &arg;

		// 设置一个预处理的callback
		// 这个函数会在事件active的时候首先触发
		arg.pPollItems[i].pfnPrepare = OnPollPreparePfn; 

		struct epoll_event &ev = arg.pPollItems[i].stEvent;

		// 如果大于-1，说明要监听fd的相关事件了
		// 否则就是个timeout事件
		if( fds[i].fd > -1 )
		{
			// 这个相当于是个userdata, 当事件触发的时候，可以根据这个指针找到之前的数据
			ev.data.ptr = arg.pPollItems + i;

			// 将poll的事件类型转化为epoll
			ev.events = PollEvent2Epoll( fds[i].events );

			// 将fd添加入epoll中
			int ret = co_epoll_ctl( epfd,EPOLL_CTL_ADD, fds[i].fd, &ev );

			if (ret < 0 && errno == EPERM && nfds == 1 && pollfunc != NULL)
			{
				// 如果注册失败
				if( arg.pPollItems != arr )
				{
					free( arg.pPollItems );
					arg.pPollItems = NULL;
				}
				free(arg.fds);
				free(&arg);
				
				// 使用最原生的poll函数
				return pollfunc(fds, nfds, timeout);
			}
		}
		//if fail,the timeout would work
	}

	//3.add timeout
	// 获取当前时间
	unsigned long long now = GetTickMS();

	arg.ullExpireTime = now + timeout;	
	
	// 将其添加到超时链表中
	int ret = AddTimeout( ctx->pTimeout,&arg,now );

	// 如果出错了
	if( ret != 0 )
	{
		co_log_err("CO_ERR: AddTimeout ret %d now %lld timeout %d arg.ullExpireTime %lld",
				ret,now,timeout,arg.ullExpireTime);
		errno = EINVAL;

		if( arg.pPollItems != arr )
		{
			free( arg.pPollItems );
			arg.pPollItems = NULL;
		}
		free(arg.fds);
		free(&arg);

		return -__LINE__;
	}

	// 注册完事件，就yield。切换到其他协程
	// 当事件到来的时候，就会调用callback。
	co_yield_env( co_get_curr_thread_env() );

	// --------------------分割线---------------------------
	// 注意：！！这个时候，已经和上面的逻辑不在同一个时刻处理了
	// 这个时候，协程已经resume回来了！！

	// 清理数据

	// 将该项从超时链表中删除
	RemoveFromLink<stTimeoutItem_t,stTimeoutItemLink_t>( &arg );

	// 将该项涉及事件全部从epoll中删除掉
	// 事件一定要删除，不删除会出现误resume的问题
	for(nfds_t i = 0;i < nfds;i++)
	{
		int fd = fds[i].fd;
		if( fd > -1 )
		{
			co_epoll_ctl( epfd,EPOLL_CTL_DEL,fd,&arg.pPollItems[i].stEvent );
		}
		fds[i].revents = arg.fds[i].revents;
	}

	// 释放内存啦
	int iRaiseCnt = arg.iRaiseCnt;
	if( arg.pPollItems != arr )
	{
		free( arg.pPollItems );
		arg.pPollItems = NULL;
	}

	free(arg.fds);
	free(&arg);

	return iRaiseCnt;
}

/*
* libco的核心调度
* 在此处调度三种事件：
* 1. 被hook的io事件，该io事件是通过co_poll_inner注册进来的
* 2. 超时事件
* 3. 用户主动使用poll的事件
* 所以，如果用户用到了三种事件，必须得配合使用co_eventloop
*
* @param ctx epoll管理器
* @param pfn 每轮事件循环的最后会调用该函数
* @param arg pfn的参数
*/
void co_eventloop( stCoEpoll_t *ctx,pfn_co_eventloop_t pfn,void *arg )
{
	if( !ctx->result )
	{
		ctx->result = co_epoll_res_alloc( stCoEpoll_t::_EPOLL_SIZE );
	}

	co_epoll_res *result = ctx->result;

	for(;;)
	{
		// 最大超时时间设置为 1 ms
		// 所以最长1ms，epoll_wait就会被唤醒
		int ret = co_epoll_wait( ctx->iEpollFd,result,stCoEpoll_t::_EPOLL_SIZE, 1 );

		stTimeoutItemLink_t *active = (ctx->pstActiveList);
		stTimeoutItemLink_t *timeout = (ctx->pstTimeoutList);

		memset( timeout,0,sizeof(stTimeoutItemLink_t) );

		// 处理active事件
		for(int i=0;i<ret;i++)
		{
			// 取出本次epoll响应的事件所对应的stTimeoutItem_t
			stTimeoutItem_t *item = (stTimeoutItem_t*)result->events[i].data.ptr;
			
			// 如果定义了预处理函数，则首先进行预处理。
			// 如果是co_poll_inner/co_poll或者是被hook的函数，则这个函数是OnPollPreparePfn
			if( item->pfnPrepare )
			{
				item->pfnPrepare( item,result->events[i], active );
			}
			else
			{
				// 否则将其加到active的链表中
				AddTail( active,item );
			}
		}

		// 获取当前时刻
		unsigned long long now = GetTickMS();

		// 以当前时间为超时截止点
		// 取出所有的超时事件，放入timeout 链表中
		TakeAllTimeout( ctx->pTimeout, now, timeout );

		// 遍历所有的项，将bTimeout置为true
		stTimeoutItem_t *lp = timeout->head;
		while( lp )
		{
			//printf("raise timeout %p\n",lp);
			lp->bTimeout = true;
			lp = lp->pNext;
		}

		// 将timeout合并到active上面
		Join<stTimeoutItem_t,stTimeoutItemLink_t>( active,timeout );

		lp = active->head;
		while( lp )
		{
			PopHead<stTimeoutItem_t,stTimeoutItemLink_t>( active );
			if( lp->pfnProcess )
			{
				 /*
				   处理该事件，默认为OnPollProcessEvent
				   在OnPollProcessEvent中
				   会使用co_resume恢复协程

				   协程会回到co_poll_inner里面
				 */
				lp->pfnProcess( lp );
			}

			lp = active->head;
		}

		// 每轮事件循环的最后调用该函数
		if( pfn )
		{
			if( -1 == pfn( arg ) )
			{
				break;
			}
		}

	}
}
```

