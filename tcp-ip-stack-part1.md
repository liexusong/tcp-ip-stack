## Socket族系统调用入口

所有Socket相关的系统调用都由函数 `sys_socketcall()` 进入, 源码如下:
```cpp
asmlinkage int
sys_socketcall(int call, unsigned long *args)
{
  int er;
  switch (call) {
    case SYS_SOCKET:
        er = verify_area(VERIFY_READ, args, 3 * sizeof(long));
        if(er)
            return er;
        return(sock_socket(get_fs_long(args+0),
                   get_fs_long(args+1),
                   get_fs_long(args+2)));
    case SYS_BIND:
        er = verify_area(VERIFY_READ, args, 3 * sizeof(long));
        if(er)
            return er;
        return(sock_bind(get_fs_long(args+0),
                 (struct sockaddr *)get_fs_long(args+1),
                 get_fs_long(args+2)));
    case SYS_CONNECT:
        er = verify_area(VERIFY_READ, args, 3 * sizeof(long));
        if(er)
            return er;
        return(sock_connect(get_fs_long(args+0),
                    (struct sockaddr *)get_fs_long(args+1),
                    get_fs_long(args+2)));
    ...
    default:
        return(-EINVAL);
  }
}
```
从上面的代码可以知道, 参数 `call` 指定要调用哪个具体的Socket族函数, 譬如要调用 `bind()` 函数就传入 `SYS_BIND` 值, 而具体函数相关的值就由参数 `args` 传入. 由于传入参数的值保存在用户空间, 所以必须使用 `get_fs_long()` 函数来把参数copy到内核空间.

在用户进程中, 我们可以使用 `socket()` 系统调用来创建一个 `socket` 句柄, `socket()` 函数的原型如下:
```cpp
int socket(int domain, int type, int protocol);
```
各个参数的含义如下:
* domain：即协议域，又称为协议族（family）。常用的协议族有，AF_INET、AF_INET6、AF_LOCAL（或称AF_UNIX，Unix域socket）、AF_ROUTE等等。协议族决定了socket的地址类型，在通信中必须采用对应的地址，如AF_INET决定了要用ipv4地址（32位的）与端口号（16位的）的组合、AF_UNIX决定了要用一个绝对路径名作为地址。
* type：指定socket类型。常用的socket类型有，SOCK_STREAM、SOCK_DGRAM、SOCK_RAW、SOCK_PACKET、SOCK_SEQPACKET等等。
* protocol：故名思意，就是指定协议。常用的协议有，IPPROTO_TCP、IPPTOTO_UDP、IPPROTO_SCTP、IPPROTO_TIPC等，它们分别对应TCP传输协议、UDP传输协议、STCP传输协议、TIPC传输协议。

