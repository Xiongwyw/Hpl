### service_manager启动
#### service manager启动主要分三步：
1. 打开Binder文件节点
2. 设置当前进程为守护进程
3. 进入Loop状态，等待执行client请求
4. 读取到请求后，分析并处理该请求。

**在这个过程中，在Binder驱动程序中建立了一个struct binder_proc结构、一个struct  binder_thread结构和一个struct binder_node结构，这样，Service Manager就在Android系统的进程间通信机制Binder担负起守护进程的职责了**

##### 打开Binder文件节点
	  binder_open(driver, 128*1024);
  首先会通过open("/dev/binder", O_RDWR), 打开文件节点。
  进入到驱动的binder_open(struct inode *nodp, struct file *filp)
  在驱动binder_open中创建struct binder_proc保存当前进程上下文信息。 
  并保存到file->private_data。 同时也会保存到binder_procs中，驱动内部使用。

      struct binder_proc {
    	struct hlist_node proc_node;
		//进程内处理用户请求的Thread,它的最大数量由max_threads来决定
    	struct rb_root threads;
        //进程内Binder实体
    	struct rb_root nodes;
		//进程内Binder引用
    	struct rb_root refs_by_desc;
		//进程内Binder引用
    	struct rb_root refs_by_node;
    	int pid;
    	struct vm_area_struct *vma;
    	struct task_struct *tsk;
    	struct files_struct *files;
    	struct hlist_node deferred_work_node;
    	int deferred_work;
		//映射的物理地址在内核空间中的位置
    	void *buffer;
		//内核虚拟地址与进程虚拟地址之前的差值
    	ptrdiff_t user_buffer_offset;
     
    	struct list_head buffers;
    	struct rb_root free_buffers;
    	struct rb_root allocated_buffers;
    	size_t free_async_space;
     
		//物理地址数据结构
    	struct page **pages;
		//映射物理地址大小
    	size_t buffer_size;
    	uint32_t buffer_free;
    	struct list_head todo;
    	wait_queue_head_t wait;
    	struct binder_stats stats;
    	struct list_head delivered_death;
    	int max_threads;
    	int requested_threads;
    	int requested_threads_started;
    	int ready_threads;
    	long default_priority;
      };

  然后：mmap->binder_mmap, 内存地址映射。
  将fd同时映射到进程地址空间和内核地址空间，这样可以减少数据拷贝次数，提高binder效率。这也是binder精髓所在。

