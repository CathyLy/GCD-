### 多线程编程
    1.实际上CPU执行的CPU指令列为一条无分叉路径,OSX 和iOS的核心XNU内核在发生操作系统事件时会切换路径,
    使用多线程的程序可以再某一线程和其他线程之间反复的上下文切换,看上去像一个CPU核能并行执行多个线程
    2.在具有多CPU的情况下,是真正提供了多个CPU核并行执行多个线程的技术

### GCD
    GCD是异步执行任务的技术之一,将应用程序中记述的线程管理用的代码在系统中实现,开发者只需要定义想要
    执行的任务在block中并追加到适当的Dispatch Queue中,CGD就能生成必要的线程并计划执行任务
    
    1.Dispatch Queue 的种类
    Serial Dispatch Queue
    Concurrent Dispatch Queue
    dispatch_queue_create 可以生成任意多个Dispatch Queue
    dispatch_queue_t queue = dispatch_queue_create("myqueue", NULL);//串行队列
    dispatch_queue_t queue = dispatch_queue_create("myqueue", DISPATCH_QUEUE_CONCURRENT);//并行队列
   2.MainDispatch Queue/Global Dispatch Queue
     Global Dispatch Queue有4个执行优先级,但是通过XNU内核用于Global Dispatch Queue 的线程并不能保证实时性,执行优先级只是大致的判断
    
    dispatch_queue_create 函数生成的Dispatch Queue执行的优先级都与默认优先级的Global Dispatch queue相同
    dispatch_set_target_queue 变更生成的Dispatch Queue 执行的优先级顺序
    //queue为指定要变更的queue 将queue的优先级设置成和globalQueue一样的优先级
    dispatch_set_target_queue(queue, globalQueue);
    
    使用Dispatch Group
    a. dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_group_t group = dispatch_group_create();
    dispatch_group_async(group, queue, ^{
        NSLog(@"1");
    });
    dispatch_group_async(group, queue, ^{
        NSLog(@"2");
    });
    dispatch_group_notify(group, queue, ^{
        NSLog(@"end");
    });
 
    b.dispatch_group_wait表示在经过指定时间或属于指定Dispatch Group 处理的全部执行结束之前,执行该函数的线程停止
    long result = dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
    long result = dispatch_group_wait(group, DISPATCH_TIME_NOW);//不用任何等待即可判定属于Dispatch Group的处理
    是否执行结束在主线程的Runloop的每次循环中,可检查执行是否结束,从而不耗费多余的等待时间
    
    c.dispatch_barrier_async 
    函数会等待追加到并发队列上的并发执行的处理全部结束之后再讲指定的处理追加到该queue上,然后由dispatch_barrier_async函数追加的处理执行完毕后,追加到queue上的处理又开始并发执行
