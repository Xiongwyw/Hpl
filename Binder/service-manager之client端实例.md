### 获取Service Manager远程接口实例
##### 获取远程接口实例方法如下：

    sp<IServiceManager> defaultServiceManager = defaultServiceManager();
	sp<IServiceManager> defaultServiceManager()
	{
	 
	    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
	 
	    {
	        AutoMutex _l(gDefaultServiceManagerLock);
	        if (gDefaultServiceManager == NULL) {
	            gDefaultServiceManager = interface_cast<IServiceManager>(
	                ProcessState::self()->getContextObject(NULL));
	        }
	    }
	 
	    return gDefaultServiceManager;
	}
由实现可以知道，defaultServiceManager方法是单例模式。

##### 首先分析ProcessState::self()
ProcessState也是个单例模式，一个进程中只有一个ProcessState对象。

	sp<ProcessState> ProcessState:: self()
	{
	    Mutex::Autolock _l(gProcessMutex);
	    if(gProcess != NULL){
	        return gProcess;
	    }
	    gProcess = new ProcessState("/dev/binder");
	    return grProcess;
	}
ProccessState在创建过程中，会打开binder节点。并创建binder_proc记录进程上下文信息和线程信息。过程如下：
首先在ProcessState的构造函数中，会调用open_binder()打开"/dev/binder"。最终会调到binder驱动中的binder_open()。在binder_open中，会创建binder_proc记录当前进程上下文信息。同时会将/dev/binder节点映射到进程虚拟地址空间和内核虚拟地址空间。这样能保证使用一次数据拷贝，实现在两个进程中传递数据。

其次在打开"/dev/binder"节点之后，会通过ioctl向binder写入cmd。写入cmd之前，调用binder_get_thread创建binder_thread对象，保存当前thead信息。创建的binder_thread会被储存到binder_proc.threads中。

##### 其次ProcessState::getContextObject()
在ProcessState::getContextObject中， 通过handle=0构造一个BpBinder对象。这时defaultServiceManager()变换成如下形式：

	sp<IServiceManager> defaultServiceManager()
    {
		gDefaultServiceManager = interface_cast<IServiceManager>(
		                new BpBinder(0));
        return gDefaultServiceManager;
    }

interface_cast是个模板方法，最终会调到IServiceManager::asInterface(sp<Binder>&)
在asInterface方法中，创建BpServiceManager对象。defaultServiceManager最终转化为以下形式：

	sp<IServiceManager> defaultServiceManager()
    {
		gDefaultServiceManager = new BpServiceManager(new BpBinder(0));
        return gDefaultServiceManager;
    }

这样就拿到了servicemanager的远程接口gDefaultServiceManager。

在Android系统的Binder机制中，Server和Client拿到这个Service Manager远程接口之后怎么用呢？

对Server来说，就是调用IServiceManager::addService这个接口来和Binder驱动程序交互了，即调用BpServiceManager::addService 。而BpServiceManager::addService又会调用通过其基类BpRefBase的成员函数remote获得原先创建的BpBinder实例，接着调用BpBinder::transact成员函数。在BpBinder::transact函数中，又会调用IPCThreadState::transact成员函数，这里就是最终与Binder驱动程序交互的地方了。回忆一下前面的类图，IPCThreadState有一个PorcessState类型的成中变量mProcess，而mProcess有一个成员变量mDriverFD，它是设备文件/dev/binder的打开文件描述符，因此，IPCThreadState就相当于间接在拥有了设备文件/dev/binder的打开文件描述符，于是，便可以与Binder驱动程序交互了。