而 `socket()` 函数调用的内核函数就是 `sock_socket()`, 我们来看看 `sock_socket()` 函数的实现:
```cpp
static int
sock_socket(int family, int type, int protocol)
{
  int i, fd;
  struct socket *sock;
  struct proto_ops *ops;

  /* Locate the correct protocol family. */
  for (i = 0; i < NPROTO; ++i) { // 遍历所有注册的socket操作(如inet, unix等)
    if (pops[i] == NULL) continue;
    if (pops[i]->family == family) break;
  }
  ...
  ops = pops[i];
  ...
  if (!(sock = sock_alloc(1))) {
    printk("sock_socket: no more sockets\n");
    return(-EAGAIN);
  }

  sock->type = type;
  sock->ops = ops;
  if ((i = sock->ops->create(sock, protocol)) < 0) {
    sock_release(sock);
    return(i);
  }

  if ((fd = get_fd(SOCK_INODE(sock))) < 0) {
    sock_release(sock);
    return(-EINVAL);
  }

  return(fd);
}
```
`sock_socket()` 函数的第一个参数 `family` 就是指定要创建什么类型的socket, 因为在Linux系统中有着不同类型的socket, 譬如: `inet(网络版)` 和 `unix(本地版)`. 在上面的代码中, 首先遍历 `pops` 这个列表, `pops` 保存着不同类型的socket操作, 可以通过 `sock_register()` 函数注册一个socket操作. `sock_register()` 函数源码如下:
```cpp
#define NPROT  16

static struct proto_ops *pops[NPROTO];

int
sock_register(int family, struct proto_ops *ops)
{
  int i;

  cli();
  for(i = 0; i < NPROTO; i++) {
    if (pops[i] != NULL) continue;
    pops[i] = ops;
    pops[i]->family = family;
    sti();
    return(i);
  }
  sti();
  return(-ENOMEM);
}
```
不同类型的socket有着不同的操作, 所以内核需要定义一个结构来存储不同类型的socket对应的操作, `struct proto_ops` 结构体就是用来存储不同类型的socket对应的操作, 定义如下:
```cpp
struct proto_ops {
  int   family;

  int   (*create)   (struct socket *sock, int protocol);
  int   (*dup)      (struct socket *newsock, struct socket *oldsock);
  int   (*release)  (struct socket *sock, struct socket *peer);
  int   (*bind)     (struct socket *sock, struct sockaddr *umyaddr,
             int sockaddr_len);
  int   (*connect)  (struct socket *sock, struct sockaddr *uservaddr,
             int sockaddr_len, int flags);
  int   (*socketpair)   (struct socket *sock1, struct socket *sock2);
  int   (*accept)   (struct socket *sock, struct socket *newsock,
             int flags);
  int   (*getname)  (struct socket *sock, struct sockaddr *uaddr,
             int *usockaddr_len, int peer);
  int   (*read)     (struct socket *sock, char *ubuf, int size,
             int nonblock);
  int   (*write)    (struct socket *sock, char *ubuf, int size,
             int nonblock);
  int   (*select)   (struct socket *sock, int sel_type,
             select_table *wait);
  int   (*ioctl)    (struct socket *sock, unsigned int cmd,
             unsigned long arg);
  int   (*listen)   (struct socket *sock, int len);
  int   (*send)     (struct socket *sock, void *buff, int len, int nonblock,
             unsigned flags);
  int   (*recv)     (struct socket *sock, void *buff, int len, int nonblock,
             unsigned flags);
  int   (*sendto)   (struct socket *sock, void *buff, int len, int nonblock,
             unsigned flags, struct sockaddr *, int addr_len);
  int   (*recvfrom) (struct socket *sock, void *buff, int len, int nonblock,
             unsigned flags, struct sockaddr *, int *addr_len);
  int   (*shutdown) (struct socket *sock, int flags);
  int   (*setsockopt)   (struct socket *sock, int level, int optname,
             char *optval, int optlen);
  int   (*getsockopt)   (struct socket *sock, int level, int optname,
             char *optval, int *optlen);
  int   (*fcntl)    (struct socket *sock, unsigned int cmd,
             unsigned long arg);    
};
```
`struct proto_ops` 结构定义了很多函数指针, 譬如 `create`, `connect`, `bind` 等等. 学过面向对象编程的同学都知道 `接口` 这个概念, 这里使用的编程手段就有点像 `接口`. 为不同类型的socket实现 `struct proto_ops` 中的函数, 然后就可以通过相同的接口来调用不同的函数实现. 所以, 在Linux内核中使用了很多面向对象的思想.

在Linux系统初始化时会调用 `inet_proto_init()` 函数来注册 `inet` 类型的socket操作, 如下:
```cpp
static struct proto_ops inet_proto_ops = {
  AF_INET,

  inet_create,
  inet_dup,
  inet_release,
  inet_bind,
  inet_connect,
  inet_socketpair,
  inet_accept,
  inet_getname, 
  inet_read,
  inet_write,
  inet_select,
  inet_ioctl,
  inet_listen,
  inet_send,
  inet_recv,
  inet_sendto,
  inet_recvfrom,
  inet_shutdown,
  inet_setsockopt,
  inet_getsockopt,
  inet_fcntl,
};

void inet_proto_init(struct ddi_proto *pro)
{
  ...
  (void) sock_register(inet_proto_ops.family, &inet_proto_ops);
  ...
}
```
我们接着分析 `sock_socket()` 这个函数, 在找到 `family` 类型的socket操作后, 会调用 `sock_alloc()` 函数申请一个 `struct socket` 对象, 然后再调用socket操作的 `create()` 方法来初始化指定类型socket的上下文. 譬如 `inet` 类型的socket调用的就是 `inet_create()` 函数. 最后调用 `get_fd()` 来获取一个文件句柄来映射此socket.