dispatch_barrier_async 函数的处理流程
![image](https://raw.githubusercontent.com/CathyLy/imageForSource/master/dispatch_barrier_async.png)
    
    使用dispatch_barrier_async和Concurrent Dispatch Queue函数可实现高效率的数据访问和文件访问
    dispatch_barrier_async和dispatch_barrier_sync的区别
    a).两个函数都会等等待在它前面插入队列的任务先执行完,会等待他们自己的任务执行完之后再执行后面的任务
    b).在将任务插入到queue的时候,dispatch_barrier_sync需要等待自己的任务结束之后才会继续程序,然后插入被写在它之后的任务,在执行
    dispatch_barrier_async将自己的任务插入到队列之后,不会等待自己的任务结束就会继续把后面的任务插入到queue,等待自己的任务执行结束执行后面插入的任务
    
    d.dispatch_sync 和 dispatch_async
    dispatch_async:指的是将指定的Block""非同步"地追加到指定queue中,这个函数不做任何等待
    dispatch_sync:将指定的block"同步追加到指定的queue中,在追加block结束之前,dispatch_sync会一直等待,如dispatch_group_wait函数类似
    //如下代码会导致死锁,该源码在Main Dispatch Queue 即主线程中执行指定的Block,并等待其执行结束,但其实主线程中正在执行这些源码,
    //所以无法执行追加到main queue的block,类似在serial Dispatch queue中也会有同样的问题存在
    dispatch_queue_t queue = dispatch_get_main_queue();
    dispatch_sync(queue, ^{
        NSLog(@"hello");
    });
    
    e.dispatch_suspend / dispatch_resume
    dispatch_suspend函数挂起指定的queue
    dispatch_resume函数恢复指定的queue
    这个两个函数对已经执行的处理没有影响,挂起后,追加到queue中但未执行的处理在此之后停止执行,而恢复则使得这些处理能够继续执行
    
    f.dispatch_semaphore_t
     dispatch_semaphore_create　　　创建一个semaphor
     dispatch_semaphore_signal　　　发送一个信号
     dispatch_semaphore_wait　　　　等待信号
    dispatch_semaphore_t是持有计数的信号,计数为0时等待,计数大于等于1的时候不等待
    dispatch_queue_t queue1 = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    //这段代码执行后由内存错误导致应用程序异常结束的概率很高
    NSMutableArray *arr = [NSMutableArray array];
    for (int i = 0; i < 100000; i ++) {
        dispatch_async(queue1, ^{
            [arr addObject:[NSNumber numberWithInt:i]];
        });
    }
    
    //使用Dispatch Semaphore 可以解决这个问题
    dispatch_queue_t queue1 = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);
    NSMutableArray *arr = [NSMutableArray array];
    for (int i = 0; i < 100000; i ++) {
        dispatch_async(queue1, ^{
            //等待Dispatch Semaphore 直到Dispatch Semaphore的计数值大于等于1
            dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
            //Dispatch Semaphore的计数值达到大于等于1,Dispatch Semaphore减去1,dispatch_semaphore_wait函数执行返回
            //即执行到此时的Dispatch Semaphore计数值恒为"0",由于可访问NSMutableArray类对象的线程只有1个,所以可以安全更新
            [arr addObject:[NSNumber numberWithInt:i]];
            dispatch_semaphore_signal(semaphore);
        });
    }
    
##### g.Dispatch I/O
     在使用多个线程更快地并列读取文件,可以通过Dispatch I/O 和 Dispatch Data
     在通过Dispatch I/O 读写文件时,使用Global Dispatch Queue将1一个文件按照某一个大小read/write
     下面是苹果使用Dispatch I/O 和Disaptch Data的部分例子
        if (where == ASL_STORE_LOCATION_MEMORY)
    {
        /* create a pipe */
        
        asl_aux_context_t *ctx = (asl_aux_context_t *)calloc(1, sizeof(asl_aux_context_t));
        if (ctx == NULL) return -1;
        
        status = pipe(fdpair);
        if (status < 0)
        {
            free(ctx);
            return -1;
        }
        
        /* give read end to dispatch_io_read */
        fd = fdpair[0];
        sem = dispatch_semaphore_create(0);
        ctx->sem = sem;
        ctx->fd = fdpair[1];
        
        status = _asl_aux_save_context(ctx);
        if (status != 0)
        {
            close(fdpair[0]);
            close(fdpair[1]);
            dispatch_release(sem);
            free(ctx);
            return -1;
        }
        
        //创建一个串行队列
        pipe_q = dispatch_queue_create("PipeQ", NULL);
        //创建一个dispatch I/O
        /*
         创建一个持有文件描述符的通道,在创建之后,不准以任何方式修改这个文件描述符,两种类型不同的通道
         DISPATCH_IO_STREAM 0 流      如果你打开了一个套接字,可以创建一个流通道
         DISPATCH_IO_RANDOM 1 随机存取 如果你打开的是硬盘上的文件,可以使用它来创建一个随机存取的通道(因为这样的文件描述符是可寻址的)
         如果你想创建一个文件通道,最好使用一个路径参数dispatch_io_create_with_path,并且让GCD来打开这个文件
         */
        pipe_channel = dispatch_io_create(DISPATCH_IO_STREAM, fd, pipe_q, ^(int err){
            close(fd);
        });
        
        *out_fd = fdpair[1];
        
        //该函数设定一次读取的大小(分割大小)
        dispatch_io_set_low_water(pipe_channel, SIZE_MAX);
        //无论数据何时读完和写完,读写操作调用一个block来结束,这些都是以非阻塞,异步I/O的形式高效实现的
        dispatch_io_read(pipe_channel, 0, SIZE_MAX, pipe_q, ^(bool done, dispatch_data_t pipedata, int err){
            if (err == 0)//读取无误
            {
                //读取单个文件块的大小
                size_t len = dispatch_data_get_size(pipedata);
                if (len > 0)
                {
                    //定义一个字节数组bytes
                    const char *bytes = NULL;
                    char *encoded;
                    //dispatch_io_read 函数指定的读取结束时回调用的block中拿到每一块读取好的数据,并进行合并处理
                    dispatch_data_t md = dispatch_data_create_map(pipedata, (const void **)&bytes, &len);
                    encoded = asl_core_encode_buffer(bytes, len);
                    asl_set((aslmsg)merged_msg, ASL_KEY_AUX_DATA, encoded);
                    free(encoded);
                    _asl_send_message(NULL, merged_msg, -1, NULL);
                    asl_msg_release(merged_msg);
                    dispatch_release(md);
                }
            }
            
            if (done)
            {
                dispatch_semaphore_signal(sem);
                dispatch_release(pipe_channel);
                dispatch_release(pipe_q);
            }
        });
        
        return 0;
        }
　　 
　　 
