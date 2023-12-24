# 扩展练习 Challenge1：完成基于“UNIX的PIPE机制”的设计方案

```C
// 管道结构
struct pipe {
    struct semaphore mutex; // 用于互斥访问管道数据结构
    struct semaphore data_available; // 表示管道中有数据可读
    struct semaphore space_available; // 表示管道中有空间可写
    char buffer[PIPE_BUFFER_SIZE];
    int read_index;
    int write_index;
    int count; // 记录当前管道中的数据量
};
// 接口定义
// 创建管道
int pipe_create(struct pipe *p) {
    // 初始化互斥锁和信号量
    semaphore_init(&p->mutex, 1);
    semaphore_init(&p->data_available, 0);
    semaphore_init(&p->space_available, PIPE_BUFFER_SIZE);
    
    // 初始化其他管道结构成员
    p->read_index = 0;
    p->write_index = 0;
    p->count = 0;

    return 0; // 成功创建
}
// 写入数据到管道
void pipe_write(struct pipe *p, char *data, int size) {
    semaphore_down(&p->space_available); // 等待空间可写
    semaphore_down(&p->mutex); // 获取互斥锁

    // 写入数据到管道缓冲区
    for (int i = 0; i < size; ++i) {
        p->buffer[p->write_index] = data[i];
        p->write_index = (p->write_index + 1) % PIPE_BUFFER_SIZE;
    }
    p->count += size;

    semaphore_up(&p->mutex); // 释放互斥锁
    semaphore_up(&p->data_available); // 通知有数据可读
}
// 从管道读取数据
void pipe_read(struct pipe *p, char *buffer, int size) {
    semaphore_down(&p->data_available); // 等待数据可读
    semaphore_down(&p->mutex); // 获取互斥锁

    // 读取数据从管道缓冲区
    for (int i = 0; i < size; ++i) {
        buffer[i] = p->buffer[p->read_index];
        p->read_index = (p->read_index + 1) % PIPE_BUFFER_SIZE;
    }
    p->count -= size;

    semaphore_up(&p->mutex); // 释放互斥锁
    semaphore_up(&p->space_available); // 通知有空间可写
}
// 关闭管道
void pipe_close(struct pipe *p) {
    // 清理资源，释放信号量等
    semaphore_cleanup(&p->mutex);
    semaphore_cleanup(&p->data_available);
    semaphore_cleanup(&p->space_available);
}
```
**同步互斥问题处理：**

通过mutex信号量实现对管道数据结构的互斥访问。

使用两个信号量data_available和space_available，分别表示有数据可读和有空间可写，确保读写进程之间的同步。

# 扩展练习 Challenge2：完成基于“UNIX的软连接和硬连接机制”的设计方案

```C
// 索引节点结构
struct inode {
    // 其他inode相关信息...
    int link_count; // 链接计数，记录硬链接的数量
    char symlink_path[MAX_PATH_LENGTH]; // 用于存储软链接的路径
    struct semaphore inode_mutex; // 用于互斥访问inode结构
};
// 接口定义
// 创建硬链接
int create_hard_link(const char *existing_path, const char *new_path) {
    // 获取互斥锁，确保在创建链接时不会有其他进程修改inode
    semaphore_down(&inode_mutex);
    
    // 找到已存在文件的inode，并将新路径指向相同的inode
    // 增加链接计数
    
    semaphore_up(&inode_mutex);
    return 0; // 成功创建硬链接
}
// 创建软链接
int create_soft_link(const char *target_path, const char *link_path) {
    // 获取互斥锁，确保在创建链接时不会有其他进程修改inode
    semaphore_down(&inode_mutex);

    // 创建一个新的inode，其中包含软链接的路径
    
    semaphore_up(&inode_mutex);
    return 0; // 成功创建软链接
}
// 读取软链接内容
int read_soft_link(const char *link_path, char *buffer, size_t size) {
    // 通过link_path找到软链接的inode
    // 读取软链接中存储的路径信息到buffer
}
```

**设计实现概要**

1. 创建硬链接:
   + 在文件系统中，通过create_hard_link接口实现。该接口需要获取互斥锁，以确保在创建链接时不会有其他进程修改inode。
   + 在文件系统中找到已存在文件的inode，并将新路径指向相同的inode。同时，增加链接计数，表示有一个额外的链接指向该inode。
2. 创建软链接:
   + 通过create_soft_link接口在文件系统中创建一个新的inode，其中包含软链接的路径。需要获取互斥锁，确保在创建链接时不会有其他进程修改inode。
3. 读取软链接内容:
   + 通过read_soft_link接口读取软链接的内容。需要根据link_path找到软链接的inode，并从中读取存储的路径信息。
4. 同步互斥问题处理:
   + 在inode结构中加入互斥锁inode_mutex，确保在多线程或多进程环境中，对inode结构的访问是互斥的，避免竞争条件。