##### 告知驱动自己是service manager守护进程
	  binder_become_context_manager(struct binder_state* bs);
	  ->ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
      //cmd为BINDER_SET_CONTEXT_MGR
      ->binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
  在binder_ioctl会创建binder_thread结构体，并为service manager创建binder实体binder_node。
  binder_thread表示一个线程，当前上下文中表示执行binder_become_context_manager的线程。
  binder_thread结构如下：
      struct binder_thread {
		//进程信息
    	struct binder_proc *proc;
		//链接到binder_proc中的threads红黑树
    	struct rb_node rb_node;
    	int pid;
		//当前线程状态
    	int looper;
        //当前线程正在处理的任务
    	struct binder_transaction *transaction_stack;
        //发往当前线程的数据列表
    	struct list_head todo;
    	uint32_t return_error; /* Write failed, return error code in read buf */
    	uint32_t return_error2; /* Write failed, return error code in read */
    		/* buffer. Used when sending a reply to a dead process that */
    		/* we are also waiting on */
    	wait_queue_head_t wait;
        //保存一些统计信息
    	struct binder_stats stats;
      };

  binder_node结构：
	  struct binder_node {
		int debug_id;
		struct binder_work work;
		union {
            //链接到proc nodes红黑树
			struct rb_node rb_node;
            //实体所在进程被销毁时，该实体被其它进程引用。实体存放到dead_node哈希表中
			struct hlist_node dead_node;
		};
		struct binder_proc *proc;
        //所有引用该实体组成的链表
		struct hlist_head refs;
        //实体引用计数
		int internal_strong_refs;
		int local_weak_refs;
		int local_strong_refs;
        //实体在用户空间的地址
		void __user *ptr;
        //实体附加数据
		void __user *cookie;
		unsigned has_strong_ref : 1;
		unsigned pending_strong_ref : 1;
		unsigned has_weak_ref : 1;
		unsigned pending_weak_ref : 1;
		unsigned has_async_transaction : 1;
		unsigned accept_fds : 1;
		int min_priority : 8;
		struct list_head async_todo;
	};
  
  binder_ioctl代码如下：
		static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned 	long arg)
		{
			int ret;
			struct binder_proc *proc = filp->private_data;
			struct binder_thread *thread;
			unsigned int size = _IOC_SIZE(cmd);
			void __user *ubuf = (void __user *)arg;
		 
			/*printk(KERN_INFO "binder_ioctl: %d:%d %x %lx\n", proc->pid, current->pid, cmd, arg);*/
		 
			ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
			if (ret)
				return ret;
		 
			mutex_lock(&binder_lock);
            //创建binder_thread
			thread = binder_get_thread(proc);
			if (thread == NULL) {
				ret = -ENOMEM;
				goto err;
			}
		 
			switch (cmd) {
		        ......
			case BINDER_SET_CONTEXT_MGR:
				if (binder_context_mgr_node != NULL) {
					printk(KERN_ERR "binder: BINDER_SET_CONTEXT_MGR already set\n");
					ret = -EBUSY;
					goto err;
				}
				if (binder_context_mgr_uid != -1) {
					if (binder_context_mgr_uid != current->cred->euid) {
						printk(KERN_ERR "binder: BINDER_SET_"
							"CONTEXT_MGR bad uid %d != %d\n",
							current->cred->euid,
							binder_context_mgr_uid);
						ret = -EPERM;
						goto err;
					}
				} else
				binder_context_mgr_uid = current->cred->euid;
                //创建binder实体
				binder_context_mgr_node = binder_new_node(proc, NULL, NULL);
				if (binder_context_mgr_node == NULL) {
					ret = -ENOMEM;
					goto err;
				}
				binder_context_mgr_node->local_weak_refs++;
				binder_context_mgr_node->local_strong_refs++;
				binder_context_mgr_node->has_strong_ref = 1;
				binder_context_mgr_node->has_weak_ref = 1;
				break;
		        ......
			default:
				ret = -EINVAL;
				goto err;
			}
			ret = 0;
		err:
			if (thread)
				thread->looper &= ~BINDER_LOOPER_STATE_NEED_RETURN;
			mutex_unlock(&binder_lock);
			wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
			if (ret && ret != -ERESTARTSYS)
				printk(KERN_INFO "binder: %d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
			return ret;
		}  

##### 进入Loop循环，等待执行client请求
首先通过ioctl写入BC_ENTER_LOOPER命令，使用了binder_write_read结构体。最终会修改proc->threads->loop的状态为BINDER_LOOPER_STATE_ENTERED
binder_write_read结构如下：

    struct binder_write_read {
    	signed long	write_size;	/* bytes to write */
    	signed long	write_consumed;	/* bytes consumed by driver */
        //指向binder_transaction_data结构体
    	unsigned long	write_buffer;
    	signed long	read_size;	/* bytes to read */
    	signed long	read_consumed;	/* bytes consumed by driver */
        //指向binder_transaction_data结构体
    	unsigned long	read_buffer;
    };

binder_transaction_data结构体如下：
	struct binder_transaction_data {
		/* The first two are only used for bcTRANSACTION and brTRANSACTION,
		 * identifying the target and contents of the transaction.
		 */
		union {
            //binder是引用时，用handle表示
			size_t	handle;	/* target descriptor of command transaction */
			//binder是实体时，ptr表示在进程中的地址
			void	*ptr;	/* target descriptor of return transaction */
		} target;
		void		*cookie;	/* target object cookie */
        //请求的命令，当前上下文是BC_ENTER_LOOPER
		unsigned int	code;		/* transaction command */
	 
		/* General information about the transaction. */
		unsigned int	flags;
		pid_t		sender_pid;
		uid_t		sender_euid;
		size_t		data_size;	/* number of bytes of data */
		size_t		offsets_size;	/* number of bytes of offsets */
	 
		/* If this transaction is inline, the data immediately
		 * follows here; otherwise, it ends with a pointer to
		 * the data buffer.
		 */
        //data是实际写入的数据，
		union {
			struct {
				/* transaction data */
				const void	*buffer;
				/* offsets from buffer to flat_binder_object structs */
				const void	*offsets;
			} ptr;
			uint8_t	buf[8];
		} data;
	};

每一个Binder实体或者引用，通过struct flat_binder_object

	struct flat_binder_object {
		/* 8 bytes for large_flat_header. */
		unsigned long		type;
		unsigned long		flags;
	 
		/* 8 bytes of data. */
		union {
			void		*binder;	/* local object */
			signed long	handle;		/* remote object */
		};
	 
		/* extra data associated with local object */
		void			*cookie;
	};


binder_loop function:

	void binder_loop(struct binder_state *bs, binder_handler func)
	{
	    int res;
	    struct binder_write_read bwr;
	    unsigned readbuf[32];
	 
	    bwr.write_size = 0;
	    bwr.write_consumed = 0;
	    bwr.write_buffer = 0;
	    
	    readbuf[0] = BC_ENTER_LOOPER;
        //写入BC_ENTER_LOOPER命令， 告知进入Loop待状态
	    binder_write(bs, readbuf, sizeof(unsigned));
	 
	    for (;;) {
	        bwr.read_size = sizeof(readbuf);
	        bwr.read_consumed = 0;
	        bwr.read_buffer = (unsigned) readbuf;
	 		//等待client请求的到来
	        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
	 
	        if (res < 0) {
	            LOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
	            break;
	        }
	        //分析读取到的请求， 并交由func处理
	        res = binder_parse(bs, 0, readbuf, bwr.read_consumed, func);
	        if (res == 0) {
	            LOGE("binder_loop: unexpected reply?!\n");
	            break;
	        }
	        if (res < 0) {
	            LOGE("binder_loop: io error %d %s\n", res, strerror(errno));
	            break;
	        }
	    }
	}