#### GCD的实现
##### 一.Dispatch Queue 
    用于管理追加的Block的C语言层实现的FIFO队列
    atomic函数中实现的用于排他控制的轻量级信号
    用于管理线程的C语言层实现的一些容器
    用于实现Dispatch Queue 而使用的软件组件
    

组件名称 | 提供技术
---|---
libdispatch | Dispatch Queue
Libc(pthreads) | pthread_workqueue
XNU内核 | workqueue
    FIFO队列管理是通过dispatch_async等函数所追加的Block
    Block 并不是直接加入到FIFO队列中,而是加入到dispatch_continuation_t类型的结构体中,然后在加入到FIFO队列中,
    dispatch_continuation_t 用于记录Block所属的Dispatch Group和一些其他信息(类似执行的上下文)
    Global Dispatch Queue 有8种优先级:
      High Priority
      Default Priority
      Low Priority
      Background Priority
      High Overcommit Priority
      Default Overcommit Priority
      Low Overcommit Priority
      Background Overcommit Priority
    优先级中附有Overcommit的Global Dispatch Queue 使用在serial Dispatch Queue 中,不管系统状态如何,都会强制生成线程的Dispatch Queue
    
    pthread_workqueue:工作队列,是一个用于创建内核线程的接口,通过它创建的内核线程来执行内核其他模块排列到队列里的工作
    使用 pthread_workqueue_create_np 创建pthread_workqueue
    
    a.pthread_workqueue包含在Libc提供的pthreads API中,其使用bsdthread_register和workq_open 系统调用
    b.在初始化XNU内核的workqueue 之后获取workqueue信息
    
    XNU内核持有4种workqueue:
       WORKQUEUE_HIGH_PRIOQUEUE
       WORKQUEUE_DEFALUT_PRIOQUEUE
       WORKQUEUE_LOW_PRIOQUEUE
       WORKQUEUE_BG_PRIOQUEUE
       
##### 一.Block的执行过程 
    a.libdispatch从Global Dispatch Queue 自身的FIFO队列中取出Dispatch Continuation 
    b.调用pthread_workqueue_additem_np函数,传入这些参数：dispatch queue自身、一个符合其优先级workqueue,执行dispatch continuation的回调函数
    
    注:回调函数是一个通过函数指针调用的函数,如果你把函数的指针作为参数传递给另一个函数,这个指针被用来调用其所指向的函数
    
    c.pthread_workqueue_additem_np函数使用workq_kernreturn系统调用,通知workqueue应当执行的项目,XNU内核基于系统状态判断是否要生成线程,
    如果是Overcommit优先级的Global Dispatch Queue,workqueue会始终生成线程
    d.workqueue的线程执行pthread_workqueue 函数,该函数调用libdispatch的回调函数,在该回调函数中执行加入到的Dispatch Continuation 的Block
    e.Block执行结束后,进行通知Dispatch Group结束 ,释放Dispatch Continuation 等处理 ,执行下一个Block

##### Dispatch Source
    
    
    
    
    

　　 

