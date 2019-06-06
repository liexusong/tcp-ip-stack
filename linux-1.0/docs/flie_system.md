# 文件系统分析

## 文件系统初始化
在系统初始化时 `init进程` 会调用 `setup()` 函数，而 `setup()` 函数会调用内核函数 `sys_setup()` 函数。`sys_setup()` 函数定义在 `drivers/block/genhd.c` 文件中，源码如下：
```c
asmlinkage int sys_setup(void * BIOS)
{
    ...
    mount_root();
    return (0);
}
```
`sys_setup()` 函数会调用 `mount_root()` 函数来读取根目录文件系统，源码如下：
```c
struct file_system_type file_systems[] = {
#ifdef CONFIG_MINIX_FS
	{minix_read_super,	"minix",	1},
#endif
#ifdef CONFIG_EXT_FS
	{ext_read_super,	"ext",		1},
#endif
#ifdef CONFIG_EXT2_FS
	{ext2_read_super,	"ext2",		1},
#endif
#ifdef CONFIG_XIA_FS
	{xiafs_read_super,	"xiafs",	1},
#endif
#ifdef CONFIG_MSDOS_FS
	{msdos_read_super,	"msdos",	1},
#endif
#ifdef CONFIG_PROC_FS
	{proc_read_super,	"proc",		0},
#endif
#ifdef CONFIG_NFS_FS
	{nfs_read_super,	"nfs",		0},
#endif
#ifdef CONFIG_ISO9660_FS
	{isofs_read_super,	"iso9660",	1},
#endif
#ifdef CONFIG_SYSV_FS
	{sysv_read_super,	"xenix",	1},
	{sysv_read_super,	"sysv",		1},
	{sysv_read_super,	"coherent",	1},
#endif
#ifdef CONFIG_HPFS_FS
	{hpfs_read_super,	"hpfs",		1},
#endif
	{NULL,			NULL,		0}
};

struct file_system_type *get_fs_type(char *name)
{
    int a;
    
    if (!name)
        return &file_systems[0];
    for(a = 0 ; file_systems[a].read_super ; a++)
        if (!strcmp(name,file_systems[a].name))
            return(&file_systems[a]);
    return NULL;
}

static struct super_block * read_super(dev_t dev,
        char *name,int flags, void *data, int silent)
{
    struct super_block * s;
    struct file_system_type *type;
    ...
    if (!(type = get_fs_type(name))) {
        return NULL;
    }
    ...
    s->s_dev = dev;
    s->s_flags = flags;
    if (!type->read_super(s,data, silent)) {
        s->s_dev = 0;
        return NULL;
    }
    s->s_dev = dev;
    s->s_covered = NULL;
    s->s_rd_only = 0;
    s->s_dirt = 0;
    return s;
}

void mount_root(void)
{
	struct file_system_type * fs_type;
	struct super_block * sb;
	struct inode * inode;
	...
	for (fs_type = file_systems; fs_type->read_super; fs_type++) {
		if (!fs_type->requires_dev)
			continue;
		sb = read_super(ROOT_DEV,fs_type->name,root_mountflags,NULL,1);
		if (sb) {
			inode = sb->s_mounted;
			inode->i_count += 3 ;	/* NOTE! it is logically used 4 times, not 1 */
			sb->s_covered = inode;
			sb->s_flags = root_mountflags;
			current->pwd = inode;
			current->root = inode;
			return;
		}
	}
	panic("VFS: Unable to mount root");
}
```
`mount_root()` 函数首先会遍历系统支持的所有文件系统，然后调用他们的 `read_super()` （譬如minix文件系统会调用minix_read_super()）方法来尝试读取 `根目录` 的文件系统超级块，如果 `read_super()` 方法返回一个超级块对象，表示读取成功，根目录使用此文件系统格式化的。然后把当前进程（`init进程`）的 `pwd` (当前工作目录) 和 `root` (根目录) 字段设置为根目录的 `inode` 节点。

