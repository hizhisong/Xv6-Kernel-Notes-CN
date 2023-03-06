# fork & exec

板块: 内存管理

## shell中执行其他程序

在shell中执行ls:

Xv6 shell进程占据4个物理Page，当我们在shell中执行ls，shell fork一个子进程，fork干了这几件事：

![sysproc.c](fork%20&%20exec%209aa7484717d148fcb4caf0d5d595f7cc/Untitled.png)

sysproc.c

![Untitled](fork%20&%20exec%209aa7484717d148fcb4caf0d5d595f7cc/Untitled%201.png)

我们注意到新fork的进程在allocproc函数中新分配进程号和分配了trapframe、trampoline空间与映射，接着回到fork在内存模块子进程分配新空间将父进程内容赋值了过来，保证两者一致

父进程shell有4个Page，子进程也新建了4个Page

但这其实是一种很笨的方法，因为fork后常常还会exec删除内存重新分配内存载入新程序映像，前面分配空间复制父进程内存内容就很没必要

> 所以对于这个特定场景有一个非常有效的优化：当我们创建子进程时，与其创建，分配并拷贝内容到新的物理内存，其实我们可以直接共享父进程的物理内存page。所以这里，我们可以设置父子进程的只读PTE指向父进程对应的物理内存page。
> 

现代OS都会有类似[COW fork](Page%20Fault%2090304762266b455dac4b7e82e9b1a665.md)的功能，减少成本

# exec

```c
int
exec(char *path, char **argv)
{
  char *s, *last;
  int i, off;
  uint64 argc, sz = 0, sp, ustack[MAXARG+1], stackbase;
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pagetable_t pagetable = 0, oldpagetable;
  struct proc *p = myproc();

  begin_op();

  // 获取可执行文件inode

  if((ip = namei(path)) == 0){
    end_op();
    return -1;
  }
  ilock(ip);

  // 检查ELF文件合法性
  // Check ELF header
  if(readi(ip, 0, (uint64)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;
  if(elf.magic != ELF_MAGIC)
    goto bad;

  // 给进程分配一个仅map trampoline and trapframe的页表
  if((pagetable = proc_pagetable(p)) == 0)
    goto bad;

  // Load program into memory.
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    uint64 sz1;
    if((sz1 = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    sz = sz1;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
  iunlockput(ip);
  end_op();
  ip = 0;

  p = myproc();
  uint64 oldsz = p->sz;

  // Allocate two pages at the next page boundary.
  // Use the second as the user stack.
  // 两个PGSIZE大小 一个用作guard page(防止stack overflow伤及其他区域) 一个用于用户栈()
  sz = PGROUNDUP(sz);
  uint64 sz1;
  if((sz1 = uvmalloc(pagetable, sz, sz + 2*PGSIZE)) == 0)
    goto bad;
  sz = sz1;
  uvmclear(pagetable, sz-2*PGSIZE); // 栈和guard page标记为非法
  sp = sz;
  stackbase = sp - PGSIZE;  // 栈顶

  // Push argument strings, prepare rest of stack in ustack.
  // exec 参数复制到栈中并随后记录参数地址到全部参数后
  // 栈向低地址增长
  for(argc = 0; argv[argc]; argc++) {
    if(argc >= MAXARG)
      goto bad;
    sp -= strlen(argv[argc]) + 1;
    sp -= sp % 16; // riscv sp must be 16-byte aligned
    if(sp < stackbase)
      goto bad;
    if(copyout(pagetable, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      goto bad;
    ustack[argc] = sp;
  }
  ustack[argc] = 0;

  // push the array of argv[] pointers.
  sp -= (argc+1) * sizeof(uint64);
  sp -= sp % 16;
  if(sp < stackbase)
    goto bad;
  if(copyout(pagetable, sp, (char *)ustack, (argc+1)*sizeof(uint64)) < 0)
    goto bad;

  // arguments to user main(argc, argv)
  // argc is returned via the system call return
  // value, which goes in a0.
  p->trapframe->a1 = sp;

  // Save program name for debugging.
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(p->name, last, sizeof(p->name));
    
  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;   // 进程exec后用户空间(不计算trapframe and trampoline)初始大小为栈底地址值
  p->trapframe->epc = elf.entry;  // initial program counter = main       // 用户态PC(用户态program counter 指向正在执行的指令)
  p->trapframe->sp = sp; // initial stack pointer     // 在args参数之后
  proc_freepagetable(oldpagetable, oldsz);            // fork旧进程复制过来的虚拟地址空间被释放

  return argc; // this ends up in a0, the first argument to main(argc, argv)

 bad:
  if(pagetable)
    proc_freepagetable(pagetable, sz);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
}
```