### service_manager启动

* 打开Binder文件节点
	```binder_open(driver, 128*1024);```
  首先会通过open("/dev/binder", O_RDWR), 打开文件节点。
  进入到驱动的binder_open(struct inode *nodp, struct file *filp)
	