## 打开文件操作
在用户态通过调用 `open()` 系统调用打开一个文件，而 `open()` 系统调用最终会进入内核态，并且调用 `sys_open()` 函数来处理。现在我们来分析一下 `sys_open()` 函数的处理过程：
```cpp
asmlinkage int sys_open(const char * filename,int flags,int mode)
{
    char * tmp;
    int error;

    error = getname(filename, &tmp);
    if (error)
        return error;
    error = do_open(tmp,flags,mode);
    putname(tmp);
    return error;
}
```
从代码可以看到，`sys_open()` 函数最后会调用 `do_open()` 函数处理：
```cpp
int do_open(const char * filename,int flags,int mode)
{
    struct inode * inode;
    struct file * f;
    int flag,error,fd;

    for(fd=0 ; fd<NR_OPEN ; fd++)
        if (!current->filp[fd])
            break;
    if (fd>=NR_OPEN)
        return -EMFILE;
    FD_CLR(fd,&current->close_on_exec);
    f = get_empty_filp();
    if (!f)
        return -ENFILE;
    current->filp[fd] = f;
    f->f_flags = flag = flags;
    f->f_mode = (flag+1) & O_ACCMODE;
    if (f->f_mode)
        flag++;
    if (flag & (O_TRUNC | O_CREAT))
        flag |= 2;
    error = open_namei(filename,flag,mode,&inode,NULL);
    if (error) {
        current->filp[fd]=NULL;
        f->f_count--;
        return error;
    }

    f->f_inode = inode;
    f->f_pos = 0;
    f->f_reada = 0;
    f->f_op = NULL;
    if (inode->i_op)
        f->f_op = inode->i_op->default_file_ops;
    if (f->f_op && f->f_op->open) {
        error = f->f_op->open(inode,f);
        if (error) {
            iput(inode);
            f->f_count--;
            current->filp[fd]=NULL;
            return error;
        }
    }
    f->f_flags &= ~(O_CREAT | O_EXCL | O_NOCTTY | O_TRUNC);
    return (fd);
}
```
`do_open()` 函数首先找到一个为被使用的文件fd，然后调用 `get_empty_filp()` 函数获取一个空闲的 `struct file` 结构，再调用 `open_namei()` 函数创建一个 `inode` 结构，最后通过调用指定文件系统的 `open()` 方法来打开文件。

上面的步骤中，最重要的是调用 `open_namei()` 函数，我们来分析一下 `open_namei()` 函数的实现，由于 `open_namei()` 比较复杂，所以我们分段来分析这个函数：
```cpp
int open_namei(const char * pathname, int flag, int mode,
    struct inode ** res_inode, struct inode * base)
{
    const char * basename;
    int namelen,error;
    struct inode * dir, *inode;
    struct task_struct ** p;

    mode &= S_IALLUGO & ~current->umask;
    mode |= S_IFREG;
    error = dir_namei(pathname,&namelen,&basename,base,&dir);
    if (error)
        return error;
```
`open_namei()` 函数首先通过调用 `dir_namei()` 函数来获得目录的 `inode` 结构，和放回文件名：
```cpp
static int dir_namei(const char * pathname, int * namelen, const char ** name,
    struct inode * base, struct inode ** res_inode)
{
    char c;
    const char * thisname;
    int len,error;
    struct inode * inode;

    *res_inode = NULL;
    if (!base) {
        base = current->pwd;
        base->i_count++;
    }
    if ((c = *pathname) == '/') { // 从根目录开始查找
        iput(base);
        base = current->root;
        pathname++;
        base->i_count++;
    }
    while (1) {
        thisname = pathname;
        for(len=0;(c = *(pathname++))&&(c != '/');len++) // 找到下一个目录
            /* nothing */ ;
        if (!c)
            break;
        base->i_count++;
        error = lookup(base,thisname,len,&inode); // 调用lookup()查找当前目录的inode
        if (error) {
            iput(base);
            return error;
        }
        error = follow_link(base,inode,0,0,&base); // 一般把base设置为inode
        if (error)
            return error;
    }
    if (!base->i_op || !base->i_op->lookup) {
        iput(base);
        return -ENOTDIR;
    }
    *name = thisname;  // 文件名
    *namelen = len;    // 文件名长度
    *res_inode = base; // 文件所在目录inode
    return 0;
}
```
`dir_namei()` 函数会沿着路径一步步查找，直到查找到最后一级目录，返最后一级目录的 `inode` 和文件名。

接着分析 `do_open()` 函数：
```cpp
    dir->i_count++;     /* lookup eats the dir */
    if (flag & O_CREAT) { // 如果是创建文件
        down(&dir->i_sem);
        error = lookup(dir,basename,namelen,&inode);
        if (!error) {
            if (flag & O_EXCL) { // 如果文件存在并且设置了O_EXCL标志, 返回错误
                iput(inode);
                error = -EEXIST;
            }
        } else if (!permission(dir,MAY_WRITE | MAY_EXEC)) // 如果目录没有写权限
            error = -EACCES;
        else if (!dir->i_op || !dir->i_op->create) // 没有创建文件的方法
            error = -EACCES;
        else if (IS_RDONLY(dir))
            error = -EROFS;
        else { // 创建文件
            dir->i_count++;     /* create eats the dir */
            error = dir->i_op->create(dir,basename,namelen,mode,res_inode);
            up(&dir->i_sem);
            iput(dir);
            return error;
        }
        up(&dir->i_sem);
    } else
        error = lookup(dir,basename,namelen,&inode);
```
上面的代码首先判断我们是否要创建文件，如果创建文件且文件已经存在，那么要检测是否设置了 `O_EXCL` 标志，如果设置了 `O_EXCL` 标志（文件不能存在），那么返回错误。然后检测目录是否有写权限，如果有就调用目录 `inode` 的 `create` （minix文件系统对应 `minix_create()` ）方法来创建文件，并且得到文件的 `inode` 结构。
