lib syscall_all.c中sys_write_dev和sys_read_dev：系统读写操作

fs/ide.c中ide_write和ide_write函数：用户读写

fs/fs.c中的free_block：释放磁盘块（位图法）

fs/fsformat.c中create_file函数：这个函数从一个目录文件出发，目的是**寻找第一个能够放下新的文件控制块的位置**。当它找到一个指向已经被删除了的文件的文件控制块指针时，它直接返回，以求后续操作将这个空间覆盖掉。而当没有找到时，其直接进行拓展一个Block，并返回这个新的空白空间的起始地址。这里我们需要注意到，**一个未被占用的空间被解释为文件控制块指针时，其行为和一个指向已经被删除了的文件的文件控制块指针一致**，因此能够被统一处理。

NDIRECT：直接指针数目（10）。\#define FILE2BLK   (BY2BLK/sizeof(struct File))   \#define BY2BLK    BY2PG，即FILE2BLK意思是该磁盘块中所有文件控制块数量  

fs/fs.c中的diskaddr:给出块的序号，计算指定磁盘块对应的虚拟地址

fs/fs.c中的map_block函数：检查指定磁盘块是否以及映射到内存，如果没有，分配一页内存来保存数据（unmap_block同理）

fs/fs.c中的dir_lookup函数：查找某个目录下是否存在指定文件。file_get_block(dir, i, &blk); // 读出该目录第i个磁盘块的信息并保存在blk中

user/file.c中的open函数：va = fd2data(fd);获取文件初始地址，若打开成功，返回文件描述符编号

user/fsipc.c中的fsip_remove，user/file.c中的remove，fs/serv.c中的serve_remove函数：删除文件

##### fd.c

描述了write\read\close等用户使用的接口，其基于文件描述符去寻找所需的进行的操作。

##### file.c

定义了一系列基于文件控制块的函数，通过对文件控制块的解析和调用IPC来实现功能。

##### fsipc.c

定义了一系列IPC相关的函数，主要功能在于打包所要传递的参数和发送\接收。



运行

/OSLAB/gxemul -E testmips -C R3000 -M 64 -d gxemul/fs.img gxemul/vmlinux