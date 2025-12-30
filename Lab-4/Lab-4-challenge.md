# 任务描述

- 在PKE操作系统内核中完善对文件相对路径的支撑
- 它能够实现通过相对路径对文件的访问
- 使得执行obj/app_relativepath时产生如下输出

```bash
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x000000008000d000, PKE kernel size: 0x000000000000d000 .
free physical memory address: [0x000000008000d000, 0x0000000087ffffff] 
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080008000
kernel page table is on 
RAMDISK0: base address of RAMDISK0 is: 0x0000000087f35000
RFS: format RAMDISK0 done!
Switch to user mode...
in alloc_proc. user frame 0x0000000087f29000, user stack 0x000000007ffff000, user kstack 0x0000000087f28000 
FS: created a file management struct for a process.
in alloc_proc. build proc_file_management successfully.
User application is loading.
Application: obj/app_relativepath
CODE_SEGMENT added at mapped info offset:3
DATA_SEGMENT added at mapped info offset:4
Application program entry point (virtual address): 0x00000000000100ec
going to insert process 0 to ready queue.
going to schedule process 0 to run.
======== Test 1: change current directory  ========
cwd:/
change current directory to ./RAMDISK0
cwd:/RAMDISK0
======== Test 2: write/read file by relative path  ========
write: ./ramfile
file descriptor fd: 0
write content: 
hello world
read: ./ramfile
read content: 
hello world
======== Test 3: Go to parent directory  ========
cwd:/RAMDISK0
change current directory to ..
cwd:/
read: ./hostfile.txt
file descriptor fd: 0
read content: 
This is an apple. 
Apples are good for our health. 
All tests passed!
User exit with code:0.
no more ready processes, system shutdown now.
System is shutting down with exit code 0.
```

# 代码文件

- strap.c
```c
/*
 * Utility functions for trap handling in Supervisor mode.
 */

#include "riscv.h"
#include "process.h"
#include "strap.h"
#include "syscall.h"
#include "pmm.h"
#include "vmm.h"
#include "sched.h"
#include "util/functions.h"

#include "spike_interface/spike_utils.h"

//
// handling the syscalls. will call do_syscall() defined in kernel/syscall.c
//
static void handle_syscall(trapframe *tf) {
  // tf->epc points to the address that our computer will jump to after the trap handling.
  // for a syscall, we should return to the NEXT instruction after its handling.
  // in RV64G, each instruction occupies exactly 32 bits (i.e., 4 Bytes)
  tf->epc += 4;

  // TODO (lab1_1): remove the panic call below, and call do_syscall (defined in
  // kernel/syscall.c) to conduct real operations of the kernel side for a syscall.
  // IMPORTANT: return value should be returned to user app, or else, you will encounter
  // problems in later experiments!
  panic( "call do_syscall to accomplish the syscall and lab1_1 here.\n" );

}

//
// global variable that store the recorded "ticks". added @lab1_3
static uint64 g_ticks = 0;
//
// added @lab1_3
//
void handle_mtimer_trap() {
  sprint("Ticks %d\n", g_ticks);
  // TODO (lab1_3): increase g_ticks to record this "tick", and then clear the "SIP"
  // field in sip register.
  // hint: use write_csr to disable the SIP_SSIP bit in sip.
  panic( "lab1_3: increase g_ticks by one, and clear SIP field in sip register.\n" );

}

//
// the page fault handler. added @lab2_3. parameters:
// sepc: the pc when fault happens;
// stval: the virtual address that causes pagefault when being accessed.
//
void handle_user_page_fault(uint64 mcause, uint64 sepc, uint64 stval) {
  sprint("handle_page_fault: %lx\n", stval);
  switch (mcause) {
    case CAUSE_STORE_PAGE_FAULT:
      // TODO (lab2_3): implement the operations that solve the page fault to
      // dynamically increase application stack.
      // hint: first allocate a new physical page, and then, maps the new page to the
      // virtual address that causes the page fault.
      panic( "You need to implement the operations that actually handle the page fault in lab2_3.\n" );

      break;
    default:
      sprint("unknown page fault.\n");
      break;
  }
}

//
// implements round-robin scheduling. added @lab3_3
//
void rrsched() {
  // TODO (lab3_3): implements round-robin scheduling.
  // hint: increase the tick_count member of current process by one, if it is bigger than
  // TIME_SLICE_LEN (means it has consumed its time slice), change its status into READY,
  // place it in the rear of ready queue, and finally schedule next process to run.
  panic( "You need to further implement the timer handling in lab3_3.\n" );

}

//
// kernel/smode_trap.S will pass control to smode_trap_handler, when a trap happens
// in S-mode.
//
void smode_trap_handler(void) {
  // make sure we are in User mode before entering the trap handling.
  // we will consider other previous case in lab1_3 (interrupt).
  if ((read_csr(sstatus) & SSTATUS_SPP) != 0) panic("usertrap: not from user mode");

  assert(current);
  // save user process counter.
  current->trapframe->epc = read_csr(sepc);

  // if the cause of trap is syscall from user application.
  // read_csr() and CAUSE_USER_ECALL are macros defined in kernel/riscv.h
  uint64 cause = read_csr(scause);

  // use switch-case instead of if-else, as there are many cases since lab2_3.
  switch (cause) {
    case CAUSE_USER_ECALL:
      handle_syscall(current->trapframe);
      break;
    case CAUSE_MTIMER_S_TRAP:
      handle_mtimer_trap();
      // invoke round-robin scheduler. added @lab3_3
      rrsched();
      break;
    case CAUSE_STORE_PAGE_FAULT:
    case CAUSE_LOAD_PAGE_FAULT:
      // the address of missing page is stored in stval
      // call handle_user_page_fault to process page faults
      handle_user_page_fault(cause, read_csr(sepc), read_csr(stval));
      break;
    default:
      sprint("smode_trap_handler(): unexpected scause %p\n", read_csr(scause));
      sprint("            sepc=%p stval=%p\n", read_csr(sepc), read_csr(stval));
      panic( "unexpected exception happened.\n" );
      break;
  }

  // continue (come back to) the execution of current process.
  switch_to(current);
}
```

- mtrap.c
```c
#include "kernel/riscv.h"
#include "kernel/process.h"
#include "spike_interface/spike_utils.h"

static void handle_instruction_access_fault() { panic("Instruction access fault!"); }

static void handle_load_access_fault() { panic("Load access fault!"); }

static void handle_store_access_fault() { panic("Store/AMO access fault!"); }

static void handle_illegal_instruction() { panic("Illegal instruction!"); }

static void handle_misaligned_load() { panic("Misaligned Load!"); }

static void handle_misaligned_store() { panic("Misaligned AMO!"); }

// added @lab1_3
static void handle_timer() {
  int cpuid = 0;
  // setup the timer fired at next time (TIMER_INTERVAL from now)
  *(uint64*)CLINT_MTIMECMP(cpuid) = *(uint64*)CLINT_MTIMECMP(cpuid) + TIMER_INTERVAL;

  // setup a soft interrupt in sip (S-mode Interrupt Pending) to be handled in S-mode
  write_csr(sip, SIP_SSIP);
}

//
// handle_mtrap calls a handling function according to the type of a machine mode interrupt (trap).
//
void handle_mtrap() {
  uint64 mcause = read_csr(mcause);
  switch (mcause) {
    case CAUSE_MTIMER:
      handle_timer();
      break;
    case CAUSE_FETCH_ACCESS:
      handle_instruction_access_fault();
      break;
    case CAUSE_LOAD_ACCESS:
      handle_load_access_fault();
    case CAUSE_STORE_ACCESS:
      handle_store_access_fault();
      break;
    case CAUSE_ILLEGAL_INSTRUCTION:
      // TODO (lab1_2): call handle_illegal_instruction to implement illegal instruction
      // interception, and finish lab1_2.
      panic( "call handle_illegal_instruction to accomplish illegal instruction interception for lab1_2.\n" );

      break;
    case CAUSE_MISALIGNED_LOAD:
      handle_misaligned_load();
      break;
    case CAUSE_MISALIGNED_STORE:
      handle_misaligned_store();
      break;

    default:
      sprint("machine trap(): unexpected mscause %p\n", mcause);
      sprint("            mepc=%p mtval=%p\n", read_csr(mepc), read_csr(mtval));
      panic( "unexpected exception happened in M-mode.\n" );
      break;
  }
}
```

- elf.c
```c
/*
 * routines that scan and load a (host) Executable and Linkable Format (ELF) file
 * into the (emulated) memory.
 */

#include "elf.h"
#include "string.h"
#include "riscv.h"
#include "vmm.h"
#include "pmm.h"
#include "spike_interface/spike_utils.h"

typedef struct elf_info_t {
  spike_file_t *f;
  process *p;
} elf_info;

//
// the implementation of allocater. allocates memory space for later segment loading.
// this allocater is heavily modified @lab2_1, where we do NOT work in bare mode.
//
static void *elf_alloc_mb(elf_ctx *ctx, uint64 elf_pa, uint64 elf_va, uint64 size) {
  elf_info *msg = (elf_info *)ctx->info;
  // we assume that size of proram segment is smaller than a page.
  kassert(size < PGSIZE);
  void *pa = alloc_page();
  if (pa == 0) panic("uvmalloc mem alloc falied\n");

  memset((void *)pa, 0, PGSIZE);
  user_vm_map((pagetable_t)msg->p->pagetable, elf_va, PGSIZE, (uint64)pa,
         prot_to_type(PROT_WRITE | PROT_READ | PROT_EXEC, 1));

  return pa;
}

//
// actual file reading, using the spike file interface.
//
static uint64 elf_fpread(elf_ctx *ctx, void *dest, uint64 nb, uint64 offset) {
  elf_info *msg = (elf_info *)ctx->info;
  // call spike file utility to load the content of elf file into memory.
  // spike_file_pread will read the elf file (msg->f) from offset to memory (indicated by
  // *dest) for nb bytes.
  return spike_file_pread(msg->f, dest, nb, offset);
}

//
// init elf_ctx, a data structure that loads the elf.
//
elf_status elf_init(elf_ctx *ctx, void *info) {
  ctx->info = info;

  // load the elf header
  if (elf_fpread(ctx, &ctx->ehdr, sizeof(ctx->ehdr), 0) != sizeof(ctx->ehdr)) return EL_EIO;

  // check the signature (magic value) of the elf
  if (ctx->ehdr.magic != ELF_MAGIC) return EL_NOTELF;

  return EL_OK;
}

//
// load the elf segments to memory regions.
//
elf_status elf_load(elf_ctx *ctx) {
  // elf_prog_header structure is defined in kernel/elf.h
  elf_prog_header ph_addr;
  int i, off;

  // traverse the elf program segment headers
  for (i = 0, off = ctx->ehdr.phoff; i < ctx->ehdr.phnum; i++, off += sizeof(ph_addr)) {
    // read segment headers
    if (elf_fpread(ctx, (void *)&ph_addr, sizeof(ph_addr), off) != sizeof(ph_addr)) return EL_EIO;

    if (ph_addr.type != ELF_PROG_LOAD) continue;
    if (ph_addr.memsz < ph_addr.filesz) return EL_ERR;
    if (ph_addr.vaddr + ph_addr.memsz < ph_addr.vaddr) return EL_ERR;

    // allocate memory block before elf loading
    void *dest = elf_alloc_mb(ctx, ph_addr.vaddr, ph_addr.vaddr, ph_addr.memsz);

    // actual loading
    if (elf_fpread(ctx, dest, ph_addr.memsz, ph_addr.off) != ph_addr.memsz)
      return EL_EIO;

    // record the vm region in proc->mapped_info. added @lab3_1
    int j;
    for( j=0; j<PGSIZE/sizeof(mapped_region); j++ ) //seek the last mapped region
      if( (process*)(((elf_info*)(ctx->info))->p)->mapped_info[j].va == 0x0 ) break;

    ((process*)(((elf_info*)(ctx->info))->p))->mapped_info[j].va = ph_addr.vaddr;
    ((process*)(((elf_info*)(ctx->info))->p))->mapped_info[j].npages = 1;

    // SEGMENT_READABLE, SEGMENT_EXECUTABLE, SEGMENT_WRITABLE are defined in kernel/elf.h
    if( ph_addr.flags == (SEGMENT_READABLE|SEGMENT_EXECUTABLE) ){
      ((process*)(((elf_info*)(ctx->info))->p))->mapped_info[j].seg_type = CODE_SEGMENT;
      sprint( "CODE_SEGMENT added at mapped info offset:%d\n", j );
    }else if ( ph_addr.flags == (SEGMENT_READABLE|SEGMENT_WRITABLE) ){
      ((process*)(((elf_info*)(ctx->info))->p))->mapped_info[j].seg_type = DATA_SEGMENT;
      sprint( "DATA_SEGMENT added at mapped info offset:%d\n", j );
    }else
      panic( "unknown program segment encountered, segment flag:%d.\n", ph_addr.flags );

    ((process*)(((elf_info*)(ctx->info))->p))->total_mapped_region ++;
  }

  return EL_OK;
}

typedef union {
  uint64 buf[MAX_CMDLINE_ARGS];
  char *argv[MAX_CMDLINE_ARGS];
} arg_buf;

//
// returns the number (should be 1) of string(s) after PKE kernel in command line.
// and store the string(s) in arg_bug_msg.
//
static size_t parse_args(arg_buf *arg_bug_msg) {
  // HTIFSYS_getmainvars frontend call reads command arguments to (input) *arg_bug_msg
  long r = frontend_syscall(HTIFSYS_getmainvars, (uint64)arg_bug_msg,
      sizeof(*arg_bug_msg), 0, 0, 0, 0, 0);
  kassert(r == 0);

  size_t pk_argc = arg_bug_msg->buf[0];
  uint64 *pk_argv = &arg_bug_msg->buf[1];

  int arg = 1;  // skip the PKE OS kernel string, leave behind only the application name
  for (size_t i = 0; arg + i < pk_argc; i++)
    arg_bug_msg->argv[i] = (char *)(uintptr_t)pk_argv[arg + i];

  //returns the number of strings after PKE kernel in command line
  return pk_argc - arg;
}

//
// load the elf of user application, by using the spike file interface.
//
void load_bincode_from_host_elf(process *p) {
  arg_buf arg_bug_msg;

  // retrieve command line arguements
  size_t argc = parse_args(&arg_bug_msg);
  if (!argc) panic("You need to specify the application program!\n");

  sprint("Application: %s\n", arg_bug_msg.argv[0]);

  //elf loading. elf_ctx is defined in kernel/elf.h, used to track the loading process.
  elf_ctx elfloader;
  // elf_info is defined above, used to tie the elf file and its corresponding process.
  elf_info info;

  info.f = spike_file_open(arg_bug_msg.argv[0], O_RDONLY, 0);
  info.p = p;
  // IS_ERR_VALUE is a macro defined in spike_interface/spike_htif.h
  if (IS_ERR_VALUE(info.f)) panic("Fail on openning the input application program.\n");

  // init elfloader context. elf_init() is defined above.
  if (elf_init(&elfloader, &info) != EL_OK)
    panic("fail to init elfloader.\n");

  // load elf. elf_load() is defined above.
  if (elf_load(&elfloader) != EL_OK) panic("Fail on loading elf.\n");

  // entry (virtual, also physical in lab1_x) address
  p->trapframe->epc = elfloader.ehdr.entry;

  // close the host spike file
  spike_file_close( info.f );

  sprint("Application program entry point (virtual address): 0x%lx\n", p->trapframe->epc);
}
```

- elf.h
```c
#ifndef _ELF_H_
#define _ELF_H_

#include "util/types.h"
#include "process.h"

#define MAX_CMDLINE_ARGS 64

// elf header structure
typedef struct elf_header_t {
  uint32 magic;
  uint8 elf[12];
  uint16 type;      /* Object file type */
  uint16 machine;   /* Architecture */
  uint32 version;   /* Object file version */
  uint64 entry;     /* Entry point virtual address */
  uint64 phoff;     /* Program header table file offset */
  uint64 shoff;     /* Section header table file offset */
  uint32 flags;     /* Processor-specific flags */
  uint16 ehsize;    /* ELF header size in bytes */
  uint16 phentsize; /* Program header table entry size */
  uint16 phnum;     /* Program header table entry count */
  uint16 shentsize; /* Section header table entry size */
  uint16 shnum;     /* Section header table entry count */
  uint16 shstrndx;  /* Section header string table index */
} elf_header;

// segment types, attributes of elf_prog_header_t.flags
#define SEGMENT_READABLE   0x4
#define SEGMENT_EXECUTABLE 0x1
#define SEGMENT_WRITABLE   0x2

// Program segment header.
typedef struct elf_prog_header_t {
  uint32 type;   /* Segment type */
  uint32 flags;  /* Segment flags */
  uint64 off;    /* Segment file offset */
  uint64 vaddr;  /* Segment virtual address */
  uint64 paddr;  /* Segment physical address */
  uint64 filesz; /* Segment size in file */
  uint64 memsz;  /* Segment size in memory */
  uint64 align;  /* Segment alignment */
} elf_prog_header;

#define ELF_MAGIC 0x464C457FU  // "\x7FELF" in little endian
#define ELF_PROG_LOAD 1

typedef enum elf_status_t {
  EL_OK = 0,

  EL_EIO,
  EL_ENOMEM,
  EL_NOTELF,
  EL_ERR,

} elf_status;

typedef struct elf_ctx_t {
  void *info;
  elf_header ehdr;
} elf_ctx;

elf_status elf_init(elf_ctx *ctx, void *info);
elf_status elf_load(elf_ctx *ctx);

void load_bincode_from_host_elf(process *p);

#endif
```

- syscall.c
```c
/*
 * contains the implementation of all syscalls.
 */

#include <stdint.h>
#include <errno.h>

#include "util/types.h"
#include "syscall.h"
#include "string.h"
#include "process.h"
#include "util/functions.h"
#include "pmm.h"
#include "vmm.h"
#include "sched.h"
#include "proc_file.h"

#include "spike_interface/spike_utils.h"

//
// implement the SYS_user_print syscall
//
ssize_t sys_user_print(const char* buf, size_t n) {
  // buf is now an address in user space of the given app's user stack,
  // so we have to transfer it into phisical address (kernel is running in direct mapping).
  assert( current );
  char* pa = (char*)user_va_to_pa((pagetable_t)(current->pagetable), (void*)buf);
  sprint(pa);
  return 0;
}

//
// implement the SYS_user_exit syscall
//
ssize_t sys_user_exit(uint64 code) {
  sprint("User exit with code:%d.\n", code);
  // reclaim the current process, and reschedule. added @lab3_1
  free_process( current );
  schedule();
  return 0;
}

//
// maybe, the simplest implementation of malloc in the world ... added @lab2_2
//
uint64 sys_user_allocate_page() {
  void* pa = alloc_page();
  uint64 va;
  // if there are previously reclaimed pages, use them first (this does not change the
  // size of the heap)
  if (current->user_heap.free_pages_count > 0) {
    va =  current->user_heap.free_pages_address[--current->user_heap.free_pages_count];
    assert(va < current->user_heap.heap_top);
  } else {
    // otherwise, allocate a new page (this increases the size of the heap by one page)
    va = current->user_heap.heap_top;
    current->user_heap.heap_top += PGSIZE;

    current->mapped_info[HEAP_SEGMENT].npages++;
  }
  user_vm_map((pagetable_t)current->pagetable, va, PGSIZE, (uint64)pa,
         prot_to_type(PROT_WRITE | PROT_READ, 1));

  return va;
}

//
// reclaim a page, indicated by "va". added @lab2_2
//
uint64 sys_user_free_page(uint64 va) {
  user_vm_unmap((pagetable_t)current->pagetable, va, PGSIZE, 1);
  // add the reclaimed page to the free page list
  current->user_heap.free_pages_address[current->user_heap.free_pages_count++] = va;
  return 0;
}

//
// kerenl entry point of naive_fork
//
ssize_t sys_user_fork() {
  sprint("User call fork.\n");
  return do_fork( current );
}

//
// kerenl entry point of yield. added @lab3_2
//
ssize_t sys_user_yield() {
  // TODO (lab3_2): implment the syscall of yield.
  // hint: the functionality of yield is to give up the processor. therefore,
  // we should set the status of currently running process to READY, insert it in
  // the rear of ready queue, and finally, schedule a READY process to run.
  panic( "You need to implement the yield syscall in lab3_2.\n" );

  return 0;
}

//
// open file
//
ssize_t sys_user_open(char *pathva, int flags) {
  char* pathpa = (char*)user_va_to_pa((pagetable_t)(current->pagetable), pathva);
  return do_open(pathpa, flags);
}

//
// read file
//
ssize_t sys_user_read(int fd, char *bufva, uint64 count) {
  int i = 0;
  while (i < count) { // count can be greater than page size
    uint64 addr = (uint64)bufva + i;
    uint64 pa = lookup_pa((pagetable_t)current->pagetable, addr);
    uint64 off = addr - ROUNDDOWN(addr, PGSIZE);
    uint64 len = count - i < PGSIZE - off ? count - i : PGSIZE - off;
    uint64 r = do_read(fd, (char *)pa + off, len);
    i += r; if (r < len) return i;
  }
  return count;
}

//
// write file
//
ssize_t sys_user_write(int fd, char *bufva, uint64 count) {
  int i = 0;
  while (i < count) { // count can be greater than page size
    uint64 addr = (uint64)bufva + i;
    uint64 pa = lookup_pa((pagetable_t)current->pagetable, addr);
    uint64 off = addr - ROUNDDOWN(addr, PGSIZE);
    uint64 len = count - i < PGSIZE - off ? count - i : PGSIZE - off;
    uint64 r = do_write(fd, (char *)pa + off, len);
    i += r; if (r < len) return i;
  }
  return count;
}

//
// lseek file
//
ssize_t sys_user_lseek(int fd, int offset, int whence) {
  return do_lseek(fd, offset, whence);
}

//
// read vinode
//
ssize_t sys_user_stat(int fd, struct istat *istat) {
  struct istat * pistat = (struct istat *)user_va_to_pa((pagetable_t)(current->pagetable), istat);
  return do_stat(fd, pistat);
}

//
// read disk inode
//
ssize_t sys_user_disk_stat(int fd, struct istat *istat) {
  struct istat * pistat = (struct istat *)user_va_to_pa((pagetable_t)(current->pagetable), istat);
  return do_disk_stat(fd, pistat);
}

//
// close file
//
ssize_t sys_user_close(int fd) {
  return do_close(fd);
}

//
// lib call to opendir
//
ssize_t sys_user_opendir(char * pathva){
  char * pathpa = (char*)user_va_to_pa((pagetable_t)(current->pagetable), pathva);
  return do_opendir(pathpa);
}

//
// lib call to readdir
//
ssize_t sys_user_readdir(int fd, struct dir *vdir){
  struct dir * pdir = (struct dir *)user_va_to_pa((pagetable_t)(current->pagetable), vdir);
  return do_readdir(fd, pdir);
}

//
// lib call to mkdir
//
ssize_t sys_user_mkdir(char * pathva){
  char * pathpa = (char*)user_va_to_pa((pagetable_t)(current->pagetable), pathva);
  return do_mkdir(pathpa);
}

//
// lib call to closedir
//
ssize_t sys_user_closedir(int fd){
  return do_closedir(fd);
}

//
// lib call to link
//
ssize_t sys_user_link(char * vfn1, char * vfn2){
  char * pfn1 = (char*)user_va_to_pa((pagetable_t)(current->pagetable), (void*)vfn1);
  char * pfn2 = (char*)user_va_to_pa((pagetable_t)(current->pagetable), (void*)vfn2);
  return do_link(pfn1, pfn2);
}

//
// lib call to unlink
//
ssize_t sys_user_unlink(char * vfn){
  char * pfn = (char*)user_va_to_pa((pagetable_t)(current->pagetable), (void*)vfn);
  return do_unlink(pfn);
}

//
// [a0]: the syscall number; [a1] ... [a7]: arguments to the syscalls.
// returns the code of success, (e.g., 0 means success, fail for otherwise)
//
long do_syscall(long a0, long a1, long a2, long a3, long a4, long a5, long a6, long a7) {
  switch (a0) {
    case SYS_user_print:
      return sys_user_print((const char*)a1, a2);
    case SYS_user_exit:
      return sys_user_exit(a1);
    // added @lab2_2
    case SYS_user_allocate_page:
      return sys_user_allocate_page();
    case SYS_user_free_page:
      return sys_user_free_page(a1);
    case SYS_user_fork:
      return sys_user_fork();
    case SYS_user_yield:
      return sys_user_yield();
    // added @lab4_1
    case SYS_user_open:
      return sys_user_open((char *)a1, a2);
    case SYS_user_read:
      return sys_user_read(a1, (char *)a2, a3);
    case SYS_user_write:
      return sys_user_write(a1, (char *)a2, a3);
    case SYS_user_lseek:
      return sys_user_lseek(a1, a2, a3);
    case SYS_user_stat:
      return sys_user_stat(a1, (struct istat *)a2);
    case SYS_user_disk_stat:
      return sys_user_disk_stat(a1, (struct istat *)a2);
    case SYS_user_close:
      return sys_user_close(a1);
    // added @lab4_2
    case SYS_user_opendir:
      return sys_user_opendir((char *)a1);
    case SYS_user_readdir:
      return sys_user_readdir(a1, (struct dir *)a2);
    case SYS_user_mkdir:
      return sys_user_mkdir((char *)a1);
    case SYS_user_closedir:
      return sys_user_closedir(a1);
    // added @lab4_3
    case SYS_user_link:
      return sys_user_link((char *)a1, (char *)a2);
    case SYS_user_unlink:
      return sys_user_unlink((char *)a1);
    default:
      panic("Unknown syscall %ld \n", a0);
  }
}
```

- syscall.h
```c
/*
 * define the syscall numbers of PKE OS kernel.
 */
#ifndef _SYSCALL_H_
#define _SYSCALL_H_

// syscalls of PKE OS kernel. append below if adding new syscalls.
#define SYS_user_base 64
#define SYS_user_print (SYS_user_base + 0)
#define SYS_user_exit (SYS_user_base + 1)
// added @lab2_2
#define SYS_user_allocate_page (SYS_user_base + 2)
#define SYS_user_free_page (SYS_user_base + 3)
// added @lab3_1
#define SYS_user_fork (SYS_user_base + 4)
#define SYS_user_yield (SYS_user_base + 5)
// added @lab4_1
#define SYS_user_open (SYS_user_base + 17)
#define SYS_user_read (SYS_user_base + 18)
#define SYS_user_write (SYS_user_base + 19)
#define SYS_user_lseek (SYS_user_base + 20)
#define SYS_user_stat (SYS_user_base + 21)
#define SYS_user_disk_stat (SYS_user_base + 22)
#define SYS_user_close (SYS_user_base + 23)
// added @lab4_2
#define SYS_user_opendir  (SYS_user_base + 24)
#define SYS_user_readdir  (SYS_user_base + 25)
#define SYS_user_mkdir    (SYS_user_base + 26)
#define SYS_user_closedir (SYS_user_base + 27)
// added @lab4_3
#define SYS_user_link   (SYS_user_base + 28)
#define SYS_user_unlink (SYS_user_base + 29)

long do_syscall(long a0, long a1, long a2, long a3, long a4, long a5, long a6, long a7);

#endif
```

- user_lib.c
```c
/*
 * The supporting library for applications.
 * Actually, supporting routines for applications are catalogued as the user 
 * library. we don't do that in PKE to make the relationship between application 
 * and user library more straightforward.
 */

#include "user_lib.h"
#include "util/types.h"
#include "util/snprintf.h"
#include "kernel/syscall.h"

uint64 do_user_call(uint64 sysnum, uint64 a1, uint64 a2, uint64 a3, uint64 a4, uint64 a5, uint64 a6,
                 uint64 a7) {
  int ret;

  // before invoking the syscall, arguments of do_user_call are already loaded into the argument
  // registers (a0-a7) of our (emulated) risc-v machine.
  asm volatile(
      "ecall\n"
      "sw a0, %0"  // returns a 32-bit value
      : "=m"(ret)
      :
      : "memory");

  return ret;
}

//
// printu() supports user/lab1_1_helloworld.c
//
int printu(const char* s, ...) {
  va_list vl;
  va_start(vl, s);

  char out[256];  // fixed buffer size.
  int res = vsnprintf(out, sizeof(out), s, vl);
  va_end(vl);
  const char* buf = out;
  size_t n = res < sizeof(out) ? res : sizeof(out);

  // make a syscall to implement the required functionality.
  return do_user_call(SYS_user_print, (uint64)buf, n, 0, 0, 0, 0, 0);
}

//
// applications need to call exit to quit execution.
//
int exit(int code) {
  return do_user_call(SYS_user_exit, code, 0, 0, 0, 0, 0, 0); 
}

//
// lib call to naive_malloc
//
void* naive_malloc() {
  return (void*)do_user_call(SYS_user_allocate_page, 0, 0, 0, 0, 0, 0, 0);
}

//
// lib call to naive_free
//
void naive_free(void* va) {
  do_user_call(SYS_user_free_page, (uint64)va, 0, 0, 0, 0, 0, 0);
}

//
// lib call to naive_fork
int fork() {
  return do_user_call(SYS_user_fork, 0, 0, 0, 0, 0, 0, 0);
}

//
// lib call to yield
//
void yield() {
  do_user_call(SYS_user_yield, 0, 0, 0, 0, 0, 0, 0);
}

//
// lib call to open
//
int open(const char *pathname, int flags) {
  return do_user_call(SYS_user_open, (uint64)pathname, flags, 0, 0, 0, 0, 0);
}

//
// lib call to read
//
int read_u(int fd, void * buf, uint64 count){
  return do_user_call(SYS_user_read, fd, (uint64)buf, count, 0, 0, 0, 0);
}

//
// lib call to write
//
int write_u(int fd, void *buf, uint64 count) {
  return do_user_call(SYS_user_write, fd, (uint64)buf, count, 0, 0, 0, 0);
}

//
// lib call to seek
// 
int lseek_u(int fd, int offset, int whence) {
  return do_user_call(SYS_user_lseek, fd, offset, whence, 0, 0, 0, 0);
}

//
// lib call to read file information
//
int stat_u(int fd, struct istat *istat) {
  return do_user_call(SYS_user_stat, fd, (uint64)istat, 0, 0, 0, 0, 0);
}

//
// lib call to read file information from disk
//
int disk_stat_u(int fd, struct istat *istat) {
  return do_user_call(SYS_user_disk_stat, fd, (uint64)istat, 0, 0, 0, 0, 0);
}

//
// lib call to open dir
//
int opendir_u(const char *dirname) {
  return do_user_call(SYS_user_opendir, (uint64)dirname, 0, 0, 0, 0, 0, 0);
}

//
// lib call to read dir
//
int readdir_u(int fd, struct dir *dir) {
  return do_user_call(SYS_user_readdir, fd, (uint64)dir, 0, 0, 0, 0, 0);
}

//
// lib call to make dir
//
int mkdir_u(const char *pathname) {
  return do_user_call(SYS_user_mkdir, (uint64)pathname, 0, 0, 0, 0, 0, 0);
}

//
// lib call to close dir
//
int closedir_u(int fd) {
  return do_user_call(SYS_user_closedir, fd, 0, 0, 0, 0, 0, 0);
} 

//
// lib call to link
//
int link_u(const char *fn1, const char *fn2){
  return do_user_call(SYS_user_link, (uint64)fn1, (uint64)fn2, 0, 0, 0, 0, 0);
}

//
// lib call to unlink
//
int unlink_u(const char *fn){
  return do_user_call(SYS_user_unlink, (uint64)fn, 0, 0, 0, 0, 0, 0);
}

//
// lib call to close
//
int close(int fd) {
  return do_user_call(SYS_user_close, fd, 0, 0, 0, 0, 0, 0);
}

//
// lib call to read present working directory (pwd)
//
int read_cwd(char *path) {
  return do_user_call(SYS_user_rcwd, (uint64)path, 0, 0, 0, 0, 0, 0);
}

//
// lib call to change pwd
//
int change_cwd(const char *path) {
  return do_user_call(SYS_user_ccwd, (uint64)path, 0, 0, 0, 0, 0, 0);
}
```

- user_lib.h
```c
/*
 * header file to be used by applications.
 */

#ifndef _USER_LIB_H_
#define _USER_LIB_H_
#include "util/types.h"
#include "kernel/proc_file.h"

int printu(const char *s, ...);
int exit(int code);
void* naive_malloc();
void naive_free(void* va);
int fork();
void yield();

// added @ lab4_1
int open(const char *pathname, int flags);
int read_u(int fd, void *buf, uint64 count);
int write_u(int fd, void *buf, uint64 count);
int lseek_u(int fd, int offset, int whence);
int stat_u(int fd, struct istat *istat);
int disk_stat_u(int fd, struct istat *istat);
int close(int fd);

// added @ lab4_2
int opendir_u(const char *pathname);
int readdir_u(int fd, struct dir *dir);
int mkdir_u(const char *pathname);
int closedir_u(int fd);

// added @ lab4_3
int link_u(const char *fn1, const char *fn2);
int unlink_u(const char *fn);

int read_cwd(char *path);
int change_cwd(const char *path);

#endif
```

- kernel.c
```c
/*
 * Supervisor-mode startup codes
 */

#include "riscv.h"
#include "string.h"
#include "elf.h"
#include "process.h"
#include "pmm.h"
#include "vmm.h"
#include "sched.h"
#include "memlayout.h"
#include "spike_interface/spike_utils.h"
#include "util/types.h"
#include "vfs.h"
#include "rfs.h"
#include "ramdev.h"

//
// trap_sec_start points to the beginning of S-mode trap segment (i.e., the entry point of
// S-mode trap vector). added @lab2_1
//
extern char trap_sec_start[];

//
// turn on paging. added @lab2_1
//
void enable_paging() {
  // write the pointer to kernel page (table) directory into the CSR of "satp".
  write_csr(satp, MAKE_SATP(g_kernel_pagetable));

  // refresh tlb to invalidate its content.
  flush_tlb();
}

//
// load the elf, and construct a "process" (with only a trapframe).
// load_bincode_from_host_elf is defined in elf.c
//
process* load_user_program() {
  process* proc;

  proc = alloc_process();
  sprint("User application is loading.\n");

  load_bincode_from_host_elf(proc);
  return proc;
}

//
// s_start: S-mode entry point of riscv-pke OS kernel.
//
int s_start(void) {
  sprint("Enter supervisor mode...\n");
  // in the beginning, we use Bare mode (direct) memory mapping as in lab1.
  // but now, we are going to switch to the paging mode @lab2_1.
  // note, the code still works in Bare mode when calling pmm_init() and kern_vm_init().
  write_csr(satp, 0);

  // init phisical memory manager
  pmm_init();

  // build the kernel page table
  kern_vm_init();

  // now, switch to paging mode by turning on paging (SV39)
  enable_paging();
  // the code now formally works in paging mode, meaning the page table is now in use.
  sprint("kernel page table is on \n");

  // added @lab3_1
  init_proc_pool();

  // init file system, added @lab4_1
  fs_init();

  sprint("Switch to user mode...\n");
  // the application code (elf) is first loaded into memory, and then put into execution
  // added @lab3_1
  insert_to_ready_queue( load_user_program() );
  schedule();

  // we should never reach here.
  return 0;
}
```

- process.c
```c
/*
 * Utility functions for process management. 
 *
 * Note: in Lab1, only one process (i.e., our user application) exists. Therefore, 
 * PKE OS at this stage will set "current" to the loaded user application, and also
 * switch to the old "current" process after trap handling.
 */

#include "riscv.h"
#include "strap.h"
#include "config.h"
#include "process.h"
#include "elf.h"
#include "string.h"
#include "vmm.h"
#include "pmm.h"
#include "memlayout.h"
#include "sched.h"
#include "spike_interface/spike_utils.h"

//Two functions defined in kernel/usertrap.S
extern char smode_trap_vector[];
extern void return_to_user(trapframe *, uint64 satp);

// trap_sec_start points to the beginning of S-mode trap segment (i.e., the entry point
// of S-mode trap vector).
extern char trap_sec_start[];

// process pool. added @lab3_1
process procs[NPROC];

// current points to the currently running user-mode application.
process* current = NULL;

//
// switch to a user-mode process
//
void switch_to(process* proc) {
  assert(proc);
  current = proc;

  // write the smode_trap_vector (64-bit func. address) defined in kernel/strap_vector.S
  // to the stvec privilege register, such that trap handler pointed by smode_trap_vector
  // will be triggered when an interrupt occurs in S mode.
  write_csr(stvec, (uint64)smode_trap_vector);

  // set up trapframe values (in process structure) that smode_trap_vector will need when
  // the process next re-enters the kernel.
  proc->trapframe->kernel_sp = proc->kstack;      // process's kernel stack
  proc->trapframe->kernel_satp = read_csr(satp);  // kernel page table
  proc->trapframe->kernel_trap = (uint64)smode_trap_handler;

  // SSTATUS_SPP and SSTATUS_SPIE are defined in kernel/riscv.h
  // set S Previous Privilege mode (the SSTATUS_SPP bit in sstatus register) to User mode.
  unsigned long x = read_csr(sstatus);
  x &= ~SSTATUS_SPP;  // clear SPP to 0 for user mode
  x |= SSTATUS_SPIE;  // enable interrupts in user mode

  // write x back to 'sstatus' register to enable interrupts, and sret destination mode.
  write_csr(sstatus, x);

  // set S Exception Program Counter (sepc register) to the elf entry pc.
  write_csr(sepc, proc->trapframe->epc);

  // make user page table. macro MAKE_SATP is defined in kernel/riscv.h. added @lab2_1
  uint64 user_satp = MAKE_SATP(proc->pagetable);

  // return_to_user() is defined in kernel/strap_vector.S. switch to user mode with sret.
  // note, return_to_user takes two parameters @ and after lab2_1.
  return_to_user(proc->trapframe, user_satp);
}

//
// initialize process pool (the procs[] array). added @lab3_1
//
void init_proc_pool() {
  memset( procs, 0, sizeof(process)*NPROC );

  for (int i = 0; i < NPROC; ++i) {
    procs[i].status = FREE;
    procs[i].pid = i;
  }
}

//
// allocate an empty process, init its vm space. returns the pointer to
// process strcuture. added @lab3_1
//
process* alloc_process() {
  // locate the first usable process structure
  int i;

  for( i=0; i<NPROC; i++ )
    if( procs[i].status == FREE ) break;

  if( i>=NPROC ){
    panic( "cannot find any free process structure.\n" );
    return 0;
  }

  // init proc[i]'s vm space
  procs[i].trapframe = (trapframe *)alloc_page();  //trapframe, used to save context
  memset(procs[i].trapframe, 0, sizeof(trapframe));

  // page directory
  procs[i].pagetable = (pagetable_t)alloc_page();
  memset((void *)procs[i].pagetable, 0, PGSIZE);

  procs[i].kstack = (uint64)alloc_page() + PGSIZE;   //user kernel stack top
  uint64 user_stack = (uint64)alloc_page();       //phisical address of user stack bottom
  procs[i].trapframe->regs.sp = USER_STACK_TOP;  //virtual address of user stack top

  // allocates a page to record memory regions (segments)
  procs[i].mapped_info = (mapped_region*)alloc_page();
  memset( procs[i].mapped_info, 0, PGSIZE );

  // map user stack in userspace
  user_vm_map((pagetable_t)procs[i].pagetable, USER_STACK_TOP - PGSIZE, PGSIZE,
    user_stack, prot_to_type(PROT_WRITE | PROT_READ, 1));
  procs[i].mapped_info[STACK_SEGMENT].va = USER_STACK_TOP - PGSIZE;
  procs[i].mapped_info[STACK_SEGMENT].npages = 1;
  procs[i].mapped_info[STACK_SEGMENT].seg_type = STACK_SEGMENT;

  // map trapframe in user space (direct mapping as in kernel space).
  user_vm_map((pagetable_t)procs[i].pagetable, (uint64)procs[i].trapframe, PGSIZE,
    (uint64)procs[i].trapframe, prot_to_type(PROT_WRITE | PROT_READ, 0));
  procs[i].mapped_info[CONTEXT_SEGMENT].va = (uint64)procs[i].trapframe;
  procs[i].mapped_info[CONTEXT_SEGMENT].npages = 1;
  procs[i].mapped_info[CONTEXT_SEGMENT].seg_type = CONTEXT_SEGMENT;

  // map S-mode trap vector section in user space (direct mapping as in kernel space)
  // we assume that the size of usertrap.S is smaller than a page.
  user_vm_map((pagetable_t)procs[i].pagetable, (uint64)trap_sec_start, PGSIZE,
    (uint64)trap_sec_start, prot_to_type(PROT_READ | PROT_EXEC, 0));
  procs[i].mapped_info[SYSTEM_SEGMENT].va = (uint64)trap_sec_start;
  procs[i].mapped_info[SYSTEM_SEGMENT].npages = 1;
  procs[i].mapped_info[SYSTEM_SEGMENT].seg_type = SYSTEM_SEGMENT;

  sprint("in alloc_proc. user frame 0x%lx, user stack 0x%lx, user kstack 0x%lx \n",
    procs[i].trapframe, procs[i].trapframe->regs.sp, procs[i].kstack);

  // initialize the process's heap manager
  procs[i].user_heap.heap_top = USER_FREE_ADDRESS_START;
  procs[i].user_heap.heap_bottom = USER_FREE_ADDRESS_START;
  procs[i].user_heap.free_pages_count = 0;

  // map user heap in userspace
  procs[i].mapped_info[HEAP_SEGMENT].va = USER_FREE_ADDRESS_START;
  procs[i].mapped_info[HEAP_SEGMENT].npages = 0;  // no pages are mapped to heap yet.
  procs[i].mapped_info[HEAP_SEGMENT].seg_type = HEAP_SEGMENT;

  procs[i].total_mapped_region = 4;

  // initialize files_struct
  procs[i].pfiles = init_proc_file_management();
  sprint("in alloc_proc. build proc_file_management successfully.\n");

  // return after initialization.
  return &procs[i];
}

//
// reclaim a process. added @lab3_1
//
int free_process( process* proc ) {
  // we set the status to ZOMBIE, but cannot destruct its vm space immediately.
  // since proc can be current process, and its user kernel stack is currently in use!
  // but for proxy kernel, it (memory leaking) may NOT be a really serious issue,
  // as it is different from regular OS, which needs to run 7x24.
  proc->status = ZOMBIE;

  return 0;
}

//
// implements fork syscal in kernel. added @lab3_1
// basic idea here is to first allocate an empty process (child), then duplicate the
// context and data segments of parent process to the child, and lastly, map other
// segments (code, system) of the parent to child. the stack segment remains unchanged
// for the child.
//
int do_fork( process* parent)
{
  sprint( "will fork a child from parent %d.\n", parent->pid );
  process* child = alloc_process();

  for( int i=0; i<parent->total_mapped_region; i++ ){
    // browse parent's vm space, and copy its trapframe and data segments,
    // map its code segment.
    switch( parent->mapped_info[i].seg_type ){
      case CONTEXT_SEGMENT:
        *child->trapframe = *parent->trapframe;
        break;
      case STACK_SEGMENT:
        memcpy( (void*)lookup_pa(child->pagetable, child->mapped_info[STACK_SEGMENT].va),
          (void*)lookup_pa(parent->pagetable, parent->mapped_info[i].va), PGSIZE );
        break;
      case HEAP_SEGMENT:
        // build a same heap for child process.

        // convert free_pages_address into a filter to skip reclaimed blocks in the heap
        // when mapping the heap blocks
        int free_block_filter[MAX_HEAP_PAGES];
        memset(free_block_filter, 0, MAX_HEAP_PAGES);
        uint64 heap_bottom = parent->user_heap.heap_bottom;
        for (int i = 0; i < parent->user_heap.free_pages_count; i++) {
          int index = (parent->user_heap.free_pages_address[i] - heap_bottom) / PGSIZE;
          free_block_filter[index] = 1;
        }

        // copy and map the heap blocks
        for (uint64 heap_block = current->user_heap.heap_bottom;
             heap_block < current->user_heap.heap_top; heap_block += PGSIZE) {
          if (free_block_filter[(heap_block - heap_bottom) / PGSIZE])  // skip free blocks
            continue;

          void* child_pa = alloc_page();
          memcpy(child_pa, (void*)lookup_pa(parent->pagetable, heap_block), PGSIZE);
          user_vm_map((pagetable_t)child->pagetable, heap_block, PGSIZE, (uint64)child_pa,
                      prot_to_type(PROT_WRITE | PROT_READ, 1));
        }

        child->mapped_info[HEAP_SEGMENT].npages = parent->mapped_info[HEAP_SEGMENT].npages;

        // copy the heap manager from parent to child
        memcpy((void*)&child->user_heap, (void*)&parent->user_heap, sizeof(parent->user_heap));
        break;
      case CODE_SEGMENT:
        // TODO (lab3_1): implment the mapping of child code segment to parent's
        // code segment.
        // hint: the virtual address mapping of code segment is tracked in mapped_info
        // page of parent's process structure. use the information in mapped_info to
        // retrieve the virtual to physical mapping of code segment.
        // after having the mapping information, just map the corresponding virtual
        // address region of child to the physical pages that actually store the code
        // segment of parent process.
        // DO NOT COPY THE PHYSICAL PAGES, JUST MAP THEM.
        panic( "You need to implement the code segment mapping of child in lab3_1.\n" );

        // after mapping, register the vm region (do not delete codes below!)
        child->mapped_info[child->total_mapped_region].va = parent->mapped_info[i].va;
        child->mapped_info[child->total_mapped_region].npages =
          parent->mapped_info[i].npages;
        child->mapped_info[child->total_mapped_region].seg_type = CODE_SEGMENT;
        child->total_mapped_region++;
        break;
    }
  }

  child->status = READY;
  child->trapframe->regs.a0 = 0;
  child->parent = parent;
  insert_to_ready_queue( child );

  return child->pid;
}
```

- process.h
```c
#ifndef _PROC_H_
#define _PROC_H_

#include "riscv.h"
#include "proc_file.h"

typedef struct trapframe_t {
  // space to store context (all common registers)
  /* offset:0   */ riscv_regs regs;

  // process's "user kernel" stack
  /* offset:248 */ uint64 kernel_sp;
  // pointer to smode_trap_handler
  /* offset:256 */ uint64 kernel_trap;
  // saved user process counter
  /* offset:264 */ uint64 epc;

  // kernel page table. added @lab2_1
  /* offset:272 */ uint64 kernel_satp;
}trapframe;

// riscv-pke kernel supports at most 32 processes
#define NPROC 32
// maximum number of pages in a process's heap
#define MAX_HEAP_PAGES 32

// possible status of a process
enum proc_status {
  FREE,            // unused state
  READY,           // ready state
  RUNNING,         // currently running
  BLOCKED,         // waiting for something
  ZOMBIE,          // terminated but not reclaimed yet
};

// types of a segment
enum segment_type {
  STACK_SEGMENT = 0,   // runtime stack segment
  CONTEXT_SEGMENT, // trapframe segment
  SYSTEM_SEGMENT,  // system segment
  HEAP_SEGMENT,    // runtime heap segment
  CODE_SEGMENT,    // ELF segment
  DATA_SEGMENT,    // ELF segment
};

// the VM regions mapped to a user process
typedef struct mapped_region {
  uint64 va;       // mapped virtual address
  uint32 npages;   // mapping_info is unused if npages == 0
  uint32 seg_type; // segment type, one of the segment_types
} mapped_region;

typedef struct process_heap_manager {
  // points to the last free page in our simple heap.
  uint64 heap_top;
  // points to the bottom of our simple heap.
  uint64 heap_bottom;

  // the address of free pages in the heap
  uint64 free_pages_address[MAX_HEAP_PAGES];
  // the number of free pages in the heap
  uint32 free_pages_count;
}process_heap_manager;

// the extremely simple definition of process, used for begining labs of PKE
typedef struct process_t {
  // pointing to the stack used in trap handling.
  uint64 kstack;
  // user page table
  pagetable_t pagetable;
  // trapframe storing the context of a (User mode) process.
  trapframe* trapframe;

  // points to a page that contains mapped_regions. below are added @lab3_1
  mapped_region *mapped_info;
  // next free mapped region in mapped_info
  int total_mapped_region;

  // heap management
  process_heap_manager user_heap;

  // process id
  uint64 pid;
  // process status
  int status;
  // parent process
  struct process_t *parent;
  // next queue element
  struct process_t *queue_next;

  // accounting. added @lab3_3
  int tick_count;

  // file system. added @lab4_1
  proc_file_management *pfiles;
}process;

// switch to run user app
void switch_to(process*);

// initialize process pool (the procs[] array)
void init_proc_pool();
// allocate an empty process, init its vm space. returns its pid
process* alloc_process();
// reclaim a process, destruct its vm space and free physical pages.
int free_process( process* proc );
// fork a child from parent
int do_fork(process* parent);

// current running process
extern process* current;

#endif
```

- riscv.h
```c
#ifndef _RISCV_H_
#define _RISCV_H_

#include "util/types.h"
#include "config.h"

// fields of mstatus, the Machine mode Status register
#define MSTATUS_MPP_MASK (3L << 11) // previous mode mask
#define MSTATUS_MPP_M (3L << 11)    // machine mode (m-mode)
#define MSTATUS_MPP_S (1L << 11)    // supervisor mode (s-mode)
#define MSTATUS_MPP_U (0L << 11)    // user mode (u-mode)
#define MSTATUS_MIE (1L << 3)       // machine-mode interrupt enable
#define MSTATUS_MPIE (1L << 7)      // preserve MIE bit

// values of mcause, the Machine Cause register
#define IRQ_S_EXT 9                 // s-mode external interrupt
#define IRQ_S_TIMER 5               // s-mode timer interrupt
#define IRQ_S_SOFT 1                // s-mode software interrupt
#define IRQ_M_SOFT 3                // m-mode software interrupt

// fields of mip, the Machine Interrupt Pending register
#define MIP_SEIP (1 << IRQ_S_EXT)   // s-mode external interrupt pending
#define MIP_SSIP (1 << IRQ_S_SOFT)  // s-mode software interrupt pending
#define MIP_STIP (1 << IRQ_S_TIMER) // s-mode timer interrupt pending
#define MIP_MSIP (1 << IRQ_M_SOFT)  // m-mode software interrupt pending

// pysical memory protection choices
#define PMP_R 0x01
#define PMP_W 0x02
#define PMP_X 0x04
#define PMP_A 0x18
#define PMP_L 0x80
#define PMP_SHIFT 2

#define PMP_TOR 0x08
#define PMP_NA4 0x10
#define PMP_NAPOT 0x18

// exceptions
#define CAUSE_MISALIGNED_FETCH 0x0     // Instruction address misaligned
#define CAUSE_FETCH_ACCESS 0x1         // Instruction access fault
#define CAUSE_ILLEGAL_INSTRUCTION 0x2  // Illegal Instruction
#define CAUSE_BREAKPOINT 0x3           // Breakpoint
#define CAUSE_MISALIGNED_LOAD 0x4      // Load address misaligned
#define CAUSE_LOAD_ACCESS 0x5          // Load access fault
#define CAUSE_MISALIGNED_STORE 0x6     // Store/AMO address misaligned
#define CAUSE_STORE_ACCESS 0x7         // Store/AMO access fault
#define CAUSE_USER_ECALL 0x8           // Environment call from U-mode
#define CAUSE_SUPERVISOR_ECALL 0x9     // Environment call from S-mode
#define CAUSE_MACHINE_ECALL 0xb        // Environment call from M-mode
#define CAUSE_FETCH_PAGE_FAULT 0xc     // Instruction page fault
#define CAUSE_LOAD_PAGE_FAULT 0xd      // Load page fault
#define CAUSE_STORE_PAGE_FAULT 0xf     // Store/AMO page fault

// irqs (interrupts). added @lab1_3
#define CAUSE_MTIMER 0x8000000000000007
#define CAUSE_MTIMER_S_TRAP 0x8000000000000001

//Supervisor interrupt-pending register
#define SIP_SSIP (1L << 1)

// core local interruptor (CLINT), which contains the timer.
#define CLINT 0x2000000L
#define CLINT_MTIMECMP(hartid) (CLINT + 0x4000 + 8 * (hartid))
#define CLINT_MTIME (CLINT + 0xBFF8)  // cycles since boot.

// fields of sstatus, the Supervisor mode Status register
#define SSTATUS_SPP (1L << 8)   // Previous mode, 1=Supervisor, 0=User
#define SSTATUS_SPIE (1L << 5)  // Supervisor Previous Interrupt Enable
#define SSTATUS_UPIE (1L << 4)  // User Previous Interrupt Enable
#define SSTATUS_SIE (1L << 1)   // Supervisor Interrupt Enable
#define SSTATUS_UIE (1L << 0)   // User Interrupt Enable
#define SSTATUS_SUM 0x00040000
#define SSTATUS_FS 0x00006000

// Supervisor Interrupt Enable
#define SIE_SEIE (1L << 9)  // external
#define SIE_STIE (1L << 5)  // timer
#define SIE_SSIE (1L << 1)  // software

// Machine-mode Interrupt Enable
#define MIE_MEIE (1L << 11)  // external
#define MIE_MTIE (1L << 7)   // timer
#define MIE_MSIE (1L << 3)   // software

#define read_const_csr(reg)              \
  ({                                     \
    unsigned long __tmp;                 \
    asm("csrr %0, " #reg : "=r"(__tmp)); \
    __tmp;                               \
  })

static inline int supports_extension(char ext) {
  return read_const_csr(misa) & (1 << (ext - 'A'));
}

#define read_csr(reg)                             \
  ({                                              \
    unsigned long __tmp;                          \
    asm volatile("csrr %0, " #reg : "=r"(__tmp)); \
    __tmp;                                        \
  })

#define write_csr(reg, val) ({ asm volatile("csrw " #reg ", %0" ::"rK"(val)); })

#define swap_csr(reg, val)                                            \
  ({                                                                  \
    unsigned long __tmp;                                              \
    asm volatile("csrrw %0, " #reg ", %1" : "=r"(__tmp) : "rK"(val)); \
    __tmp;                                                            \
  })

#define set_csr(reg, bit)                                             \
  ({                                                                  \
    unsigned long __tmp;                                              \
    asm volatile("csrrs %0, " #reg ", %1" : "=r"(__tmp) : "rK"(bit)); \
    __tmp;                                                            \
  })

// enable device interrupts
static inline void intr_on(void) { write_csr(sstatus, read_csr(sstatus) | SSTATUS_SIE); }

// disable device interrupts
static inline void intr_off(void) { write_csr(sstatus, read_csr(sstatus) & ~SSTATUS_SIE); }

// are device interrupts enabled?
static inline int is_intr_enable(void) {
  //  uint64 x = r_sstatus();
  uint64 x = read_csr(sstatus);
  return (x & SSTATUS_SIE) != 0;
}

// read sp, the stack pointer
static inline uint64 read_sp(void) {
  uint64 x;
  asm volatile("mv %0, sp" : "=r"(x));
  return x;
}

// read tp, the thread pointer, holding hartid (core number), the index into cpus[].
static inline uint64 read_tp(void) {
  uint64 x;
  asm volatile("mv %0, tp" : "=r"(x));
  return x;
}

// write tp, the thread pointer, holding hartid (core number), the index into cpus[].
static inline void write_tp(uint64 x) { asm volatile("mv tp, %0" : : "r"(x)); }

typedef struct riscv_regs_t {
  /*  0  */ uint64 ra;
  /*  8  */ uint64 sp;
  /*  16 */ uint64 gp;
  /*  24 */ uint64 tp;
  /*  32 */ uint64 t0;
  /*  40 */ uint64 t1;
  /*  48 */ uint64 t2;
  /*  56 */ uint64 s0;
  /*  64 */ uint64 s1;
  /*  72 */ uint64 a0;
  /*  80 */ uint64 a1;
  /*  88 */ uint64 a2;
  /*  96 */ uint64 a3;
  /* 104 */ uint64 a4;
  /* 112 */ uint64 a5;
  /* 120 */ uint64 a6;
  /* 128 */ uint64 a7;
  /* 136 */ uint64 s2;
  /* 144 */ uint64 s3;
  /* 152 */ uint64 s4;
  /* 160 */ uint64 s5;
  /* 168 */ uint64 s6;
  /* 176 */ uint64 s7;
  /* 184 */ uint64 s8;
  /* 192 */ uint64 s9;
  /* 196 */ uint64 s10;
  /* 208 */ uint64 s11;
  /* 216 */ uint64 t3;
  /* 224 */ uint64 t4;
  /* 232 */ uint64 t5;
  /* 240 */ uint64 t6;
}riscv_regs;

// following lines are added @lab2_1
static inline void flush_tlb(void) { asm volatile("sfence.vma zero, zero"); }
#define PGSIZE 4096  // bytes per page
#define PGSHIFT 12   // offset bits within a page

// use riscv's sv39 page table scheme.
#define SATP_SV39 (8L << 60)
#define MAKE_SATP(pagetable) (SATP_SV39 | (((uint64)pagetable) >> 12))

#define PTE_V (1L << 0)  // valid
#define PTE_R (1L << 1)  // readable
#define PTE_W (1L << 2)  // writable
#define PTE_X (1L << 3)  // executable
#define PTE_U (1L << 4)  // 1->user can access, 0->otherwise
#define PTE_G (1L << 5)  // global
#define PTE_A (1L << 6)  // accessed
#define PTE_D (1L << 7)  // dirty

// shift a physical address to the right place for a PTE.
#define PA2PTE(pa) ((((uint64)pa) >> 12) << 10)

// convert a pte content into its corresponding physical address
#define PTE2PA(pte) (((pte) >> 10) << 12)

// extract the property bits of a pte
#define PTE_FLAGS(pte) ((pte)&0x3FF)

// extract the three 9-bit page table indices from a virtual address.
#define PXMASK 0x1FF  // 9 bits

#define PXSHIFT(level) (PGSHIFT + (9 * (level)))
#define PX(level, va) ((((uint64)(va)) >> PXSHIFT(level)) & PXMASK)

// one beyond the highest possible virtual address.
// MAXVA is actually one bit less than the max allowed by
// Sv39, to avoid having to sign-extend virtual addresses
// that have the high bit set.
#define MAXVA (1L << (9 + 9 + 9 + 12 - 1))

typedef uint64 pte_t;
typedef uint64 *pagetable_t;  // 512 PTEs

#endif
```

- strap.h
```c
#ifndef _STRAP_H_
#define _STRAP_H_

void smode_trap_handler(void);

#endif
```

- strap_vector.S
```S
.section trapsec
.globl trap_sec_start
trap_sec_start:

#include "util/load_store.S"

#
# When a trap (e.g., a syscall from User mode in this lab) happens and the computer
# enters the Supervisor mode, the computer will continue to execute the following
# function (smode_trap_vector) to actually handle the trap.
#
# NOTE: sscratch points to the trapframe of current process before entering
# smode_trap_vector. It is done by reture_to_user function (defined below) when
# scheduling a user-mode application to run.
#
.globl smode_trap_vector
.align 4
smode_trap_vector:
    # swap a0 and sscratch, so that points a0 to the trapframe of current process
    csrrw a0, sscratch, a0

    # save the context (user registers) of current process in its trapframe.
    addi t6, a0 , 0

    # store_all_registers is a macro defined in util/load_store.S, it stores contents
    # of all general purpose registers into a piece of memory started from [t6].
    store_all_registers

    # come back to save a0 register before entering trap handling in trapframe
    # [t0]=[sscratch]
    csrr t0, sscratch
    sd t0, 72(a0)

    # use the "user kernel" stack (whose pointer stored in p->trapframe->kernel_sp)
    ld sp, 248(a0)

    # load the address of smode_trap_handler() from p->trapframe->kernel_trap
    ld t0, 256(a0)

    # restore kernel page table from p->trapframe->kernel_satp. added @lab2_1
    ld t1, 272(a0)
    csrw satp, t1
    sfence.vma zero, zero

    # jump to smode_trap_handler() that is defined in kernel/trap.c
    jr t0

#
# return from Supervisor mode to User mode, transition is made by using a trapframe,
# which stores the context of a user application.
# return_to_user() takes one parameter, i.e., the pointer (a0 register) pointing to a
# trapframe (defined in kernel/process.h) of the process.
#
.globl return_to_user
return_to_user:
    # a0: TRAPFRAME
    # a1: user page table, for satp.

    # switch to the user page table. added @lab2_1
    csrw satp, a1
    sfence.vma zero, zero

    # [sscratch]=[a0], save a0 in sscratch, so sscratch points to a trapframe now.
    csrw sscratch, a0

    # let [t6]=[a0]
    addi t6, a0, 0

    # restore_all_registers is a assembly macro defined in util/load_store.S.
    # the macro restores all registers from trapframe started from [t6] to all general
    # purpose registers, so as to resort the execution of a process.
    restore_all_registers 

    # return to user mode and user pc.
    sret
```

- minit.c
```c
/*
 * Machine-mode C startup codes
 */

#include "util/types.h"
#include "kernel/riscv.h"
#include "kernel/config.h"
#include "spike_interface/spike_utils.h"

//
// global variables are placed in the .data section.
// stack0 is the privilege mode stack(s) of the proxy kernel on CPU(s)
// allocates 4KB stack space for each processor (hart)
//
// NCPU is defined to be 1 in kernel/config.h, as we consider only one HART in basic
// labs.
//
__attribute__((aligned(16))) char stack0[4096 * NCPU];

// sstart() is the supervisor state entry point defined in kernel/kernel.c
extern void s_start();
// M-mode trap entry point, added @lab1_2
extern void mtrapvec();

// htif is defined in spike_interface/spike_htif.c, marks the availability of HTIF
extern uint64 htif;
// g_mem_size is defined in spike_interface/spike_memory.c, size of the emulated memory
extern uint64 g_mem_size;
// struct riscv_regs is define in kernel/riscv.h, and g_itrframe is used to save
// registers when interrupt hapens in M mode. added @lab1_2
riscv_regs g_itrframe;

//
// get the information of HTIF (calling interface) and the emulated memory by
// parsing the Device Tree Blog (DTB, actually DTS) stored in memory.
//
// the role of DTB is similar to that of Device Address Resolution Table (DART)
// in Intel series CPUs. it records the details of devices and memory of the
// platform simulated using Spike.
//
void init_dtb(uint64 dtb) {
  // defined in spike_interface/spike_htif.c, enabling Host-Target InterFace (HTIF)
  query_htif(dtb);
  if (htif) sprint("HTIF is available!\r\n");

  // defined in spike_interface/spike_memory.c, obtain information about emulated memory
  query_mem(dtb);
  sprint("(Emulated) memory size: %ld MB\n", g_mem_size >> 20);
}

//
// delegate (almost all) interrupts and most exceptions to S-mode.
// after delegation, syscalls will handled by the PKE OS kernel running in S-mode.
//
static void delegate_traps() {
  // supports_extension macro is defined in kernel/riscv.h
  if (!supports_extension('S')) {
    // confirm that our processor supports supervisor mode. abort if it does not.
    sprint("S mode is not supported.\n");
    return;
  }

  // macros used in following two statements are defined in kernel/riscv.h
  uintptr_t interrupts = MIP_SSIP | MIP_STIP | MIP_SEIP;
  uintptr_t exceptions = (1U << CAUSE_MISALIGNED_FETCH) | (1U << CAUSE_FETCH_PAGE_FAULT) |
                         (1U << CAUSE_BREAKPOINT) | (1U << CAUSE_LOAD_PAGE_FAULT) |
                         (1U << CAUSE_STORE_PAGE_FAULT) | (1U << CAUSE_USER_ECALL);

  // writes 64-bit values (interrupts and exceptions) to 'mideleg' and 'medeleg' (two
  // priviledged registers of RV64G machine) respectively.
  //
  // write_csr and read_csr are macros defined in kernel/riscv.h
  write_csr(mideleg, interrupts);
  write_csr(medeleg, exceptions);
  assert(read_csr(mideleg) == interrupts);
  assert(read_csr(medeleg) == exceptions);
}

//
// enabling timer interrupt (irq) in Machine mode. added @lab1_3
//
void timerinit(uintptr_t hartid) {
  // fire timer irq after TIMER_INTERVAL from now.
  *(uint64*)CLINT_MTIMECMP(hartid) = *(uint64*)CLINT_MTIME + TIMER_INTERVAL;

  // enable machine-mode timer irq in MIE (Machine Interrupt Enable) csr.
  write_csr(mie, read_csr(mie) | MIE_MTIE);
}

//
// m_start: machine mode C entry point.
//
void m_start(uintptr_t hartid, uintptr_t dtb) {
  // init the spike file interface (stdin,stdout,stderr)
  // functions with "spike_" prefix are all defined in codes under spike_interface/,
  // sprint is also defined in spike_interface/spike_utils.c
  spike_file_init();
  sprint("In m_start, hartid:%d\n", hartid);

  // init HTIF (Host-Target InterFace) and memory by using the Device Table Blob (DTB)
  // init_dtb() is defined above.
  init_dtb(dtb);

  // save the address of trap frame for interrupt in M mode to "mscratch". added @lab1_2
  write_csr(mscratch, &g_itrframe);

  // set previous privilege mode to S (Supervisor), and will enter S mode after 'mret'
  // write_csr is a macro defined in kernel/riscv.h
  write_csr(mstatus, ((read_csr(mstatus) & ~MSTATUS_MPP_MASK) | MSTATUS_MPP_S));

  // set M Exception Program Counter to sstart, for mret (requires gcc -mcmodel=medany)
  write_csr(mepc, (uint64)s_start);

  // setup trap handling vector for machine mode. added @lab1_2
  write_csr(mtvec, (uint64)mtrapvec);

  // enable machine-mode interrupts. added @lab1_3
  write_csr(mstatus, read_csr(mstatus) | MSTATUS_MIE);

  // delegate all interrupts and exceptions to supervisor mode.
  // delegate_traps() is defined above.
  delegate_traps();

  // also enables interrupt handling in supervisor mode. added @lab1_3
  write_csr(sie, read_csr(sie) | SIE_SEIE | SIE_STIE | SIE_SSIE);

  // init timing. added @lab1_3
  timerinit(hartid);

  // switch to supervisor mode (S mode) and jump to s_start(), i.e., set pc to mepc
  asm volatile("mret");
}
```

- mtrap_vector.S
```S
#include "util/load_store.S"

#
# M-mode trap entry point
#
.globl mtrapvec
.align 4
mtrapvec:
    # mscratch -> g_itrframe (cf. kernel/machine/minit.c line 94)
    # swap a0 and mscratch, so that a0 points to interrupt frame,
    # i.e., [a0] = &g_itrframe
    csrrw a0, mscratch, a0

    # save the registers in g_itrframe
    addi t6, a0, 0
    store_all_registers
    # save the original content of a0 in g_itrframe
    csrr t0, mscratch
    sd t0, 72(a0)

    # switch stack (to use stack0) for the rest of machine mode
    # trap handling.
    la sp, stack0
    li a3, 4096
    csrr a4, mhartid
    addi a4, a4, 1
    mul a3, a3, a4
    add sp, sp, a3

    # pointing mscratch back to g_itrframe
    csrw mscratch, a0

    # call machine mode trap handling function
    call handle_mtrap

    # restore all registers, come back to the status before entering
    # machine mode handling.
    csrr t6, mscratch
    restore_all_registers

    mret
```

- mentry.S
```S
#
# _mentry is the entry point of riscv-pke OS kernel.
#
# !Important (for your understanding)
# Before entering _mentry, two argument registers, i.e., a0(x10) and a1(x11), are set by
# our emulator (i.e., spike).
# [a0] = processor ID  (in the context of RISC-V, a processor is called as a HART, i.e.,
# Hardware Thread).
# [a1] = pointer to the DTS (i.e., Device Tree String), which is stored in the memory of
# RISC-V guest computer emulated by spike.
#

.globl _mentry
_mentry:
    # [mscratch] = 0; mscratch points the stack bottom of machine mode computer
    csrw mscratch, x0

    # following codes allocate a 4096-byte stack for each HART, although we use only
    # ONE HART in this lab.
    la sp, stack0		# stack0 is statically defined in kernel/machine/minit.c 
    li a3, 4096			# 4096-byte stack
    csrr a4, mhartid	# [mhartid] = core ID
    addi a4, a4, 1
    mul a3, a3, a4
    add sp, sp, a3		# re-arrange the stack points so that they don't overlap

    # jump to mstart(), i.e., machine state start function in kernel/machine/minit.c
    call m_start
```

- hostfc.c
```c
/*
 * Interface functions between VFS and host-fs. added @lab4_1.
 */
#include "hostfs.h"

#include "pmm.h"
#include "spike_interface/spike_file.h"
#include "spike_interface/spike_utils.h"
#include "util/string.h"
#include "util/types.h"
#include "vfs.h"

/**** host-fs vinode interface ****/
const struct vinode_ops hostfs_i_ops = {
    .viop_read = hostfs_read,
    .viop_write = hostfs_write,
    .viop_create = hostfs_create,
    .viop_lseek = hostfs_lseek,
    .viop_lookup = hostfs_lookup,

    .viop_hook_open = hostfs_hook_open,
    .viop_hook_close = hostfs_hook_close,

    .viop_write_back_vinode = hostfs_write_back_vinode,

    // not implemented
    .viop_link = hostfs_link,
    .viop_unlink = hostfs_unlink,
    .viop_readdir = hostfs_readdir,
    .viop_mkdir = hostfs_mkdir,
};

/**** hostfs utility functions ****/
//
// append hostfs to the fs list.
//
int register_hostfs() {
  struct file_system_type *fs_type = (struct file_system_type *)alloc_page();
  fs_type->type_num = HOSTFS_TYPE;
  fs_type->get_superblock = hostfs_get_superblock;

  for (int i = 0; i < MAX_SUPPORTED_FS; i++) {
    if (fs_list[i] == NULL) {
      fs_list[i] = fs_type;
      return 0;
    }
  }
  return -1;
}

//
// append new device under "name" to vfs_dev_list.
//
struct device *init_host_device(char *name) {
  // find rfs in registered fs list
  struct file_system_type *fs_type = NULL;
  for (int i = 0; i < MAX_SUPPORTED_FS; i++) {
    if (fs_list[i] != NULL && fs_list[i]->type_num == HOSTFS_TYPE) {
      fs_type = fs_list[i];
      break;
    }
  }
  if (!fs_type)
    panic("init_host_device: No HOSTFS file system found!\n");

  // allocate a vfs device
  struct device *device = (struct device *)alloc_page();
  // set the device name and index
  strcpy(device->dev_name, name);
  // we only support one host-fs device
  device->dev_id = 0;
  device->fs_type = fs_type;

  // add the device to the vfs device list
  for (int i = 0; i < MAX_VFS_DEV; i++) {
    if (vfs_dev_list[i] == NULL) {
      vfs_dev_list[i] = device;
      break;
    }
  }

  return device;
}

//
// recursive call to assemble a path.
//
void path_backtrack(char *path, struct dentry *dentry) {
  if (dentry->parent == NULL) {
    return;
  }
  path_backtrack(path, dentry->parent);
  strcat(path, "/");
  strcat(path, dentry->name);
}

//
// obtain the absolute path for "dentry", from root to file.
//
void get_path_string(char *path, struct dentry *dentry) {
  strcpy(path, H_ROOT_DIR);
  path_backtrack(path, dentry);
}

//
// allocate a vfs inode for an host fs file.
//
struct vinode *hostfs_alloc_vinode(struct super_block *sb) {
  struct vinode *vinode = default_alloc_vinode(sb);
  vinode->inum = -1; 
  vinode->i_fs_info = NULL;
  vinode->i_ops = &hostfs_i_ops;
  return vinode;
}

int hostfs_write_back_vinode(struct vinode *vinode) { return 0; }

//
// populate the vfs inode of an hostfs file, according to its stats.
//
int hostfs_update_vinode(struct vinode *vinode) {
  spike_file_t *f = vinode->i_fs_info;
  if ((int64)f < 0) {  // is a direntry
    vinode->type = H_DIR;
    return -1;
  }

  struct stat stat;
  spike_file_stat(f, &stat);

  vinode->inum = stat.st_ino;
  vinode->size = stat.st_size;
  vinode->nlinks = stat.st_nlink;
  vinode->blocks = stat.st_blocks;

  if (S_ISDIR(stat.st_mode)) {
    vinode->type = H_DIR;
  } else if (S_ISREG(stat.st_mode)) {
    vinode->type = H_FILE;
  } else {
    sprint("hostfs_lookup:unknown file type!");
    return -1;
  }

  return 0;
}

/**** vfs-host-fs interface functions ****/
//
// read a hostfs file.
//
ssize_t hostfs_read(struct vinode *f_inode, char *r_buf, ssize_t len,
                    int *offset) {
  spike_file_t *pf = (spike_file_t *)f_inode->i_fs_info;
  if (pf < 0) {
    sprint("hostfs_read: invalid file handle!\n");
    return -1;
  }
  int read_len = spike_file_read(pf, r_buf, len);
  // obtain current offset
  *offset = spike_file_lseek(pf, 0, 1);
  return read_len;
}

//
// write a hostfs file.
//
ssize_t hostfs_write(struct vinode *f_inode, const char *w_buf, ssize_t len,
                     int *offset) {
  spike_file_t *pf = (spike_file_t *)f_inode->i_fs_info;
  if (pf < 0) {
    sprint("hostfs_write: invalid file handle!\n");
    return -1;
  }
  int write_len = spike_file_write(pf, w_buf, len);
  // obtain current offset
  *offset = spike_file_lseek(pf, 0, 1);
  return write_len;
}

//
// lookup a hostfs file, and establish its vfs inode in PKE vfs.
//
struct vinode *hostfs_lookup(struct vinode *parent, struct dentry *sub_dentry) {
  // get complete path string
  char path[MAX_PATH_LEN];
  get_path_string(path, sub_dentry);

  spike_file_t *f = spike_file_open(path, O_RDWR, 0);

  struct vinode *child_inode = hostfs_alloc_vinode(parent->sb);
  child_inode->i_fs_info = f;
  hostfs_update_vinode(child_inode);

  child_inode->ref = 0;
  return child_inode;
}

//
// creates a hostfs file, and establish its vfs inode. 
//
struct vinode *hostfs_create(struct vinode *parent, struct dentry *sub_dentry) {
  char path[MAX_PATH_LEN];
  get_path_string(path, sub_dentry);

  spike_file_t *f = spike_file_open(path, O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
  if ((int64)f < 0) {
    sprint("hostfs_create cannot create the given file.\n");
    return NULL;
  }

  struct vinode *new_inode = hostfs_alloc_vinode(parent->sb);
  new_inode->i_fs_info = f;

  if (hostfs_update_vinode(new_inode) != 0) return NULL;

  new_inode->ref = 0;
  return new_inode;
}

//
// reposition read/write file offset
//
int hostfs_lseek(struct vinode *f_inode, ssize_t new_offset, int whence,
                  int *offset) {
  spike_file_t *f = (spike_file_t *)f_inode->i_fs_info;
  if (f < 0) {
    sprint("hostfs_lseek: invalid file handle!\n");
    return -1;
  }

  *offset = spike_file_lseek(f, new_offset, whence);
  if (*offset >= 0)
    return 0;
  return -1;
}

int hostfs_link(struct vinode *parent, struct dentry *sub_dentry,
                struct vinode *link_node) {
  panic("hostfs_link not implemented!\n");
  return -1;
}

int hostfs_unlink(struct vinode *parent, struct dentry *sub_dentry, struct vinode *unlink_node) {
  panic("hostfs_unlink not implemented!\n");
  return -1;
}

int hostfs_readdir(struct vinode *dir_vinode, struct dir *dir, int *offset) {
  panic("hostfs_readdir not implemented!\n");
  return -1;
}

struct vinode *hostfs_mkdir(struct vinode *parent, struct dentry *sub_dentry) {
  panic("hostfs_mkdir not implemented!\n");
  return NULL;
}

/**** vfs-hostfs hook interface functions ****/
//
// open a hostfs file (after having its vfs inode).
//
int hostfs_hook_open(struct vinode *f_inode, struct dentry *f_dentry) {
  if (f_inode->i_fs_info != NULL) return 0;

  char path[MAX_PATH_LEN];
  get_path_string(path, f_dentry);
  spike_file_t *f = spike_file_open(path, O_RDWR, 0);
  if ((int64)f < 0) {
    sprint("hostfs_hook_open cannot open the given file.\n");
    return -1;
  }

  f_inode->i_fs_info = f;
  return 0;
}

//
// close a hostfs file.
//
int hostfs_hook_close(struct vinode *f_inode, struct dentry *dentry) {
  spike_file_t *f = (spike_file_t *)f_inode->i_fs_info;
  spike_file_close(f);
  return 0;
}

/**** vfs-hostfs file system type interface functions ****/
struct super_block *hostfs_get_superblock(struct device *dev) {
  // set the data for the vfs super block
  struct super_block *sb = alloc_page();
  sb->s_dev = dev;

  struct vinode *root_inode = hostfs_alloc_vinode(sb);
  root_inode->type = H_DIR;

  struct dentry *root_dentry = alloc_vfs_dentry("/", root_inode, NULL);
  sb->s_root = root_dentry;

  return sb;
}
```

- hostfc.h
```c
#ifndef _HOSTFS_H_
#define _HOSTFS_H_
#include "vfs.h"

#define HOSTFS_TYPE 1

// dinode type
#define H_FILE FILE_I
#define H_DIR DIR_I

// root directory
#define H_ROOT_DIR "./hostfs_root"

// hostfs utility functin declarations
int register_hostfs();
struct device *init_host_device(char *name);
void get_path_string(char *path, struct dentry *dentry);
struct vinode *hostfs_alloc_vinode(struct super_block *sb);
int hostfs_write_back_vinode(struct vinode *vinode);
int hostfs_update_vinode(struct vinode *vinode);

// hostfs interface function declarations
ssize_t hostfs_read(struct vinode *f_inode, char *r_buf, ssize_t len,
                    int *offset);
ssize_t hostfs_write(struct vinode *f_inode, const char *w_buf, ssize_t len,
                     int *offset);
struct vinode *hostfs_lookup(struct vinode *parent, struct dentry *sub_dentry);
struct vinode *hostfs_create(struct vinode *parent, struct dentry *sub_dentry);
int hostfs_lseek(struct vinode *f_inode, ssize_t new_offset, int whence,
                  int *offset);
int hostfs_link(struct vinode *parent, struct dentry *sub_dentry, struct vinode *link_node);
int hostfs_unlink(struct vinode *parent, struct dentry *sub_dentry, struct vinode *unlink_node);
int hostfs_hook_open(struct vinode *f_inode, struct dentry *f_dentry);
int hostfs_hook_close(struct vinode *f_inode, struct dentry *dentry);
int hostfs_readdir(struct vinode *dir_vinode, struct dir *dir, int *offset);
struct vinode *hostfs_mkdir(struct vinode *parent, struct dentry *sub_dentry);
struct super_block *hostfs_get_superblock(struct device *dev);

extern const struct vinode_ops hostfs_node_ops;

#endif
```

- memlayout.h
```c
#ifndef _MEMLAYOUT_H
#define _MEMLAYOUT_H
#include "riscv.h"

// RISC-V machine places its physical memory above DRAM_BASE
#define DRAM_BASE 0x80000000

// the beginning virtual address of PKE kernel
#define KERN_BASE 0x80000000

// default stack size
#define STACK_SIZE 4096

// virtual address of stack top of user process
#define USER_STACK_TOP 0x7ffff000

// start virtual address (4MB) of our simple heap. added @lab2_2
#define USER_FREE_ADDRESS_START 0x00000000 + PGSIZE * 1024

#endif
```

- pmm.c
```c
#include "pmm.h"
#include "util/functions.h"
#include "riscv.h"
#include "config.h"
#include "util/string.h"
#include "memlayout.h"
#include "spike_interface/spike_utils.h"

// _end is defined in kernel/kernel.lds, it marks the ending (virtual) address of PKE kernel
extern char _end[];
// g_mem_size is defined in spike_interface/spike_memory.c, it indicates the size of our
// (emulated) spike machine. g_mem_size's value is obtained when initializing HTIF. 
extern uint64 g_mem_size;

static uint64 free_mem_start_addr;  //beginning address of free memory
static uint64 free_mem_end_addr;    //end address of free memory (not included)

typedef struct node {
  struct node *next;
} list_node;

// g_free_mem_list is the head of the list of free physical memory pages
static list_node g_free_mem_list;

//
// actually creates the freepage list. each page occupies 4KB (PGSIZE), i.e., small page.
// PGSIZE is defined in kernel/riscv.h, ROUNDUP is defined in util/functions.h.
//
static void create_freepage_list(uint64 start, uint64 end) {
  g_free_mem_list.next = 0;
  for (uint64 p = ROUNDUP(start, PGSIZE); p + PGSIZE < end; p += PGSIZE)
    free_page( (void *)p );
}

//
// place a physical page at *pa to the free list of g_free_mem_list (to reclaim the page)
//
void free_page(void *pa) {
  if (((uint64)pa % PGSIZE) != 0 || (uint64)pa < free_mem_start_addr || (uint64)pa >= free_mem_end_addr)
    panic("free_page 0x%lx \n", pa);

  // insert a physical page to g_free_mem_list
  list_node *n = (list_node *)pa;
  n->next = g_free_mem_list.next;
  g_free_mem_list.next = n;
}

//
// takes the first free page from g_free_mem_list, and returns (allocates) it.
// Allocates only ONE page!
//
void *alloc_page(void) {
  list_node *n = g_free_mem_list.next;
  if (n) g_free_mem_list.next = n->next;

  return (void *)n;
}

//
// pmm_init() establishes the list of free physical pages according to available
// physical memory space.
//
void pmm_init() {
  // start of kernel program segment
  uint64 g_kernel_start = KERN_BASE;
  uint64 g_kernel_end = (uint64)&_end;

  uint64 pke_kernel_size = g_kernel_end - g_kernel_start;
  sprint("PKE kernel start 0x%lx, PKE kernel end: 0x%lx, PKE kernel size: 0x%lx .\n",
    g_kernel_start, g_kernel_end, pke_kernel_size);

  // free memory starts from the end of PKE kernel and must be page-aligined
  free_mem_start_addr = ROUNDUP(g_kernel_end , PGSIZE);

  // recompute g_mem_size to limit the physical memory space that our riscv-pke kernel
  // needs to manage
  g_mem_size = MIN(PKE_MAX_ALLOWABLE_RAM, g_mem_size);
  if( g_mem_size < pke_kernel_size )
    panic( "Error when recomputing physical memory size (g_mem_size).\n" );

  free_mem_end_addr = g_mem_size + DRAM_BASE;
  sprint("free physical memory address: [0x%lx, 0x%lx] \n", free_mem_start_addr,
    free_mem_end_addr - 1);

  sprint("kernel memory manager is initializing ...\n");
  // create the list of free pages
  create_freepage_list(free_mem_start_addr, free_mem_end_addr);
}
```

- pmm.h
```c
#ifndef _PMM_H_
#ifndef _PMM_H_
#define _PMM_H_

// Initialize phisical memeory manager
void pmm_init();
// Allocate a free phisical page
void* alloc_page();
// Free an allocated page
void free_page(void* pa);

#endif
```

- proc_file.c
```c
/*
 * Interface functions between file system and kernel/processes. added @lab4_1
 */

#include "proc_file.h"

#include "hostfs.h"
#include "pmm.h"
#include "process.h"
#include "ramdev.h"
#include "rfs.h"
#include "riscv.h"
#include "spike_interface/spike_file.h"
#include "spike_interface/spike_utils.h"
#include "util/functions.h"
#include "util/string.h"

//
// initialize file system
//
void fs_init(void) {
  // initialize the vfs
  vfs_init();

  // register hostfs and mount it as the root
  if( register_hostfs() < 0 ) panic( "fs_init: cannot register hostfs.\n" );
  struct device *hostdev = init_host_device("HOSTDEV");
  vfs_mount("HOSTDEV", MOUNT_AS_ROOT);

  // register and mount rfs
  if( register_rfs() < 0 ) panic( "fs_init: cannot register rfs.\n" );
  struct device *ramdisk0 = init_rfs_device("RAMDISK0");
  rfs_format_dev(ramdisk0);
  vfs_mount("RAMDISK0", MOUNT_DEFAULT);
}

//
// initialize a proc_file_management data structure for a process.
// return the pointer to the page containing the data structure.
//
proc_file_management *init_proc_file_management(void) {
  proc_file_management *pfiles = (proc_file_management *)alloc_page();
  pfiles->cwd = vfs_root_dentry; // by default, cwd is the root
  pfiles->nfiles = 0;

  for (int fd = 0; fd < MAX_FILES; ++fd)
    pfiles->opened_files[fd].status = FD_NONE;

  sprint("FS: created a file management struct for a process.\n");
  return pfiles;
}

//
// reclaim the open-file management data structure of a process.
// note: this function is not used as PKE does not actually reclaim a process.
//
void reclaim_proc_file_management(proc_file_management *pfiles) {
  free_page(pfiles);
  return;
}

//
// get an opened file from proc->opened_file array.
// return: the pointer to the opened file structure.
//
struct file *get_opened_file(int fd) {
  struct file *pfile = NULL;

  // browse opened file list to locate the fd
  for (int i = 0; i < MAX_FILES; ++i) {
    pfile = &(current->pfiles->opened_files[i]);  // file entry
    if (i == fd) break;
  }
  if (pfile == NULL) panic("do_read: invalid fd!\n");
  return pfile;
}

//
// open a file named as "pathname" with the permission of "flags".
// return: -1 on failure; non-zero file-descriptor on success.
//
int do_open(char *pathname, int flags) {
  struct file *opened_file = NULL;
  if ((opened_file = vfs_open(pathname, flags)) == NULL) return -1;

  int fd = 0;
  if (current->pfiles->nfiles >= MAX_FILES) {
    panic("do_open: no file entry for current process!\n");
  }
  struct file *pfile;
  for (fd = 0; fd < MAX_FILES; ++fd) {
    pfile = &(current->pfiles->opened_files[fd]);
    if (pfile->status == FD_NONE) break;
  }

  // initialize this file structure
  memcpy(pfile, opened_file, sizeof(struct file));

  ++current->pfiles->nfiles;
  return fd;
}

//
// read content of a file ("fd") into "buf" for "count".
// return: actual length of data read from the file.
//
int do_read(int fd, char *buf, uint64 count) {
  struct file *pfile = get_opened_file(fd);

  if (pfile->readable == 0) panic("do_read: no readable file!\n");

  char buffer[count + 1];
  int len = vfs_read(pfile, buffer, count);
  buffer[count] = '\0';
  strcpy(buf, buffer);
  return len;
}

//
// write content ("buf") whose length is "count" to a file "fd".
// return: actual length of data written to the file.
//
int do_write(int fd, char *buf, uint64 count) {
  struct file *pfile = get_opened_file(fd);

  if (pfile->writable == 0) panic("do_write: cannot write file!\n");

  int len = vfs_write(pfile, buf, count);
  return len;
}

//
// reposition the file offset
//
int do_lseek(int fd, int offset, int whence) {
  struct file *pfile = get_opened_file(fd);
  return vfs_lseek(pfile, offset, whence);
}

//
// read the vinode information
//
int do_stat(int fd, struct istat *istat) {
  struct file *pfile = get_opened_file(fd);
  return vfs_stat(pfile, istat);
}

//
// read the inode information on the disk
//
int do_disk_stat(int fd, struct istat *istat) {
  struct file *pfile = get_opened_file(fd);
  return vfs_disk_stat(pfile, istat);
}

//
// close a file
//
int do_close(int fd) {
  struct file *pfile = get_opened_file(fd);
  return vfs_close(pfile);
}

//
// open a directory
// return: the fd of the directory file
//
int do_opendir(char *pathname) {
  struct file *opened_file = NULL;
  if ((opened_file = vfs_opendir(pathname)) == NULL) return -1;

  int fd = 0;
  struct file *pfile;
  for (fd = 0; fd < MAX_FILES; ++fd) {
    pfile = &(current->pfiles->opened_files[fd]);
    if (pfile->status == FD_NONE) break;
  }
  if (pfile->status != FD_NONE)  // no free entry
    panic("do_opendir: no file entry for current process!\n");

  // initialize this file structure
  memcpy(pfile, opened_file, sizeof(struct file));

  ++current->pfiles->nfiles;
  return fd;
}

//
// read a directory entry
//
int do_readdir(int fd, struct dir *dir) {
  struct file *pfile = get_opened_file(fd);
  return vfs_readdir(pfile, dir);
}

//
// make a new directory
//
int do_mkdir(char *pathname) {
  return vfs_mkdir(pathname);
}

//
// close a directory
//
int do_closedir(int fd) {
  struct file *pfile = get_opened_file(fd);
  return vfs_closedir(pfile);
}

//
// create hard link to a file
//
int do_link(char *oldpath, char *newpath) {
  return vfs_link(oldpath, newpath);
}

//
// remove a hard link to a file
//
int do_unlink(char *path) {
  return vfs_unlink(path);
}
```

- proc_file.h
```c
#ifndef _PROC_FILE_H_
#define _PROC_FILE_H_

#include "spike_interface/spike_file.h"
#include "util/types.h"
#include "vfs.h"

//
// file operations
//
int do_open(char *pathname, int flags);
int do_read(int fd, char *buf, uint64 count);
int do_write(int fd, char *buf, uint64 count);
int do_lseek(int fd, int offset, int whence);
int do_stat(int fd, struct istat *istat);
int do_disk_stat(int fd, struct istat *istat);
int do_close(int fd);

int do_opendir(char *pathname);
int do_readdir(int fd, struct dir *dir);
int do_mkdir(char *pathname);
int do_closedir(int fd);

int do_link(char *oldpath, char *newpath);
int do_unlink(char *path);

void fs_init(void);

// data structure that manages all openned files in a PCB
typedef struct proc_file_management_t {
  struct dentry *cwd;  // vfs dentry of current working directory
  struct file opened_files[MAX_FILES];  // opened files array
  int nfiles;  // the number of files opened by a process
} proc_file_management;

proc_file_management *init_proc_file_management(void);

void reclaim_proc_file_management(proc_file_management *pfiles);

#endif
```

- ramdev.c
```c
/*
 * Utility functions operating the devices. support only RAM disk device. added @lab4_1.
 */

#include "ramdev.h"
#include "vfs.h"
#include "pmm.h"
#include "riscv.h"
#include "util/types.h"
#include "util/string.h"
#include "spike_interface/spike_utils.h"
#include "rfs.h"

struct rfs_device *rfs_device_list[MAX_RAMDISK_COUNT];

//
// write the content stored in "buff" to the "blkno"^th block of disk.
//
int ramdisk_write(struct rfs_device *rfs_device, int blkno){
  if ( blkno < 0 || blkno >= RAMDISK_BLOCK_COUNT )
    panic("ramdisk_write: write block No %d out of range!\n", blkno);
  void * dst = (void *)((uint64)rfs_device->d_address + blkno * RAMDISK_BLOCK_SIZE);
  memcpy(dst, rfs_device->iobuffer, RAMDISK_BLOCK_SIZE);
  return 0;
}

//
// read the "blkno"^th block from the RAM disk and store its content into buffer.
//
int ramdisk_read(struct rfs_device *rfs_device, int blkno){
  if ( blkno < 0 || blkno >= RAMDISK_BLOCK_COUNT )
    panic("ramdisk_read: read block No out of range!\n");
  void * src = (void *)((uint64)rfs_device->d_address + blkno * RAMDISK_BLOCK_SIZE);
  memcpy(rfs_device->iobuffer, src, RAMDISK_BLOCK_SIZE);
  return 0;
}

//
// alloc RAMDISK_BLOCK_COUNT continuous pages (blocks) for the RAM Disk
// setup an vfs node, initialize RAM disk device, and attach the device with the vfs node.
//
struct device *init_rfs_device(const char *dev_name) {
  // find rfs in registered fs list
  struct file_system_type *fs_type = NULL;
  for (int i = 0; i < MAX_SUPPORTED_FS; i++) {
    if (fs_list[i] != NULL && fs_list[i]->type_num == RFS_TYPE) {
      fs_type = fs_list[i];
      break; 
    }
  }
  if (!fs_type) {
    panic("No RFS file system found!\n");
  }

  // alloc blocks for the RAM Disk
  void *curr_addr = NULL;
  void *last_addr = NULL;
  void *ramdisk_addr = NULL;
  for ( int i = 0; i < RAMDISK_BLOCK_COUNT; ++ i ){
    last_addr = curr_addr;
    curr_addr = alloc_page();
    if ( last_addr != NULL && last_addr - curr_addr != PGSIZE ){
      panic("RAM Disk0: address is discontinuous!\n");
    }
  }
  ramdisk_addr = curr_addr;

  // find a free rfs device
  struct rfs_device **rfs_device = NULL;
  int device_id = 0;
  for (int i = 0; i < MAX_RAMDISK_COUNT; i++) {
    if (rfs_device_list[i] == NULL) {
      rfs_device = &rfs_device_list[i];
      device_id = i;
      break;
    }
  }
  if (!rfs_device) {
    panic("RAM Disk0: no free device!\n");
  }
  
  *rfs_device = (struct rfs_device *)alloc_page();
  (*rfs_device)->d_blocks = RAMDISK_BLOCK_COUNT;
  (*rfs_device)->d_blocksize = RAMDISK_BLOCK_SIZE;
  (*rfs_device)->d_write = ramdisk_write;
  (*rfs_device)->d_read = ramdisk_read;
  (*rfs_device)->d_address = ramdisk_addr;
  (*rfs_device)->iobuffer = alloc_page();

  // allocate a vfs device
  struct device * device = (struct device *)alloc_page();
  // set the device name and index
  strcpy(device->dev_name, dev_name);
  device->dev_id = device_id;
  device->fs_type = fs_type;

  // add the device to the vfs device list
  for(int i = 0; i < MAX_VFS_DEV; i++) {
    if (vfs_dev_list[i] == NULL) {
      vfs_dev_list[i] = device;
      break;
    }
  }

  sprint("%s: base address of %s is: %p\n",dev_name, dev_name, ramdisk_addr);
  return device;
}
```

- ramdev.h
```c
#ifndef _RAMDEV_H_
#define _RAMDEV_H_

#include "riscv.h"
#include "util/types.h"

#define RAMDISK_BLOCK_COUNT 128
#define RAMDISK_BLOCK_SIZE  PGSIZE

#define MAX_RAMDISK_COUNT 10

#define RAMDISK_FREE 0
#define RAMDISK_USED 1

struct rfs_device {
  void *d_address;  // the ramdisk base address
  int d_blocks;     // the number of blocks of the device
  int d_blocksize;  // the blocksize (bytes) per block
  void *iobuffer;   // iobuffer for write/read
  int (*d_write)(struct rfs_device *rdev, int blkno);  // device write funtion
  int (*d_read)(struct rfs_device *rdev, int blkno); // device read funtion
};

#define dop_write(rdev, blkno) ((rdev)->d_write(rdev, blkno))
#define dop_read(rdev, blkno)  ((rdev)->d_read(rdev, blkno))

struct device *init_rfs_device(const char *dev_name);
struct rfs_device *alloc_rfs_device(void);

extern struct rfs_device *rfs_device_list[MAX_RAMDISK_COUNT];

#endif
```

- rfs.c
```c
/*
 * RFS (Ramdisk File System) is a customized simple file system installed in the
 * RAM disk. added @lab4_1.
 * Layout of the file system:
 *
 * ******** RFS MEM LAYOUT (112 BLOCKS) ****************
 *   superblock  |  disk inodes  |  bitmap  |  free blocks  *
 *     1 block   |   10 blocks   |     1    |     100       *
 * *****************************************************
 *
 * The disk layout of rfs is similar to the fs in xv6.
 */
#include "rfs.h"

#include "pmm.h"
#include "ramdev.h"
#include "spike_interface/spike_utils.h"
#include "util/string.h"
#include "vfs.h"

/**** vinode inteface ****/
const struct vinode_ops rfs_i_ops = {
    .viop_read = rfs_read,
    .viop_write = rfs_write,
    .viop_create = rfs_create,
    .viop_lseek = rfs_lseek,
    .viop_disk_stat = rfs_disk_stat,
    .viop_link = rfs_link,
    .viop_unlink = rfs_unlink,
    .viop_lookup = rfs_lookup,

    .viop_readdir = rfs_readdir,
    .viop_mkdir = rfs_mkdir,

    .viop_write_back_vinode = rfs_write_back_vinode,

    .viop_hook_opendir = rfs_hook_opendir,
    .viop_hook_closedir = rfs_hook_closedir,
};

/**** rfs utility functions ****/
//
// register rfs to the fs list supported by PKE.
//
int register_rfs() {
  struct file_system_type *fs_type = (struct file_system_type *)alloc_page();
  fs_type->type_num = RFS_TYPE;
  fs_type->get_superblock = rfs_get_superblock;

  for (int i = 0; i < MAX_SUPPORTED_FS; i++) {
    if (fs_list[i] == NULL) {
      fs_list[i] = fs_type;
      return 0;
    }
  }
  return -1;
}

//
// format "dev" with rfs. note the "dev" should be a ram disk device.
//
int rfs_format_dev(struct device *dev) {
  struct rfs_device *rdev = rfs_device_list[dev->dev_id];

  // ** first, format the superblock
  // build a new superblock
  struct super_block *super = (struct super_block *)rdev->iobuffer;
  super->magic = RFS_MAGIC;
  super->size =
      1 + RFS_MAX_INODE_BLKNUM + 1 + RFS_MAX_INODE_BLKNUM * RFS_DIRECT_BLKNUM;
  // only direct index blocks
  super->nblocks = RFS_MAX_INODE_BLKNUM * RFS_DIRECT_BLKNUM;
  super->ninodes = RFS_BLKSIZE / RFS_INODESIZE * RFS_MAX_INODE_BLKNUM;

  // write the superblock to RAM Disk0
  if (rfs_w1block(rdev, RFS_BLK_OFFSET_SUPER) != 0)  // write to device
    panic("RFS: failed to write superblock!\n");

  // ** second, set up the inodes and write them to RAM disk
  // build an empty inode disk block which has RFS_BLKSIZE/RFS_INODESIZE(=32)
  // disk inodes
  struct rfs_dinode *p_dinode = (struct rfs_dinode *)rdev->iobuffer;
  for (int i = 0; i < RFS_BLKSIZE / RFS_INODESIZE; ++i) {
    p_dinode->size = 0;
    p_dinode->type = R_FREE;
    p_dinode->nlinks = 0;
    p_dinode->blocks = 0;
    p_dinode = (struct rfs_dinode *)((char *)p_dinode + RFS_INODESIZE);
  }

  // write RFS_MAX_INODE_BLKNUM(=10) empty inode disk blocks to RAM Disk0
  for (int inode_block = 0; inode_block < RFS_MAX_INODE_BLKNUM; ++inode_block) {
    if (rfs_w1block(rdev, RFS_BLK_OFFSET_INODE + inode_block) != 0)
      panic("RFS: failed to initialize empty inodes!\n");
  }

  // build root directory inode (ino = 0)
  struct rfs_dinode root_dinode;
  root_dinode.size = 0;
  root_dinode.type = R_DIR;
  root_dinode.nlinks = 1;
  root_dinode.blocks = 1;
  root_dinode.addrs[0] = RFS_BLK_OFFSET_FREE;

  // write root directory inode to RAM Disk0 (ino = 0)
  if (rfs_write_dinode(rdev, &root_dinode, 0) != 0) {
    sprint("RFS: failed to write root inode!\n");
    return -1;
  }

  // ** third, write freemap to disk
  int *freemap = (int *)rdev->iobuffer;
  memset(freemap, 0, RFS_BLKSIZE);
  freemap[0] = 1;  // the first data block is used for root directory

  // write the bitmap to RAM Disk0
  if (rfs_w1block(rdev, RFS_BLK_OFFSET_BITMAP) != 0) {  // write to device
    sprint("RFS: failed to write bitmap!\n");
    return -1;
  }

  sprint("RFS: format %s done!\n", dev->dev_name);
  return 0;
}

// ** Note: If you use the following four functions interchangeably,
// ** be sure to watch out for IOBUFFER BEING OVERWRITTEN !!!

//
// call ramdisk_read via the device structure.
// read the "n_block"^th block from RAM disk to the iobuffer of rfs_dev.
//
int rfs_r1block(struct rfs_device *rfs_dev, int n_block) {
  return dop_read(rfs_dev, n_block);
}

//
// call ramdisk_write via the device structure.
// write iobuffer of rfs_dev to RAM disk at the "n_block"^th block.
//
int rfs_w1block(struct rfs_device *rfs_dev, int n_block) {
  return dop_write(rfs_dev, n_block);
}

//
// read disk inode from RAM disk
//
struct rfs_dinode *rfs_read_dinode(struct rfs_device *rdev, int n_inode) {
  int n_block = n_inode / (RFS_BLKSIZE / RFS_INODESIZE) + RFS_BLK_OFFSET_INODE;
  int offset = n_inode % (RFS_BLKSIZE / RFS_INODESIZE);

  // call ramdisk_read defined in dev.c
  if (dop_read(rdev, n_block) != 0) return NULL;
  struct rfs_dinode *dinode = (struct rfs_dinode *)alloc_page();
  memcpy(dinode, (char *)rdev->iobuffer + offset * RFS_INODESIZE,
         sizeof(struct rfs_dinode));
  return dinode;
}

//
// write disk inode to RAM disk.
// note: we need first read the "disk" block containing the "n_inode"^th inode,
// modify it, and write the block back to "disk" eventually.
//
int rfs_write_dinode(struct rfs_device *rdev, const struct rfs_dinode *dinode,
                     int n_inode) {
  int n_block = n_inode / (RFS_BLKSIZE / RFS_INODESIZE) + RFS_BLK_OFFSET_INODE;
  int offset = n_inode % (RFS_BLKSIZE / RFS_INODESIZE);

  // call ramdisk_read defined in dev.c
  dop_read(rdev, n_block);
  memcpy(rdev->iobuffer + offset * RFS_INODESIZE, dinode,
         sizeof(struct rfs_dinode));
  // call ramdisk_write defined in dev.c
  int ret = dop_write(rdev, n_block);

  return ret;
}

//
// allocate a block from RAM disk
//
int rfs_alloc_block(struct super_block *sb) {
  int free_block = -1;
  // think of s_fs_info as freemap information
  int *freemap = (int *)sb->s_fs_info;
  for (int block = 0; block < sb->nblocks; ++block) {
    if (freemap[block] == 0) {  // find a free block
      freemap[block] = 1;
      free_block = RFS_BLK_OFFSET_FREE + block;
      break;
    }
  }
  if (free_block == -1) panic("rfs_alloc_block: no more free block!\n");
  return free_block;
}

//
// free a block in RAM disk
//
int rfs_free_block(struct super_block *sb, int block_num) {
  int *freemap = (int *)sb->s_fs_info;
  freemap[block_num - RFS_BLK_OFFSET_FREE] = 0;
  return 0;
}

//
// add a new directory entry to a directory
//
int rfs_add_direntry(struct vinode *dir, const char *name, int inum) {
  if (dir->type != DIR_I) {
    sprint("rfs_add_direntry: not a directory!\n");
    return -1;
  }

  struct rfs_device *rdev = rfs_device_list[dir->sb->s_dev->dev_id];
  int n_block = dir->addrs[dir->size / RFS_BLKSIZE];
  if (rfs_r1block(rdev, n_block) != 0) {
    sprint("rfs_add_direntry: failed to read block %d!\n", n_block);
    return -1;
  }

  // prepare iobuffer
  char *addr = (char *)rdev->iobuffer + dir->size % RFS_BLKSIZE;
  struct rfs_direntry *p_direntry = (struct rfs_direntry *)addr;
  p_direntry->inum = inum;
  strcpy(p_direntry->name, name);

  // write the modified (parent) directory block back to disk
  if (rfs_w1block(rdev, n_block) != 0) {
    sprint("rfs_add_direntry: failed to write block %d!\n", n_block);
    return -1;
  }

  // update its parent dir state
  dir->size += sizeof(struct rfs_direntry);

  // write the parent dir inode back to disk
  if (rfs_write_back_vinode(dir) != 0) {
    sprint("rfs_add_direntry: failed to write back parent dir inode!\n");
    return -1;
  }

  return 0;
}

//
// alloc a new (and empty) vinode
//
struct vinode *rfs_alloc_vinode(struct super_block *sb) {
  struct vinode *vinode = default_alloc_vinode(sb);
  vinode->i_ops = &rfs_i_ops;
  return vinode;
}

//
// convert vfs inode to disk inode, and write it back to disk
//
int rfs_write_back_vinode(struct vinode *vinode) {
  // copy vinode info to disk inode
  struct rfs_dinode dinode;
  dinode.size = vinode->size;
  dinode.nlinks = vinode->nlinks;
  dinode.blocks = vinode->blocks;
  dinode.type = vinode->type;
  for (int i = 0; i < RFS_DIRECT_BLKNUM; ++i) {
    dinode.addrs[i] = vinode->addrs[i];
  }

  struct rfs_device *rdev = rfs_device_list[vinode->sb->s_dev->dev_id];
  if (rfs_write_dinode(rdev, &dinode, vinode->inum) != 0) {
    sprint("rfs_free_write_back_inode: failed to write back disk inode!\n");
    return -1;
  }

  return 0;
}

//
// update vinode info by reading disk inode
//
int rfs_update_vinode(struct vinode *vinode) {
  struct rfs_device *rdev = rfs_device_list[vinode->sb->s_dev->dev_id];
  struct rfs_dinode *dinode = rfs_read_dinode(rdev, vinode->inum);
  if (dinode == NULL) {
    sprint("rfs_update_vinode: failed to read disk inode!\n");
    return -1;
  }
  vinode->size = dinode->size;
  vinode->nlinks = dinode->nlinks;
  vinode->blocks = dinode->blocks;
  vinode->type = dinode->type;
  for (int i = 0; i < RFS_DIRECT_BLKNUM; ++i) {
    vinode->addrs[i] = dinode->addrs[i];
  }
  free_page(dinode);

  return 0;
}

/**** vfs-rfs file interface functions ****/
//
// read the content (for "len") of a file ("f_inode"), and copy the content
// to "r_buf".
//
ssize_t rfs_read(struct vinode *f_inode, char *r_buf, ssize_t len,
                 int *offset) {
  // obtain disk inode from vfs inode
  if (f_inode->size < *offset)
    panic("rfs_read:offset should less than file size!");

  if (f_inode->size < (*offset + len)) len = f_inode->size - *offset;

  char buffer[len + 1];

  // compute how many blocks we need to read
  int align = *offset % RFS_BLKSIZE;
  int block_offset = *offset / RFS_BLKSIZE;
  int buf_offset = 0;

  int readtimes = (align + len) / RFS_BLKSIZE;
  int remain = (align + len) % RFS_BLKSIZE;

  struct rfs_device *rdev = rfs_device_list[f_inode->sb->s_dev->dev_id];

  // read first block
  rfs_r1block(rdev, f_inode->addrs[block_offset]);
  int first_block_len = (readtimes == 0 ? len : RFS_BLKSIZE - align);
  memcpy(buffer + buf_offset, rdev->iobuffer + align, first_block_len);
  buf_offset += first_block_len;
  block_offset++;
  readtimes--;

  // readtimes < 0 means that the file has only one block (and not full),
  // so our work is done
  // otherwise...
  if (readtimes >= 0) {
    // read in complete blocks
    while (readtimes != 0) {
      rfs_r1block(rdev, f_inode->addrs[block_offset]);
      memcpy(buffer + buf_offset, rdev->iobuffer, RFS_BLKSIZE);
      buf_offset += RFS_BLKSIZE;
      block_offset++;
      readtimes--;
    }

    // read in the remaining data
    if (remain > 0) {
      rfs_r1block(rdev, f_inode->addrs[block_offset]);
      memcpy(buffer + buf_offset, rdev->iobuffer, remain);
    }
  }

  buffer[len] = '\0';
  strcpy(r_buf, buffer);

  *offset += len;
  return len;
}

//
// write the content of "w_buf" (lengthed "len") to a file ("f_inode").
//
ssize_t rfs_write(struct vinode *f_inode, const char *w_buf, ssize_t len,
                  int *offset) {
  if (f_inode->size < *offset) {
    panic("rfs_write:offset should less than file size!");
  }

  // compute how many blocks we need to write
  int align = *offset % RFS_BLKSIZE;
  int writetimes = (len + align) / RFS_BLKSIZE;
  int remain = (len + align) % RFS_BLKSIZE;

  int buf_offset = 0;
  int block_offset = *offset / RFS_BLKSIZE;

  struct rfs_device *rdev = rfs_device_list[f_inode->sb->s_dev->dev_id];

  // write first block
  if (align != 0) {
    rfs_r1block(rdev, f_inode->addrs[block_offset]);
    int first_block_len = (writetimes == 0 ? len : RFS_BLKSIZE - align);
    memcpy(rdev->iobuffer + align, w_buf, first_block_len);
    rfs_w1block(rdev, f_inode->addrs[block_offset]);

    buf_offset += first_block_len;
    block_offset++;
    writetimes--;
  }

  // writetimes < 0 means that the file has only one block (and not full),
  // so our work is done
  // otherwise...
  if (writetimes >= 0) {
    // write complete blocks
    while (writetimes != 0) {
      if (block_offset == f_inode->blocks) {  // need to create new block
        // allocate a free block for the file
        f_inode->addrs[block_offset] = rfs_alloc_block(f_inode->sb);
        f_inode->blocks++;
      }

      memcpy(rdev->iobuffer, w_buf + buf_offset, RFS_BLKSIZE);
      rfs_w1block(rdev, f_inode->addrs[block_offset]);

      buf_offset += RFS_BLKSIZE;
      block_offset++;
      writetimes--;
    }

    // write the remaining data
    if (remain > 0) {
      if (block_offset == f_inode->blocks) {
        f_inode->addrs[block_offset] = rfs_alloc_block(f_inode->sb);
        ++f_inode->blocks;
      }
      memcpy(rdev->iobuffer, w_buf + buf_offset, remain);
      rfs_w1block(rdev, f_inode->addrs[block_offset]);
    }
  }

  // update file size
  f_inode->size =
      (f_inode->size < *offset + len ? *offset + len : f_inode->size);

  *offset += len;
  return len;
}

//
// lookup a directory entry("sub_dentry") under "parent".
// note that this is a one level lookup ,and the vfs layer will call this
// function several times until the final file is found.
// return: if found, return its vinode, otherwise return NULL
//
struct vinode *rfs_lookup(struct vinode *parent, struct dentry *sub_dentry) {
  struct rfs_direntry *p_direntry = NULL;
  struct vinode *child_vinode = NULL;

  int total_direntrys = parent->size / sizeof(struct rfs_direntry);
  int one_block_direntrys = RFS_BLKSIZE / sizeof(struct rfs_direntry);

  struct rfs_device *rdev = rfs_device_list[parent->sb->s_dev->dev_id];

  // browse the dir entries contained in a directory file
  for (int i = 0; i < total_direntrys; ++i) {
    if (i % one_block_direntrys == 0) {  // read in the disk block at boundary
      rfs_r1block(rdev, parent->addrs[i / one_block_direntrys]);
      p_direntry = (struct rfs_direntry *)rdev->iobuffer;
    }
    if (strcmp(p_direntry->name, sub_dentry->name) == 0) {  // found
      child_vinode = rfs_alloc_vinode(parent->sb);
      child_vinode->inum = p_direntry->inum;
      if (rfs_update_vinode(child_vinode) != 0)
        panic("rfs_lookup: read inode failed!");
      break;
    }
    ++p_direntry;
  }
  return child_vinode;
}

//
// create a file with "sub_dentry->name" at directory "parent" in rfs.
// return the vfs inode of the file being created.
//
struct vinode *rfs_create(struct vinode *parent, struct dentry *sub_dentry) {
  struct rfs_device *rdev = rfs_device_list[parent->sb->s_dev->dev_id];

  // ** find a free disk inode to store the file that is going to be created
  struct rfs_dinode *free_dinode = NULL;
  int free_inum = 0;
  for (int i = 0; i < (RFS_BLKSIZE / RFS_INODESIZE * RFS_MAX_INODE_BLKNUM);
       ++i) {
    free_dinode = rfs_read_dinode(rdev, i);
    if (free_dinode->type == R_FREE) {  // found
      free_inum = i;
      break;
    }
    free_page(free_dinode);
  }

  if (free_dinode == NULL)
    panic("rfs_create: no more free disk inode, we cannot create file.\n" );

  // initialize the states of the file being created

  // TODO (lab4_1): implement the code for populating the disk inode (free_dinode) 
  // of a new file being created.
  // hint:  members of free_dinode to be filled are:
  // size, should be zero for a new file.
  // type, see kernel/rfs.h and find the type for a rfs file.
  // nlinks, i.e., the number of links.
  // blocks, i.e., its block count.
  // Note: DO NOT DELETE CODE BELOW PANIC.
  panic("You need to implement the code of populating a disk inode in lab4_1.\n" );

  // DO NOT REMOVE ANY CODE BELOW.
  // allocate a free block for the file
  free_dinode->addrs[0] = rfs_alloc_block(parent->sb);

  // **  write the disk inode of file being created to disk
  rfs_write_dinode(rdev, free_dinode, free_inum);
  free_page(free_dinode);

  // ** build vfs inode according to dinode
  struct vinode *new_vinode = rfs_alloc_vinode(parent->sb);
  new_vinode->inum = free_inum;
  rfs_update_vinode(new_vinode);

  // ** append the new file as a direntry to its parent dir
  int result = rfs_add_direntry(parent, sub_dentry->name, free_inum);
  if (result == -1) {
    sprint("rfs_create: rfs_add_direntry failed");
    return NULL;
  }

  return new_vinode;
}

//
// there are two types of seek (specify by whence): LSEEK_SET, SEEK_CUR
// LSEEK_SET: set the file pointer to the offset
// LSEEK_CUR: set the file pointer to the current offset plus the offset
// return 0 if success, otherwise return -1
//
int rfs_lseek(struct vinode *f_inode, ssize_t new_offset, int whence, int *offset) {
  int file_size = f_inode->size;

  switch (whence) {
    case LSEEK_SET:
      if (new_offset < 0 || new_offset > file_size) {
        sprint("rfs_lseek: invalid offset!\n");
        return -1;
      }
      *offset = new_offset;
      break;
    case LSEEK_CUR:
      if (*offset + new_offset < 0 || *offset + new_offset > file_size) {
        sprint("rfs_lseek: invalid offset!\n");
        return -1;
      }
      *offset += new_offset;
      break;
    default:
      sprint("rfs_lseek: invalid whence!\n");
      return -1;
  }
  
  return 0;
}

//
//  read disk inode information from disk
//
int rfs_disk_stat(struct vinode *vinode, struct istat *istat) {
  struct rfs_device *rdev = rfs_device_list[vinode->sb->s_dev->dev_id];
  struct rfs_dinode *dinode = rfs_read_dinode(rdev, vinode->inum);
  if (dinode == NULL) {
    sprint("rfs_disk_stat: read dinode failed!\n");
    return -1;
  }

  istat->st_inum = 1;
  istat->st_inum = vinode->inum;  // get inode number from vinode

  istat->st_size = dinode->size;
  istat->st_type = dinode->type;
  istat->st_nlinks = dinode->nlinks;
  istat->st_blocks = dinode->blocks;
  free_page(dinode);
  return 0;
}

//
// create a hard link under a direntry "parent" for an existing file of "link_node"
//
int rfs_link(struct vinode *parent, struct dentry *sub_dentry, struct vinode *link_node) {
  // TODO (lab4_3): we now need to establish a hard link to an existing file whose vfs
  // inode is "link_node". To do that, we need first to know the name of the new (link)
  // file, and then, we need to increase the link count of the existing file. Lastly, 
  // we need to make the changes persistent to disk. To know the name of the new (link)
  // file, you need to stuty the structure of dentry, that contains the name member;
  // To incease the link count of the existing file, you need to study the structure of
  // vfs inode, since it contains the inode information of the existing file.
  //
  // hint: to accomplish this experiment, you need to:
  // 1) increase the link count of the file to be hard-linked;
  // 2) append the new (link) file as a dentry to its parent directory; you can use 
  //    rfs_add_direntry here.
  // 3) persistent the changes to disk. you can use rfs_write_back_vinode here.
  //
  panic("You need to implement the code for creating a hard link in lab4_3.\n" );
}

//
// remove a hard link with "name" under a direntry "parent"
//
int rfs_unlink(struct vinode *parent, struct dentry *sub_dentry, struct vinode *unlink_vinode) {
  struct rfs_device *rdev = rfs_device_list[parent->sb->s_dev->dev_id];

  // ** find the direntry in the directory file
  int total_direntrys = parent->size / sizeof(struct rfs_direntry);
  int one_block_direntrys = RFS_BLKSIZE / sizeof(struct rfs_direntry);

  struct rfs_direntry *p_direntry = NULL;
  int delete_index;
  for (delete_index = 0; delete_index < total_direntrys; ++delete_index) {
    // read in the disk block at boundary
    if (delete_index % one_block_direntrys == 0) {
      rfs_r1block(rdev, parent->addrs[delete_index / one_block_direntrys]);
      p_direntry = (struct rfs_direntry *)rdev->iobuffer;
    }
    if (strcmp(p_direntry->name, sub_dentry->name) == 0) {  // found
      break;
    }
    ++p_direntry;
  }

  int inum = p_direntry->inum;

  if (delete_index == total_direntrys) {
    sprint("unlink: file %s not found.\n", sub_dentry->name);
    return -1;
  }

  // ** read the disk inode of the file to be unlinked
  struct rfs_dinode *unlink_dinode = rfs_read_dinode(rdev, inum);

  // if this assertion fails, it indicates that the previous modification to nlinks
  // was not written back to disk, which is not allowed
  assert(unlink_vinode->nlinks == unlink_dinode->nlinks);

  // ** decrease vinode nlinks by 1
  unlink_vinode->nlinks--;

  // ** update disk inode nlinks
  unlink_dinode->nlinks = unlink_vinode->nlinks;

  // ** if nlinks == 0, free the disk inode and disk blocks
  if (unlink_dinode->nlinks == 0) {
    // free disk blocks
    for (int i = 0; i < unlink_dinode->blocks; ++i) {
      rfs_free_block(parent->sb, unlink_dinode->addrs[i]);
    }
    // free disk inode
    unlink_dinode->type = R_FREE;
  }
  // ** write the disk inode back to disk
  rfs_write_dinode(rdev, unlink_dinode, inum);
  free_page(unlink_dinode);

  // ** remove the direntry from the directory

  // handle the first block
  int delete_block_index = delete_index / one_block_direntrys;
  rfs_r1block(rdev, parent->addrs[delete_block_index]);

  int offset = delete_index % one_block_direntrys;
  memmove(rdev->iobuffer + offset * sizeof(struct rfs_direntry),
          rdev->iobuffer + (offset + 1) * sizeof(struct rfs_direntry),
          (one_block_direntrys - offset - 1) * sizeof(struct rfs_direntry));

  struct rfs_direntry *previous_block = (struct rfs_direntry *)alloc_page();
  memcpy(previous_block, rdev->iobuffer, RFS_BLKSIZE);

  for (int i = delete_block_index + 1; i < parent->blocks; i++) {
    rfs_r1block(rdev, parent->addrs[i]);
    struct rfs_direntry *this_block = (struct rfs_direntry *)alloc_page();
    memcpy(this_block, rdev->iobuffer, RFS_BLKSIZE);

    // copy the first direntry of this block to the last direntry 
    // of previous block
    memcpy(previous_block + one_block_direntrys - 1, rdev->iobuffer,
           sizeof(struct rfs_direntry));

    // move the direntry in this block forward by one
    memmove(this_block, this_block + 1,
            (one_block_direntrys - 1) * sizeof(struct rfs_direntry));

    // write the previous block back to disk
    memcpy(rdev->iobuffer, previous_block, RFS_BLKSIZE);
    rfs_w1block(rdev, parent->addrs[i - 1]);

    // update previous block
    free_page(previous_block);
    previous_block = this_block;
  }

  // write the last block back to disk
  memcpy(rdev->iobuffer, previous_block, RFS_BLKSIZE);
  rfs_w1block(rdev, parent->addrs[parent->blocks - 1]);
  free_page(previous_block);

  // if the last block is empty, free it
  total_direntrys--;
  if (total_direntrys % one_block_direntrys == 0 && parent->blocks > 1) {
    rfs_free_block(parent->sb, parent->addrs[parent->blocks - 1]);
    parent->blocks--;
  }

  // ** update the directory file's size
  parent->size -= sizeof(struct rfs_direntry);

  // ** write the directory file's inode back to disk
  if (rfs_write_back_vinode(parent) != 0) {
    sprint("rfs_unlink: rfs_write_back_vinode failed");
    return -1;
  }

  return 0;
}

//
// when a directory is opened, the contents of the directory file are read
// into the memory for directory read operations
//
int rfs_hook_opendir(struct vinode *dir_vinode, struct dentry *dentry) {
  // allocate space and read the contents of the dir block into memory
  void *pdire = NULL;
  void *previous = NULL;
  struct rfs_device *rdev = rfs_device_list[dir_vinode->sb->s_dev->dev_id];

  // read-in the directory file, store all direntries in dir cache.
  for (int i = dir_vinode->blocks - 1; i >= 0; i--) {
    previous = pdire;
    pdire = alloc_page();

    if (previous != NULL && previous - pdire != RFS_BLKSIZE)
      panic("rfs_hook_opendir: memory discontinuity");

    rfs_r1block(rdev, dir_vinode->addrs[i]);
    memcpy(pdire, rdev->iobuffer, RFS_BLKSIZE);
  }

  // save the pointer to the directory block in the vinode
  struct rfs_dir_cache *dir_cache = (struct rfs_dir_cache *)alloc_page();
  dir_cache->block_count = dir_vinode->blocks;
  dir_cache->dir_base_addr = (struct rfs_direntry *)pdire;

  dir_vinode->i_fs_info = dir_cache;

  return 0;
}

//
// when a directory is closed, the memory space allocated for the directory
// block is freed
//
int rfs_hook_closedir(struct vinode *dir_vinode, struct dentry *dentry) {
  struct rfs_dir_cache *dir_cache =
      (struct rfs_dir_cache *)dir_vinode->i_fs_info;

  // reclaim the dir cache
  for (int i = 0; i < dir_cache->block_count; ++i) {
    free_page((char *)dir_cache->dir_base_addr + i * RFS_BLKSIZE);
  }
  return 0;
}

//
// read a directory entry from the directory "dir", and the "offset" indicate
// the position of the entry to be read. if offset is 0, the first entry is read,
// if offset is 1, the second entry is read, and so on.
// return: 0 on success, -1 when there are no more entry (end of the list).
//
int rfs_readdir(struct vinode *dir_vinode, struct dir *dir, int *offset) {
  int total_direntrys = dir_vinode->size / sizeof(struct rfs_direntry);
  int one_block_direntrys = RFS_BLKSIZE / sizeof(struct rfs_direntry);

  int direntry_index = *offset;
  if (direntry_index >= total_direntrys) {
    // no more direntry
    return -1;
  }

  // reads a directory entry from the directory cache stored in vfs inode.
  struct rfs_dir_cache *dir_cache =
      (struct rfs_dir_cache *)dir_vinode->i_fs_info;
  struct rfs_direntry *p_direntry = dir_cache->dir_base_addr + direntry_index;

  // TODO (lab4_2): implement the code to read a directory entry.
  // hint: in the above code, we had found the directory entry that located at the
  // *offset, and used p_direntry to point it.
  // in the remaining processing, we need to return our discovery.
  // the method of returning is to popular proper members of "dir", more specifically,
  // dir->name and dir->inum.
  // note: DO NOT DELETE CODE BELOW PANIC.
  panic("You need to implement the code for reading a directory entry of rfs in lab4_2.\n" );

  // DO NOT DELETE CODE BELOW.
  (*offset)++;
  return 0;
}

//
// make a new direntry named "sub_dentry->name" under the directory "parent",
// return the vfs inode of subdir being created.
//
struct vinode *rfs_mkdir(struct vinode *parent, struct dentry *sub_dentry) {
  struct rfs_device *rdev = rfs_device_list[parent->sb->s_dev->dev_id];

  // ** find a free disk inode to store the file that is going to be created
  struct rfs_dinode *free_dinode = NULL;
  int free_inum = 0;
  for (int i = 0; i < (RFS_BLKSIZE / RFS_INODESIZE * RFS_MAX_INODE_BLKNUM); i++) {
    free_dinode = rfs_read_dinode(rdev, i);
    if (free_dinode->type == R_FREE) {  // found
      free_inum = i;
      break;
    }
    free_page(free_dinode);
  }

  if (free_dinode == NULL)
    panic( "rfs_mkdir: no more free disk inode, we cannot create directory.\n" );

  // initialize the states of the file being created
  free_dinode->size = 0;
  free_dinode->type = R_DIR;
  free_dinode->nlinks = 1;
  free_dinode->blocks = 1;
  // allocate a free block for the file
  free_dinode->addrs[0] = rfs_alloc_block(parent->sb);

  // **  write the disk inode of file being created to disk
  rfs_write_dinode(rdev, free_dinode, free_inum);
  free_page(free_dinode);

  // ** add a direntry to the directory
  int result = rfs_add_direntry(parent, sub_dentry->name, free_inum);
  if (result == -1) {
    sprint("rfs_mkdir: rfs_add_direntry failed");
    return NULL;
  }

  // ** allocate a new vinode
  struct vinode *sub_vinode = rfs_alloc_vinode(parent->sb);
  sub_vinode->inum = free_inum;
  rfs_update_vinode(sub_vinode);

  return sub_vinode;
}

/**** vfs-rfs file system type interface functions ****/
struct super_block *rfs_get_superblock(struct device *dev) {
  struct rfs_device *rdev = rfs_device_list[dev->dev_id];

  // read super block from ramdisk
  if (rfs_r1block(rdev, RFS_BLK_OFFSET_SUPER) != 0)
    panic("RFS: failed to read superblock!\n");

  struct rfs_superblock d_sb;
  memcpy(&d_sb, rdev->iobuffer, sizeof(struct rfs_superblock));

  // set the data for the vfs super block
  struct super_block *sb = alloc_page();
  sb->magic = d_sb.magic;
  sb->size = d_sb.size;
  sb->nblocks = d_sb.nblocks;
  sb->ninodes = d_sb.ninodes;
  sb->s_dev = dev;

  if( sb->magic != RFS_MAGIC ) 
    panic("rfs_get_superblock: wrong ramdisk device!\n");

  // build root dentry and root inode
  struct vinode *root_inode = rfs_alloc_vinode(sb);
  root_inode->inum = 0;
  rfs_update_vinode(root_inode);

  struct dentry *root_dentry = alloc_vfs_dentry("/", root_inode, NULL);
  sb->s_root = root_dentry;

  // save the bitmap in the s_fs_info field
  if (rfs_r1block(rdev, RFS_BLK_OFFSET_BITMAP) != 0)
    panic("RFS: failed to read bitmap!\n");
  void *bitmap = alloc_page();
  memcpy(bitmap, rdev->iobuffer, RFS_BLKSIZE);
  sb->s_fs_info = bitmap;

  return sb;
}
```

- rfs.h
```c
#ifndef _RFS_H_
#define _RFS_H_

#include "ramdev.h"
#include "riscv.h"
#include "util/types.h"
#include "vfs.h"

#define RFS_TYPE 0
#define RFS_MAGIC 0xBEAF
#define RFS_BLKSIZE PGSIZE
#define RFS_INODESIZE 128  // block size must be divisible by this value
#define RFS_MAX_INODE_BLKNUM 10
#define RFS_MAX_FILE_NAME_LEN 28
#define RFS_DIRECT_BLKNUM DIRECT_BLKNUM

// rfs block offset
#define RFS_BLK_OFFSET_SUPER 0
#define RFS_BLK_OFFSET_INODE 1
#define RFS_BLK_OFFSET_BITMAP 11
#define RFS_BLK_OFFSET_FREE 12

// dinode type
#define R_FILE FILE_I
#define R_DIR DIR_I
#define R_FREE 2

// file system super block
struct rfs_superblock {
  int magic;    // magic number of the
  int size;     // size of file system image (blocks)
  int nblocks;  // number of data blocks
  int ninodes;  // number of inodes.
};

// disk inode
struct rfs_dinode {
  int size;                      // size of the file (in bytes)
  int type;                      // one of R_FREE, R_FILE, R_DIR
  int nlinks;                    // number of hard links to this file
  int blocks;                    // number of blocks
  int addrs[RFS_DIRECT_BLKNUM];  // direct blocks
};

// directory entry
struct rfs_direntry {
  int inum;                          // inode number
  char name[RFS_MAX_FILE_NAME_LEN];  // file name
};

// directory memory cache (used by opendir/readdir/closedir)
struct rfs_dir_cache {
  int block_count;
  struct rfs_direntry *dir_base_addr;
};

// rfs utility functin declarations
int register_rfs();
int rfs_format_dev(struct device *dev);

int rfs_r1block(struct rfs_device *rfs_dev, int n_block);
int rfs_w1block(struct rfs_device *rfs_dev, int n_block);
struct rfs_dinode *rfs_read_dinode(struct rfs_device *rdev, int n_inode);
int rfs_write_dinode(struct rfs_device *rdev, const struct rfs_dinode *dinode,
                     int n_inode);
int rfs_alloc_block(struct super_block *sb);
int rfs_free_block(struct super_block *sb, int block_num);
int rfs_add_direntry(struct vinode *dir, const char *name, int inum);

struct vinode *rfs_alloc_vinode(struct super_block *sb);
int rfs_write_back_vinode(struct vinode *vinode);
int rfs_update_vinode(struct vinode *vinode);

// rfs interface function declarations
ssize_t rfs_read(struct vinode *f_inode, char *r_buf, ssize_t len, int *offset);
ssize_t rfs_write(struct vinode *f_inode, const char *w_buf, ssize_t len,
                  int *offset);
struct vinode *rfs_lookup(struct vinode *parent, struct dentry *sub_dentry);
struct vinode *rfs_create(struct vinode *parent, struct dentry *sub_dentry);
int rfs_lseek(struct vinode *f_inode, ssize_t new_offset, int whence, int *offset);
int rfs_disk_stat(struct vinode *vinode, struct istat *istat);
int rfs_link(struct vinode *parent, struct dentry *sub_dentry, struct vinode *link_node);
int rfs_unlink(struct vinode *parent, struct dentry *sub_dentry, struct vinode *unlink_vinode);

int rfs_hook_opendir(struct vinode *dir_vinode, struct dentry *dentry);
int rfs_hook_closedir(struct vinode *dir_vinode, struct dentry *dentry);
int rfs_readdir(struct vinode *dir_vinode, struct dir *dir, int *offset);
struct vinode *rfs_mkdir(struct vinode *parent, struct dentry *sub_dentry);

struct super_block *rfs_get_superblock(struct device *dev);

extern const struct vinode_ops rfs_i_ops;

#endif
```

- sched.c
```c
/*
 * implementing the scheduler
 */

#include "sched.h"
#include "spike_interface/spike_utils.h"

process* ready_queue_head = NULL;

//
// insert a process, proc, into the END of ready queue.
//
void insert_to_ready_queue( process* proc ) {
  sprint( "going to insert process %d to ready queue.\n", proc->pid );
  // if the queue is empty in the beginning
  if( ready_queue_head == NULL ){
    proc->status = READY;
    proc->queue_next = NULL;
    ready_queue_head = proc;
    return;
  }

  // ready queue is not empty
  process *p;
  int count = 0;
  // browse the ready queue to see if proc is already in-queue
  // Add counter to prevent infinite loop in case of circular queue
  for( p=ready_queue_head; p->queue_next!=NULL && count < NPROC; p=p->queue_next, count++ )
    if( p == proc ) return;  //already in queue
  
  if (count >= NPROC) {
    panic("insert_to_ready_queue: circular queue detected!\n");
  }

  // p points to the last element of the ready queue
  if( p==proc ) return;
  p->queue_next = proc;
  proc->status = READY;
  proc->queue_next = NULL;

  return;
}

//
// choose a proc from the ready queue, and put it to run.
// note: schedule() does not take care of previous current process. If the current
// process is still runnable, you should place it into the ready queue (by calling
// ready_queue_insert), and then call schedule().
//
extern process procs[NPROC];
void schedule() {
  if ( !ready_queue_head ){
    // by default, if there are no ready process, and all processes are in the status of
    // FREE and ZOMBIE, we should shutdown the emulated RISC-V machine.
    int should_shutdown = 1;

    for( int i=0; i<NPROC; i++ )
      if( (procs[i].status != FREE) && (procs[i].status != ZOMBIE) ){
        should_shutdown = 0;
        sprint( "ready queue empty, but process %d is not in free/zombie state:%d\n", 
          i, procs[i].status );
      }

    if( should_shutdown ){
      sprint( "no more ready processes, system shutdown now.\n" );
      shutdown( 0 );
    }else{
      panic( "Not handled: we should let system wait for unfinished processes.\n" );
    }
  }

  current = ready_queue_head;
  assert( current->status == READY );
  ready_queue_head = ready_queue_head->queue_next;

  current->status = RUNNING;
  sprint( "going to schedule process %d to run.\n", current->pid );
  switch_to( current );
}
```

- sched.h
```c
#ifndef _SCHED_H_
#define _SCHED_H_

#include "process.h"

//length of a time slice, in number of ticks
#define TIME_SLICE_LEN  2

void insert_to_ready_queue( process* proc );
void schedule();

#endif
```

- vmm.c
```c
/*
 * virtual address mapping related functions.
 */

#include "vmm.h"
#include "riscv.h"
#include "pmm.h"
#include "util/types.h"
#include "memlayout.h"
#include "util/string.h"
#include "spike_interface/spike_utils.h"
#include "util/functions.h"

/* --- utility functions for virtual address mapping --- */
//
// establish mapping of virtual address [va, va+size] to phyiscal address [pa, pa+size]
// with the permission of "perm".
//
int map_pages(pagetable_t page_dir, uint64 va, uint64 size, uint64 pa, int perm) {
  uint64 first, last;
  pte_t *pte;

  for (first = ROUNDDOWN(va, PGSIZE), last = ROUNDDOWN(va + size - 1, PGSIZE);
      first <= last; first += PGSIZE, pa += PGSIZE) {
    if ((pte = page_walk(page_dir, first, 1)) == 0) return -1;
    if (*pte & PTE_V)
      panic("map_pages fails on mapping va (0x%lx) to pa (0x%lx)", first, pa);
    *pte = PA2PTE(pa) | perm | PTE_V;
  }
  return 0;
}

//
// convert permission code to permission types of PTE
//
uint64 prot_to_type(int prot, int user) {
  uint64 perm = 0;
  if (prot & PROT_READ) perm |= PTE_R | PTE_A;
  if (prot & PROT_WRITE) perm |= PTE_W | PTE_D;
  if (prot & PROT_EXEC) perm |= PTE_X | PTE_A;
  if (perm == 0) perm = PTE_R;
  if (user) perm |= PTE_U;
  return perm;
}

//
// traverse the page table (starting from page_dir) to find the corresponding pte of va.
// returns: PTE (page table entry) pointing to va.
//
pte_t *page_walk(pagetable_t page_dir, uint64 va, int alloc) {
  if (va >= MAXVA) panic("page_walk");

  // starting from the page directory
  pagetable_t pt = page_dir;

  // traverse from page directory to page table.
  // as we use risc-v sv39 paging scheme, there will be 3 layers: page dir,
  // page medium dir, and page table.
  for (int level = 2; level > 0; level--) {
    // macro "PX" gets the PTE index in page table of current level
    // "pte" points to the entry of current level
    pte_t *pte = pt + PX(level, va);

    // now, we need to know if above pte is valid (established mapping to a phyiscal page)
    // or not.
    if (*pte & PTE_V) {  //PTE valid
      // phisical address of pagetable of next level
      pt = (pagetable_t)PTE2PA(*pte);
    } else { //PTE invalid (not exist).
      // allocate a page (to be the new pagetable), if alloc == 1
      if( alloc && ((pt = (pte_t *)alloc_page(1)) != 0) ){
        memset(pt, 0, PGSIZE);
        // writes the physical address of newly allocated page to pte, to establish the
        // page table tree.
        *pte = PA2PTE(pt) | PTE_V;
      }else //returns NULL, if alloc == 0, or no more physical page remains
        return 0;
    }
  }

  // return a PTE which contains phisical address of a page
  return pt + PX(0, va);
}

//
// look up a virtual page address, return the physical page address or 0 if not mapped.
//
uint64 lookup_pa(pagetable_t pagetable, uint64 va) {
  pte_t *pte;
  uint64 pa;

  if (va >= MAXVA) return 0;

  pte = page_walk(pagetable, va, 0);
  if (pte == 0 || (*pte & PTE_V) == 0 || ((*pte & PTE_R) == 0 && (*pte & PTE_W) == 0))
    return 0;
  pa = PTE2PA(*pte);

  return pa;
}

/* --- kernel page table part --- */
// _etext is defined in kernel.lds, it points to the address after text and rodata segments.
extern char _etext[];

// pointer to kernel page director
pagetable_t g_kernel_pagetable;

//
// maps virtual address [va, va+sz] to [pa, pa+sz] (for kernel).
//
void kern_vm_map(pagetable_t page_dir, uint64 va, uint64 pa, uint64 sz, int perm) {
  // map_pages is defined in kernel/vmm.c
  if (map_pages(page_dir, va, sz, pa, perm) != 0) panic("kern_vm_map");
}

//
// kern_vm_init() constructs the kernel page table.
//
void kern_vm_init(void) {
  // pagetable_t is defined in kernel/riscv.h. it's actually uint64*
  pagetable_t t_page_dir;

  // allocate a page (t_page_dir) to be the page directory for kernel. alloc_page is defined in kernel/pmm.c
  t_page_dir = (pagetable_t)alloc_page();
  // memset is defined in util/string.c
  memset(t_page_dir, 0, PGSIZE);

  // map virtual address [KERN_BASE, _etext] to physical address [DRAM_BASE, DRAM_BASE+(_etext - KERN_BASE)],
  // to maintain (direct) text section kernel address mapping.
  kern_vm_map(t_page_dir, KERN_BASE, DRAM_BASE, (uint64)_etext - KERN_BASE,
         prot_to_type(PROT_READ | PROT_EXEC, 0));

  sprint("KERN_BASE 0x%lx\n", lookup_pa(t_page_dir, KERN_BASE));

  // also (direct) map remaining address space, to make them accessable from kernel.
  // this is important when kernel needs to access the memory content of user's app
  // without copying pages between kernel and user spaces.
  kern_vm_map(t_page_dir, (uint64)_etext, (uint64)_etext, PHYS_TOP - (uint64)_etext,
         prot_to_type(PROT_READ | PROT_WRITE, 0));

  sprint("physical address of _etext is: 0x%lx\n", lookup_pa(t_page_dir, (uint64)_etext));

  g_kernel_pagetable = t_page_dir;
}

/* --- user page table part --- */
//
// convert and return the corresponding physical address of a virtual address (va) of
// application.
//
void *user_va_to_pa(pagetable_t page_dir, void *va) {
  // TODO (lab2_1): implement user_va_to_pa to convert a given user virtual address "va"
  // to its corresponding physical address, i.e., "pa". To do it, we need to walk
  // through the page table, starting from its directory "page_dir", to locate the PTE
  // that maps "va". If found, returns the "pa" by using:
  // pa = PYHS_ADDR(PTE) + (va & (1<<PGSHIFT -1))
  // Here, PYHS_ADDR() means retrieving the starting address (4KB aligned), and
  // (va & (1<<PGSHIFT -1)) means computing the offset of "va" inside its page.
  // Also, it is possible that "va" is not mapped at all. in such case, we can find
  // invalid PTE, and should return NULL.
  panic( "You have to implement user_va_to_pa (convert user va to pa) to print messages in lab2_1.\n" );

}

//
// maps virtual address [va, va+sz] to [pa, pa+sz] (for user application).
//
void user_vm_map(pagetable_t page_dir, uint64 va, uint64 size, uint64 pa, int perm) {
  if (map_pages(page_dir, va, size, pa, perm) != 0) {
    panic("fail to user_vm_map .\n");
  }
}

//
// unmap virtual address [va, va+size] from the user app.
// reclaim the physical pages if free!=0
//
void user_vm_unmap(pagetable_t page_dir, uint64 va, uint64 size, int free) {
  // TODO (lab2_2): implement user_vm_unmap to disable the mapping of the virtual pages
  // in [va, va+size], and free the corresponding physical pages used by the virtual
  // addresses when if 'free' (the last parameter) is not zero.
  // basic idea here is to first locate the PTEs of the virtual pages, and then reclaim
  // (use free_page() defined in pmm.c) the physical pages. lastly, invalidate the PTEs.
  // as naive_free reclaims only one page at a time, you only need to consider one page
  // to make user/app_naive_malloc to behave correctly.
  panic( "You have to implement user_vm_unmap to free pages using naive_free in lab2_2.\n" );

}

//
// debug function, print the vm space of a process. added @lab3_1
//
void print_proc_vmspace(process* proc) {
  sprint( "======\tbelow is the vm space of process%d\t========\n", proc->pid );
  for( int i=0; i<proc->total_mapped_region; i++ ){
    sprint( "-va:%lx, npage:%d, ", proc->mapped_info[i].va, proc->mapped_info[i].npages);
    switch(proc->mapped_info[i].seg_type){
      case CODE_SEGMENT: sprint( "type: CODE SEGMENT" ); break;
      case DATA_SEGMENT: sprint( "type: DATA SEGMENT" ); break;
      case STACK_SEGMENT: sprint( "type: STACK SEGMENT" ); break;
      case CONTEXT_SEGMENT: sprint( "type: TRAPFRAME SEGMENT" ); break;
      case SYSTEM_SEGMENT: sprint( "type: USER KERNEL STACK SEGMENT" ); break;
    }
    sprint( ", mapped to pa:%lx\n", lookup_pa(proc->pagetable, proc->mapped_info[i].va) );
  }
}
```

- vmm.h
```c
#ifndef _VMM_H_
#define _VMM_H_

#include "riscv.h"
#include "process.h"

/* --- utility functions for virtual address mapping --- */
int map_pages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm);
// permission codes.
enum VMPermision {
  PROT_NONE = 0,
  PROT_READ = 1,
  PROT_WRITE = 2,
  PROT_EXEC = 4,
};

uint64 prot_to_type(int prot, int user);
pte_t *page_walk(pagetable_t pagetable, uint64 va, int alloc);
uint64 lookup_pa(pagetable_t pagetable, uint64 va);

/* --- kernel page table --- */
// pointer to kernel page directory
extern pagetable_t g_kernel_pagetable;

void kern_vm_map(pagetable_t page_dir, uint64 va, uint64 pa, uint64 sz, int perm);

// Initialize the kernel pagetable
void kern_vm_init(void);

/* --- user page table --- */
void *user_va_to_pa(pagetable_t page_dir, void *va);
void user_vm_map(pagetable_t page_dir, uint64 va, uint64 size, uint64 pa, int perm);
void user_vm_unmap(pagetable_t page_dir, uint64 va, uint64 size, int free);
void print_proc_vmspace(process* proc);

#endif
```

- vfs.c
```c
/*
 * VFS (Virtual File System) interface utilities. added @lab4_1.
 */

#include "vfs.h"

#include "pmm.h"
#include "spike_interface/spike_utils.h"
#include "util/string.h"
#include "util/types.h"
#include "util/hash_table.h"

struct dentry *vfs_root_dentry;               // system root direntry
struct super_block *vfs_sb_list[MAX_MOUNTS];  // system superblock list
struct device *vfs_dev_list[MAX_VFS_DEV];     // system device list in vfs layer
struct hash_table dentry_hash_table;
struct hash_table vinode_hash_table;

//
// initializes the dentry hash list and vinode hash list
//
int vfs_init() {
  int ret;
  ret = hash_table_init(&dentry_hash_table, dentry_hash_equal, dentry_hash_func,
                            NULL, NULL, NULL);
  if (ret != 0) return ret;

  ret = hash_table_init(&vinode_hash_table, vinode_hash_equal, vinode_hash_func,
                            NULL, NULL, NULL);
  if (ret != 0) return ret;
  return 0; 
}

//
// mount a file system from the device named "dev_name"
// PKE does not support mounting a device at an arbitrary directory as in Linux,
// but can only mount a device in one of the following two ways (according to 
// the mnt_type parameter) :
// 1. when mnt_type = MOUNT_AS_ROOT
//    Mount the device AS the root directory.
//    that is, mount the device under system root direntry:"/".
//    In this case, the device specified by parameter dev_name will be used as
//    the root file system.
// 2. when mnt_type = MOUNT_DEFAULT
//    Mount the device UNDER the root directory.
//    that is, mount the device to "/DEVICE_NAME" (/DEVICE_NAME will be
//    automatically created) folder.
//
struct super_block *vfs_mount(const char *dev_name, int mnt_type) {
  // device pointer
  struct device *p_device = NULL;

  // find the device entry in vfs_device_list named as dev_name
  for (int i = 0; i < MAX_VFS_DEV; ++i) {
    p_device = vfs_dev_list[i];
    if (p_device && strcmp(p_device->dev_name, dev_name) == 0) break;
  }
  if (p_device == NULL) panic("vfs_mount: cannot find the device entry!\n");

  // add the super block into vfs_sb_list
  struct file_system_type *fs_type = p_device->fs_type;
  struct super_block *sb = fs_type->get_superblock(p_device);

  // add the root vinode into vinode_hash_table
  hash_put_vinode(sb->s_root->dentry_inode);

  int err = 1;
  for (int i = 0; i < MAX_MOUNTS; ++i) {
    if (vfs_sb_list[i] == NULL) {
      vfs_sb_list[i] = sb;
      err = 0;
      break;
    }
  }
  if (err) panic("vfs_mount: too many mounts!\n");

  // mount the root dentry of the file system to right place
  if (mnt_type == MOUNT_AS_ROOT) {
    vfs_root_dentry = sb->s_root;

    // insert the mount point into hash table
    hash_put_dentry(sb->s_root);
  } else if (mnt_type == MOUNT_DEFAULT) {
    if (!vfs_root_dentry)
      panic("vfs_mount: root dentry not found, please mount the root device first!\n");

    struct dentry *mnt_point = sb->s_root;

    // set the mount point directory's name to device name
    char *dev_name = p_device->dev_name;
    strcpy(mnt_point->name, dev_name);

    // by default, it is mounted under the vfs root directory
    mnt_point->parent = vfs_root_dentry;

    // insert the mount point into hash table
    hash_put_dentry(sb->s_root);
  } else {
    panic("vfs_mount: unknown mount type!\n");
  }

  return sb;
}

//
// open a file located at "path" with permission of "flags".
// if the file does not exist, and O_CREAT bit is set in "flags", the file will
// be created.
// return: the file pointer to the opened file.
//
struct file *vfs_open(const char *path, int flags) {
  struct dentry *parent = vfs_root_dentry; // we start the path lookup from root.
  char miss_name[MAX_PATH_LEN];

  // path lookup.
  struct dentry *file_dentry = lookup_final_dentry(path, &parent, miss_name);

  // file does not exist
  if (!file_dentry) {
    int creatable = flags & O_CREAT;

    // create the file if O_CREAT bit is set
    if (creatable) {
      char basename[MAX_PATH_LEN];
      get_base_name(path, basename);

      // a missing directory exists in the path
      if (strcmp(miss_name, basename) != 0) {
        sprint("vfs_open: cannot create file in a non-exist directory!\n");
        return NULL;
      }

      // create the file
      file_dentry = alloc_vfs_dentry(basename, NULL, parent);
      struct vinode *new_inode = viop_create(parent->dentry_inode, file_dentry);
      if (!new_inode) panic("vfs_open: cannot create file!\n");

      file_dentry->dentry_inode = new_inode;
      new_inode->ref++;
      hash_put_dentry(file_dentry);
      hash_put_vinode(new_inode); 
    } else {
      sprint("vfs_open: cannot find the file!\n");
      return NULL;
    }
  }

  if (file_dentry->dentry_inode->type != FILE_I) {
    sprint("vfs_open: cannot open a directory!\n");
    return NULL;
  }

  // get writable and readable flags
  int writable = 0;
  int readable = 0;
  switch (flags & MASK_FILEMODE) {
    case O_RDONLY:
      writable = 0;
      readable = 1;
      break;
    case O_WRONLY:
      writable = 1;
      readable = 0;
      break;
    case O_RDWR:
      writable = 1;
      readable = 1;
      break;
    default:
      panic("fs_open: invalid open flags!\n");
  }

  struct file *file = alloc_vfs_file(file_dentry, readable, writable, 0);

  // additional open operations for a specific file system
  // hostfs needs to conduct actual file open.
  if (file_dentry->dentry_inode->i_ops->viop_hook_open) {
    if (file_dentry->dentry_inode->i_ops->
        viop_hook_open(file_dentry->dentry_inode, file_dentry) < 0) {
      sprint("vfs_open: hook_open failed!\n");
    }
  }

  return file;
}

//
// read content from "file" starting from file->offset, and store it in "buf".
// return: the number of bytes actually read
//
ssize_t vfs_read(struct file *file, char *buf, size_t count) {
  if (!file->readable) {
    sprint("vfs_read: file is not readable!\n");
    return -1;
  }
  if (file->f_dentry->dentry_inode->type != FILE_I) {
    sprint("vfs_read: cannot read a directory!\n");
    return -1;
  }
  // actual reading.
  return viop_read(file->f_dentry->dentry_inode, buf, count, &(file->offset));
}

//
// write content in "buf" to "file", at file->offset.
// return: the number of bytes actually written
//
ssize_t vfs_write(struct file *file, const char *buf, size_t count) {
  if (!file->writable) {
    sprint("vfs_write: file is not writable!\n");
    return -1;
  }
  if (file->f_dentry->dentry_inode->type != FILE_I) {
    sprint("vfs_read: cannot write a directory!\n");
    return -1;
  }
  // actual writing.
  return viop_write(file->f_dentry->dentry_inode, buf, count, &(file->offset));
}

//
// reposition read/write file offset
// return: the new offset on success, -1 on failure.
//
ssize_t vfs_lseek(struct file *file, ssize_t offset, int whence) {
  if (file->f_dentry->dentry_inode->type != FILE_I) {
    sprint("vfs_read: cannot seek a directory!\n");
    return -1;
  }

  if (viop_lseek(file->f_dentry->dentry_inode, offset, whence, &(file->offset)) != 0) {
    sprint("vfs_lseek: lseek failed!\n");
    return -1;
  }

  return file->offset;
}

//
// read the vinode information
//
int vfs_stat(struct file *file, struct istat *istat) {
  istat->st_inum = file->f_dentry->dentry_inode->inum;
  istat->st_size = file->f_dentry->dentry_inode->size;
  istat->st_type = file->f_dentry->dentry_inode->type;
  istat->st_nlinks = file->f_dentry->dentry_inode->nlinks;
  istat->st_blocks = file->f_dentry->dentry_inode->blocks;
  return 0;
}

//
// read the inode information on the disk
//
int vfs_disk_stat(struct file *file, struct istat *istat) {
  return viop_disk_stat(file->f_dentry->dentry_inode, istat);
}

//
// make hard link to the file specified by "oldpath" with the name "newpath"
// return: -1 on failure, 0 on success.
//
int vfs_link(const char *oldpath, const char *newpath) {
  struct dentry *parent = vfs_root_dentry;
  char miss_name[MAX_PATH_LEN];

  // lookup oldpath
  struct dentry *old_file_dentry =
      lookup_final_dentry(oldpath, &parent, miss_name);
  if (!old_file_dentry) {
    sprint("vfs_link: cannot find the file!\n");
    return -1;
  }

  if (old_file_dentry->dentry_inode->type != FILE_I) {
    sprint("vfs_link: cannot link a directory!\n");
    return -1;
  }

  parent = vfs_root_dentry;
  // lookup the newpath
  // note that parent is changed to be the last directory entry to be accessed
  struct dentry *new_file_dentry =
      lookup_final_dentry(newpath, &parent, miss_name);
  if (new_file_dentry) {
    sprint("vfs_link: the new file already exists!\n");
    return -1;
  }

  char basename[MAX_PATH_LEN];
  get_base_name(newpath, basename);
  if (strcmp(miss_name, basename) != 0) {
    sprint("vfs_link: cannot create file in a non-exist directory!\n");
    return -1;
  }

  // do the real hard-link
  new_file_dentry = alloc_vfs_dentry(basename, old_file_dentry->dentry_inode, parent);
  int err =
      viop_link(parent->dentry_inode, new_file_dentry, old_file_dentry->dentry_inode);
  if (err) return -1;

  // make a new dentry for the new link
  hash_put_dentry(new_file_dentry);

  return 0;
}

//
// unlink (delete) a file specified by "path".
// return: -1 on failure, 0 on success.
//
int vfs_unlink(const char *path) {
  struct dentry *parent = vfs_root_dentry;
  char miss_name[MAX_PATH_LEN];

  // lookup the file, find its parent direntry
  struct dentry *file_dentry = lookup_final_dentry(path, &parent, miss_name);
  if (!file_dentry) {
    sprint("vfs_unlink: cannot find the file!\n");
    return -1;
  }

  if (file_dentry->dentry_inode->type != FILE_I) {
    sprint("vfs_unlink: cannot unlink a directory!\n");
    return -1;
  }

  if (file_dentry->d_ref > 0) {
    sprint("vfs_unlink: the file is still opened!\n");
    return -1;
  }

  // do the real unlink
  struct vinode *unlinked_vinode = file_dentry->dentry_inode;
  int err = viop_unlink(parent->dentry_inode, file_dentry, unlinked_vinode);
  if (err) return -1;

  // remove the dentry from the hash table
  hash_erase_dentry(file_dentry);
  free_vfs_dentry(file_dentry);
  unlinked_vinode->ref--; 

  // if this inode has been removed from disk
  if (unlinked_vinode->nlinks == 0) {
    // no one will remember a dead inode
    assert(unlinked_vinode->ref == 0);

    // we don't write back the inode, because it has disappeared from the disk
    hash_erase_vinode(unlinked_vinode);
    free_page(unlinked_vinode);  // free the vinode
  }
  

  return 0;
}

//
// close a file at vfs layer.
//
int vfs_close(struct file *file) {
  if (file->f_dentry->dentry_inode->type != FILE_I) {
    sprint("vfs_close: cannot close a directory!\n");
    return -1;
  }

  struct dentry *dentry = file->f_dentry;
  struct vinode *inode = dentry->dentry_inode;

  // additional close operations for a specific file system
  // hostfs needs to conduct actual file close.
  if (inode->i_ops->viop_hook_close) {
    if (inode->i_ops->viop_hook_close(inode, dentry) != 0) {
      sprint("vfs_close: hook_close failed!\n");
    }
  }

  dentry->d_ref--;
  // if the dentry is not pointed by any opened file, free the dentry
  if (dentry->d_ref == 0) {
    // free the dentry
    hash_erase_dentry(dentry);
    free_vfs_dentry(dentry);
    inode->ref--;
    // no other opened hard link
    if (inode->ref == 0) {
      // write back the inode and free it
      if (viop_write_back_vinode(inode) != 0)
        panic("vfs_close: free inode failed!\n");
      hash_erase_vinode(inode);
      free_page(inode);
    }
  }

  file->status = FD_NONE;
  return 0;
}

//
// open a dir at vfs layer. the directory must exist on disk.
//
struct file *vfs_opendir(const char *path) {
  struct dentry *parent = vfs_root_dentry;
  char miss_name[MAX_PATH_LEN];

  // lookup the dir
  struct dentry *file_dentry = lookup_final_dentry(path, &parent, miss_name);

  if (!file_dentry || file_dentry->dentry_inode->type != DIR_I) {
    sprint("vfs_opendir: cannot find the direntry!\n");
    return NULL;
  }

  // allocate a vfs file with readable/non-writable flag.
  struct file *file = alloc_vfs_file(file_dentry, 1, 0, 0);

  // additional open direntry operations for a specific file system
  // rfs needs duild dir cache.
  if (file_dentry->dentry_inode->i_ops->viop_hook_opendir) {
    if (file_dentry->dentry_inode->i_ops->
        viop_hook_opendir(file_dentry->dentry_inode, file_dentry) != 0) {
      sprint("vfs_opendir: hook opendir failed!\n");
    }
  }

  return file;
}

//
// read a direntry entry from a direntry specified by "file"
// the read direntry entry is stored in "dir"
//
int vfs_readdir(struct file *file, struct dir *dir) {
  if (file->f_dentry->dentry_inode->type != DIR_I) {
    sprint("vfs_readdir: cannot read a file!\n");
    return -1;
  }
  return viop_readdir(file->f_dentry->dentry_inode, dir, &(file->offset));
}

//
// make a new directory specified by "path" at vfs layer.
// note that only the last level directory of the path will be created,
// and its parent directory must exist.
//
int vfs_mkdir(const char *path) {
  struct dentry *parent = vfs_root_dentry;
  char miss_name[MAX_PATH_LEN];

  // lookup the dir, find its parent direntry
  struct dentry *file_dentry = lookup_final_dentry(path, &parent, miss_name);
  if (file_dentry) {
    sprint("vfs_mkdir: the directory already exists!\n");
    return -1;
  }

  char basename[MAX_PATH_LEN];
  get_base_name(path, basename);
  if (strcmp(miss_name, basename) != 0) {
    sprint("vfs_mkdir: cannot create directory in a non-exist directory!\n");
    return -1;
  }

  // do real mkdir
  struct dentry *new_dentry = alloc_vfs_dentry(basename, NULL, parent);
  struct vinode *new_dir_inode = viop_mkdir(parent->dentry_inode, new_dentry);
  if (!new_dir_inode) {
    free_page(new_dentry);
    sprint("vfs_mkdir: cannot create directory!\n");
    return -1;
  }

  new_dentry->dentry_inode = new_dir_inode;
  new_dir_inode->ref++;
  hash_put_dentry(new_dentry);
  hash_put_vinode(new_dir_inode);
  return 0;
}

//
// close a directory at vfs layer
//
int vfs_closedir(struct file *file) {
  if (file->f_dentry->dentry_inode->type != DIR_I) {
    sprint("vfs_closedir: cannot close a file!\n");
    return -1;
  }

  // even if a directory is no longer referenced, it will not be freed because
  // it will serve as a cache for later lookup operations on it or its
  // descendants
  file->f_dentry->d_ref--;
  file->status = FD_NONE;

  // additional close direntry operations for a specific file system
  // rfs needs reclaim dir cache.
  if (file->f_dentry->dentry_inode->i_ops->viop_hook_closedir) {
    if (file->f_dentry->dentry_inode->i_ops->
        viop_hook_closedir(file->f_dentry->dentry_inode, file->f_dentry) != 0) {
      sprint("vfs_closedir: hook closedir failed!\n");
    }
  }
  return 0;
}

//
// lookup the "path" and return its dentry (or NULL if not found).
// the lookup starts from parent, and stop till the full "path" is parsed.
// return: the final dentry if we find it, NULL for otherwise.
//
struct dentry *lookup_final_dentry(const char *path, struct dentry **parent,
                                   char *miss_name) {
  char path_copy[MAX_PATH_LEN];
  strcpy(path_copy, path);

  // split the path, and retrieves a token at a time.
  // note: strtok() uses a static (local) variable to store the input path
  // string at the first time it is called. thus it can out a token each time.
  // for example, when input path is: /RAMDISK0/test_dir/ramfile2
  // strtok() outputs three tokens: 1)RAMDISK0, 2)test_dir and 3)ramfile2
  // at its three continuous invocations.
  char *token = strtok(path_copy, "/");
  struct dentry *this = *parent;

  while (token != NULL) {
    *parent = this;
    this = hash_get_dentry((*parent), token);  // try hash first
    if (this == NULL) {
      // if not found in hash, try to find it in the directory
      this = alloc_vfs_dentry(token, NULL, *parent);
      // lookup subfolder/file in its parent directory. note:
      // hostfs and rfs will take different procedures for lookup.
      struct vinode *found_vinode = viop_lookup((*parent)->dentry_inode, this);
      if (found_vinode == NULL) {
        // not found in both hash table and directory file on disk.
        free_page(this);
        strcpy(miss_name, token);
        return NULL;
      }

      struct vinode *same_inode = hash_get_vinode(found_vinode->sb, found_vinode->inum);
      if (same_inode != NULL) {
        // the vinode is already in the hash table (i.e. we are opening another hard link)
        this->dentry_inode = same_inode;
        same_inode->ref++;
        free_page(found_vinode);
      } else {
        // the vinode is not in the hash table
        this->dentry_inode = found_vinode;
        found_vinode->ref++;
        hash_put_vinode(found_vinode);
      }

      hash_put_dentry(this);
    }

    // get next token
    token = strtok(NULL, "/");
  }
  return this;
}

//
// get the base name of a path
//
void get_base_name(const char *path, char *base_name) {
  char path_copy[MAX_PATH_LEN];
  strcpy(path_copy, path);

  char *token = strtok(path_copy, "/");
  char *last_token = NULL;
  while (token != NULL) {
    last_token = token;
    token = strtok(NULL, "/");
  }

  strcpy(base_name, last_token);
}

//
// alloc a (virtual) file
//
struct file *alloc_vfs_file(struct dentry *file_dentry, int readable, int writable,
                        int offset) {
  struct file *file = alloc_page();
  file->f_dentry = file_dentry;
  file_dentry->d_ref += 1;

  file->readable = readable;
  file->writable = writable;
  file->offset = 0;
  file->status = FD_OPENED;
  return file;
}

//
// alloc a (virtual) dir entry
//
struct dentry *alloc_vfs_dentry(const char *name, struct vinode *inode,
                            struct dentry *parent) {
  struct dentry *dentry = (struct dentry *)alloc_page();
  strcpy(dentry->name, name);
  dentry->dentry_inode = inode;
  if (inode) inode->ref++;

  dentry->parent = parent;
  dentry->d_ref = 0;
  return dentry;
}

//
// free a (virtual) dir entry, if it is not referenced by any file
//
int free_vfs_dentry(struct dentry *dentry) {
  if (dentry->d_ref > 0) {
    sprint("free_vfs_dentry: dentry is still in use!\n");
    return -1;
  }
  free_page((void *)dentry);
  return 0;
}

// dentry generic hash table method implementation
int dentry_hash_equal(void *key1, void *key2) {
  struct dentry_key *dentry_key1 = key1;
  struct dentry_key *dentry_key2 = key2;
  if (strcmp(dentry_key1->name, dentry_key2->name) == 0 &&
      dentry_key1->parent == dentry_key2->parent) {
    return 1;
  }
  return 0;
}

size_t dentry_hash_func(void *key) {
  struct dentry_key *dentry_key = key;
  char *name = dentry_key->name;

  size_t hash = 5381;
  int c;

  while ((c = *name++)) hash = ((hash << 5) + hash) + c;  // hash * 33 + c

  hash = ((hash << 5) + hash) + (size_t)dentry_key->parent;
  return hash % HASH_TABLE_SIZE;
}

// dentry hash table interface
struct dentry *hash_get_dentry(struct dentry *parent, char *name) {
  struct dentry_key key = {.parent = parent, .name = name};
  return (struct dentry *)dentry_hash_table.virtual_hash_get(&dentry_hash_table,
                                                             &key);
}

int hash_put_dentry(struct dentry *dentry) {
  struct dentry_key *key = alloc_page();
  key->name = dentry->name;
  key->parent = dentry->parent;

  int ret = dentry_hash_table.virtual_hash_put(&dentry_hash_table, key, dentry);
  if (ret != 0)
    free_page(key);
  return ret;
}

int hash_erase_dentry(struct dentry *dentry) {
  struct dentry_key key = {.parent = dentry->parent, .name = dentry->name};
  return dentry_hash_table.virtual_hash_erase(&dentry_hash_table, &key);
}

// vinode generic hash table method implementation
int vinode_hash_equal(void *key1, void *key2) {
  struct vinode_key *vinode_key1 = key1;
  struct vinode_key *vinode_key2 = key2;
  if (vinode_key1->inum == vinode_key2->inum && vinode_key1->sb == vinode_key2->sb) {
    return 1;
  }
  return 0;
}

size_t vinode_hash_func(void *key) {
  struct vinode_key *vinode_key = key;
  return vinode_key->inum % HASH_TABLE_SIZE;
}

// vinode hash table interface
struct vinode *hash_get_vinode(struct super_block *sb, int inum) {
  if (inum < 0) return NULL;
  struct vinode_key key = {.sb = sb, .inum = inum};
  return (struct vinode *)vinode_hash_table.virtual_hash_get(&vinode_hash_table,
                                                             &key);
}

int hash_put_vinode(struct vinode *vinode) {
  if (vinode->inum < 0) return -1;
  struct vinode_key *key = alloc_page();
  key->sb = vinode->sb;
  key->inum = vinode->inum;

  int ret = vinode_hash_table.virtual_hash_put(&vinode_hash_table, key, vinode);
  if (ret != 0) free_page(key);
  return ret;
}

int hash_erase_vinode(struct vinode *vinode) {
  if (vinode->inum < 0) return -1;
  struct vinode_key key = {.sb = vinode->sb, .inum = vinode->inum};
  return vinode_hash_table.virtual_hash_erase(&vinode_hash_table, &key);
}

//
// shared (default) actions on allocating a vfs inode.
//
struct vinode *default_alloc_vinode(struct super_block *sb) {
  struct vinode *vinode = (struct vinode *)alloc_page();
  vinode->blocks = 0;
  vinode->inum = 0;
  vinode->nlinks = 0;
  vinode->ref = 0;
  vinode->sb = sb;
  vinode->size = 0;
  return vinode;
}

struct file_system_type *fs_list[MAX_SUPPORTED_FS];
```

- vfs.h
```c
#ifndef _VFS_H_
#define _VFS_H_

#include "util/types.h"

#define MAX_VFS_DEV 10            // the maximum number of vfs_dev_list
#define MAX_DENTRY_NAME_LEN 30    // the maximum length of dentry name
#define MAX_DEVICE_NAME_LEN 30    // the maximum length of device name
#define MAX_MOUNTS 10             // the maximum number of mounts
#define MAX_DENTRY_HASH_SIZE 100  // the maximum size of dentry hash table
#define MAX_PATH_LEN 30           // the maximum length of path
#define MAX_SUPPORTED_FS 10       // the maximum number of supported file systems

#define DIRECT_BLKNUM 10          // the number of direct blocks

/**** vfs initialization function ****/
int vfs_init();

/**** vfs interfaces ****/

// device interfaces
struct super_block *vfs_mount(const char *dev_name, int mnt_type);

// file interfaces
struct file *vfs_open(const char *path, int flags);
ssize_t vfs_read(struct file *file, char *buf, size_t count);
ssize_t vfs_write(struct file *file, const char *buf, size_t count);
ssize_t vfs_lseek(struct file *file, ssize_t offset, int whence);
int vfs_stat(struct file *file, struct istat *istat);
int vfs_disk_stat(struct file *file, struct istat *istat);
int vfs_link(const char *oldpath, const char *newpath);
int vfs_unlink(const char *path);
int vfs_close(struct file *file);

// directory interfaces
struct file *vfs_opendir(const char *path);
int vfs_readdir(struct file *file, struct dir *dir);
int vfs_mkdir(const char *path);
int vfs_closedir(struct file *file);

/**** vfs abstract object types ****/
// system root direntry
extern struct dentry *vfs_root_dentry;

// vfs abstract dentry
struct dentry {
  char name[MAX_DENTRY_NAME_LEN];
  int d_ref;
  struct vinode *dentry_inode;
  struct dentry *parent;
  struct super_block *sb;
};


// dentry constructor and destructor
struct dentry *alloc_vfs_dentry(const char *name, struct vinode *inode,
                            struct dentry *parent);
int free_vfs_dentry(struct dentry *dentry);

// ** dentry hash table **
extern struct hash_table dentry_hash_table;

// dentry hash table key type
struct dentry_key {
  struct dentry *parent;
  char *name;
};

// generic hash table method implementation
int dentry_hash_equal(void *key1, void *key2);
size_t dentry_hash_func(void *key);

// dentry hash table interface
struct dentry *hash_get_dentry(struct dentry *parent, char *name);
int hash_put_dentry(struct dentry *dentry);
int hash_erase_dentry(struct dentry *dentry);

// data structure of an openned file
struct file {
  int status;
  int readable;
  int writable;
  int offset;
  struct dentry *f_dentry;
};

// file constructor and destructor(use free_page to destruct)
struct file *alloc_vfs_file(struct dentry *dentry, int readable, int writable,
                        int offset);

// abstract device entry in vfs_dev_list
struct device {
  char dev_name[MAX_DEVICE_NAME_LEN];  // the name of the device
  int dev_id;  // the id of the device (the meaning of an id is interpreted by
               // the specific file system, all we need to know is that it is
               // a unique identifier)
  struct file_system_type *fs_type;  // the file system type in the device
};

// device list in vfs layer
extern struct device *vfs_dev_list[MAX_VFS_DEV];

// supported file system types
struct file_system_type {
  int type_num;  // the number of the file system type
  struct super_block *(*get_superblock)(struct device *dev);
};

extern struct file_system_type *fs_list[MAX_SUPPORTED_FS];

// general-purpose super_block structure
struct super_block {
  int magic;              // magic number of the file system
  int size;               // size of file system image (blocks)
  int nblocks;            // number of data blocks
  int ninodes;            // number of inodes.
  struct dentry *s_root;  // root dentry of inode
  struct device *s_dev;   // device of the superblock
  void *s_fs_info;        // filesystem-specific info. for rfs, it points bitmap
};

// abstract vfs inode
struct vinode {
  int inum;                  // inode number of the disk inode
  int ref;                   // reference count
  int size;                  // size of the file (in bytes)
  int type;                  // one of FILE_I, DIR_I
  int nlinks;                // number of hard links to this file
  int blocks;                // number of blocks
  int addrs[DIRECT_BLKNUM];  // direct blocks
  void *i_fs_info;           // filesystem-specific info (see s_fs_info)
  struct super_block *sb;          // super block of the vfs inode
  const struct vinode_ops *i_ops;  // vfs inode operations
};

struct vinode_ops {
  // file operations
  ssize_t (*viop_read)(struct vinode *node, char *buf, ssize_t len,
                       int *offset);
  ssize_t (*viop_write)(struct vinode *node, const char *buf, ssize_t len,
                        int *offset);
  struct vinode *(*viop_create)(struct vinode *parent, struct dentry *sub_dentry);
  int (*viop_lseek)(struct vinode *node, ssize_t new_off, int whence, int *off);
  int (*viop_disk_stat)(struct vinode *node, struct istat *istat);
  int (*viop_link)(struct vinode *parent, struct dentry *sub_dentry,
                   struct vinode *link_node);
  int (*viop_unlink)(struct vinode *parent, struct dentry *sub_dentry,
                     struct vinode *unlink_node);
  struct vinode *(*viop_lookup)(struct vinode *parent,
                                struct dentry *sub_dentry);

  // directory operations
  int (*viop_readdir)(struct vinode *dir_vinode, struct dir *dir, int *offset);
  struct vinode *(*viop_mkdir)(struct vinode *parent, struct dentry *sub_dentry);

  // write back inode to disk
  int (*viop_write_back_vinode)(struct vinode *node);

  // hook functions
  // In the vfs layer, we do not assume that hook functions will do anything,
  // but simply call them (when they are defined) at the appropriate time.
  // Hook functions exist because the fs layer may need to do some additional
  // operations (such as allocating additional data structures) at some critical
  // times.
  int (*viop_hook_open)(struct vinode *node, struct dentry *dentry);
  int (*viop_hook_close)(struct vinode *node, struct dentry *dentry);
  int (*viop_hook_opendir)(struct vinode *node, struct dentry *dentry);
  int (*viop_hook_closedir)(struct vinode *node, struct dentry *dentry);
};

// vinode operation interface
// the implementation depends on the vinode type and the specific file system

// virtual file system inode interfaces
#define viop_read(node, buf, len, offset)      (node->i_ops->viop_read(node, buf, len, offset))
#define viop_write(node, buf, len, offset)     (node->i_ops->viop_write(node, buf, len, offset))
#define viop_create(node, name)                (node->i_ops->viop_create(node, name))
#define viop_lseek(node, new_off, whence, off) (node->i_ops->viop_lseek(node, new_off, whence, off))
#define viop_disk_stat(node, istat)            (node->i_ops->viop_disk_stat(node, istat))
#define viop_link(node, name, link_node)       (node->i_ops->viop_link(node, name, link_node))
#define viop_unlink(node, name, unlink_node)   (node->i_ops->viop_unlink(node, name, unlink_node))
#define viop_lookup(parent, sub_dentry)        (parent->i_ops->viop_lookup(parent, sub_dentry))
#define viop_readdir(dir_vinode, dir, offset)  (dir_vinode->i_ops->viop_readdir(dir_vinode, dir, offset))
#define viop_mkdir(dir, sub_dentry)            (dir->i_ops->viop_mkdir(dir, sub_dentry))
#define viop_write_back_vinode(node)           (node->i_ops->viop_write_back_vinode(node))

// vinode hash table
extern struct hash_table vinode_hash_table;

// vinode hash table key type
struct vinode_key {
  int inum;
  struct super_block *sb;
};

// generic hash table method implementation
int vinode_hash_equal(void *key1, void *key2);
size_t vinode_hash_func(void *key);

// vinode hash table interface
struct vinode *hash_get_vinode(struct super_block *sb, int inum);
int hash_put_vinode(struct vinode *vinode);
int hash_erase_vinode(struct vinode *vinode);

// other utility functions
struct vinode *default_alloc_vinode(struct super_block *sb);
struct dentry *lookup_final_dentry(const char *path, struct dentry **parent,
                                   char *miss_name);
void get_base_name(const char *path, char *base_name);

#endif
```

# 通关操作

- 用以下代码替换 `strap.c`

```c
#include "riscv.h"
#include "process.h"
#include "strap.h"
#include "syscall.h"
#include "pmm.h"
#include "vmm.h"
#include "sched.h"
#include "util/functions.h"

#include "spike_interface/spike_utils.h"

static void handle_syscall(trapframe *tf) {
  tf->epc += 4;
  tf->regs.a0 = do_syscall(tf->regs.a0, tf->regs.a1, tf->regs.a2, tf->regs.a3,
                           tf->regs.a4, tf->regs.a5, tf->regs.a6, tf->regs.a7);
}

static uint64 g_ticks = 0;

void handle_mtimer_trap() {
  g_ticks++;
  write_csr(sip, read_csr(sip) & ~SIP_SSIP);
}

void handle_user_page_fault(uint64 mcause, uint64 sepc, uint64 stval) {
  if (mcause != CAUSE_STORE_PAGE_FAULT && mcause != CAUSE_LOAD_PAGE_FAULT) {
    panic("unknown page fault.\n");
  }

  void* pa = alloc_page();
  if (pa == 0) panic("alloc_page failed\n");
  user_vm_map((pagetable_t)current->pagetable, ROUNDDOWN(stval, PGSIZE), PGSIZE, (uint64)pa,
         prot_to_type(PROT_WRITE | PROT_READ, 1));
}

void rrsched() {
  current->tick_count++;
  if (current->tick_count >= TIME_SLICE_LEN) {
    current->tick_count = 0;
    current->status = READY;
    insert_to_ready_queue(current);
    schedule();
  }
}

void smode_trap_handler(void) {
  if ((read_csr(sstatus) & SSTATUS_SPP) != 0) panic("usertrap: not from user mode");

  assert(current);
  current->trapframe->epc = read_csr(sepc);

  uint64 cause = read_csr(scause);

  switch (cause) {
    case CAUSE_USER_ECALL:
      handle_syscall(current->trapframe);
      break;
    case CAUSE_MTIMER_S_TRAP:
      handle_mtimer_trap();
      rrsched();
      break;
    case CAUSE_STORE_PAGE_FAULT:
    case CAUSE_LOAD_PAGE_FAULT:
      handle_user_page_fault(cause, read_csr(sepc), read_csr(stval));
      break;
    default:
      sprint("smode_trap_handler(): unexpected scause %p\n", read_csr(scause));
      sprint("            sepc=%p stval=%p\n", read_csr(sepc), read_csr(stval));
      panic( "unexpected exception happened.\n" );
      break;
  }

  switch_to(current);
}
```

- 用以下代码替换 `mtrap.c`

```c
#include "kernel/riscv.h"
#include "kernel/process.h"
#include "spike_interface/spike_utils.h"

static void handle_instruction_access_fault() { panic("Instruction access fault!"); }

static void handle_load_access_fault() { panic("Load access fault!"); }

static void handle_store_access_fault() { panic("Store/AMO access fault!"); }

static void handle_illegal_instruction() {
  sprint("Illegal instruction!\n");
  panic("Illegal instruction!");
}

static void handle_misaligned_load() { panic("Misaligned Load!"); }

static void handle_misaligned_store() { panic("Misaligned AMO!"); }

static void handle_timer() {
  int cpuid = 0;
  *(uint64*)CLINT_MTIMECMP(cpuid) = *(uint64*)CLINT_MTIMECMP(cpuid) + TIMER_INTERVAL;
  write_csr(sip, SIP_SSIP);
}

void handle_mtrap() {
  uint64 mcause = read_csr(mcause);
  switch (mcause) {
    case CAUSE_MTIMER:
      handle_timer();
      break;
    case CAUSE_FETCH_ACCESS:
      handle_instruction_access_fault();
      break;
    case CAUSE_LOAD_ACCESS:
      handle_load_access_fault();
      break;
    case CAUSE_STORE_ACCESS:
      handle_store_access_fault();
      break;
    case CAUSE_ILLEGAL_INSTRUCTION:
      handle_illegal_instruction();
      break;
    case CAUSE_MISALIGNED_LOAD:
      handle_misaligned_load();
      break;
    case CAUSE_MISALIGNED_STORE:
      handle_misaligned_store();
      break;

    default:
      sprint("machine trap(): unexpected mscause %p\n", mcause);
      sprint("            mepc=%p mtval=%p\n", read_csr(mepc), read_csr(mtval));
      panic( "unexpected exception happened in M-mode.\n" );
      break;
  }
}
```

- 用以下代码替换 `process.c`

```c
#include "riscv.h"
#include "strap.h"
#include "config.h"
#include "process.h"
#include "elf.h"
#include "string.h"
#include "vmm.h"
#include "pmm.h"
#include "memlayout.h"
#include "sched.h"
#include "spike_interface/spike_utils.h"

extern char smode_trap_vector[];
extern void return_to_user(trapframe *, uint64 satp);
extern char trap_sec_start[];

process procs[NPROC];
process* current = NULL;

void switch_to(process* proc) {
  assert(proc);
  current = proc;

  write_csr(stvec, (uint64)smode_trap_vector);

  proc->trapframe->kernel_sp = proc->kstack;
  proc->trapframe->kernel_satp = read_csr(satp);
  proc->trapframe->kernel_trap = (uint64)smode_trap_handler;

  unsigned long x = read_csr(sstatus);
  x &= ~SSTATUS_SPP;
  x |= SSTATUS_SPIE;
  write_csr(sstatus, x);

  write_csr(sepc, proc->trapframe->epc);

  uint64 user_satp = MAKE_SATP(proc->pagetable);
  return_to_user(proc->trapframe, user_satp);
}

void init_proc_pool() {
  memset( procs, 0, sizeof(process)*NPROC );

  for (int i = 0; i < NPROC; ++i) {
    procs[i].status = FREE;
    procs[i].pid = i;
  }
}

process* alloc_process() {
  int i;
  for( i=0; i<NPROC; i++ )
    if( procs[i].status == FREE ) break;

  if( i>=NPROC ){
    panic( "cannot find any free process structure.\n" );
    return 0;
  }

  procs[i].trapframe = (trapframe *)alloc_page();
  memset(procs[i].trapframe, 0, sizeof(trapframe));

  procs[i].pagetable = (pagetable_t)alloc_page();
  memset((void *)procs[i].pagetable, 0, PGSIZE);

  procs[i].kstack = (uint64)alloc_page() + PGSIZE;
  uint64 user_stack = (uint64)alloc_page();
  procs[i].trapframe->regs.sp = USER_STACK_TOP;

  procs[i].mapped_info = (mapped_region*)alloc_page();
  memset( procs[i].mapped_info, 0, PGSIZE );

  user_vm_map((pagetable_t)procs[i].pagetable, USER_STACK_TOP - PGSIZE, PGSIZE,
    user_stack, prot_to_type(PROT_WRITE | PROT_READ, 1));
  procs[i].mapped_info[STACK_SEGMENT].va = USER_STACK_TOP - PGSIZE;
  procs[i].mapped_info[STACK_SEGMENT].npages = 1;
  procs[i].mapped_info[STACK_SEGMENT].seg_type = STACK_SEGMENT;

  user_vm_map((pagetable_t)procs[i].pagetable, (uint64)procs[i].trapframe, PGSIZE,
    (uint64)procs[i].trapframe, prot_to_type(PROT_WRITE | PROT_READ, 0));
  procs[i].mapped_info[CONTEXT_SEGMENT].va = (uint64)procs[i].trapframe;
  procs[i].mapped_info[CONTEXT_SEGMENT].npages = 1;
  procs[i].mapped_info[CONTEXT_SEGMENT].seg_type = CONTEXT_SEGMENT;

  user_vm_map((pagetable_t)procs[i].pagetable, (uint64)trap_sec_start, PGSIZE,
    (uint64)trap_sec_start, prot_to_type(PROT_READ | PROT_EXEC, 0));
  procs[i].mapped_info[SYSTEM_SEGMENT].va = (uint64)trap_sec_start;
  procs[i].mapped_info[SYSTEM_SEGMENT].npages = 1;
  procs[i].mapped_info[SYSTEM_SEGMENT].seg_type = SYSTEM_SEGMENT;

  sprint("in alloc_proc. user frame 0x%lx, user stack 0x%lx, user kstack 0x%lx \n",
    procs[i].trapframe, procs[i].trapframe->regs.sp, procs[i].kstack);

  procs[i].user_heap.heap_top = USER_FREE_ADDRESS_START;
  procs[i].user_heap.heap_bottom = USER_FREE_ADDRESS_START;
  procs[i].user_heap.free_pages_count = 0;

  procs[i].total_mapped_region = 3;
  procs[i].tick_count = 0;
  procs[i].status = READY;

  procs[i].pfiles = init_proc_file_management();
  sprint("in alloc_proc. build proc_file_management successfully.\n");

  return &procs[i];
}

int free_process( process* proc ) {
  proc->status = ZOMBIE;
  return 0;
}

int do_fork( process* parent)
{
  sprint( "will fork a child from parent %d.\n", parent->pid );
  process* child = alloc_process();

  *child->trapframe = *parent->trapframe;

  memcpy((void*)lookup_pa(child->pagetable, child->mapped_info[STACK_SEGMENT].va),
         (void*)lookup_pa(parent->pagetable, parent->mapped_info[STACK_SEGMENT].va), PGSIZE);

  for( int i=3; i<parent->total_mapped_region; i++ ){
    mapped_region* reg = &parent->mapped_info[i];
    if (reg->seg_type == CODE_SEGMENT) {
      for (int j = 0; j < reg->npages; j++) {
        uint64 va = reg->va + j * PGSIZE;
        uint64 pa = lookup_pa(parent->pagetable, va);
        user_vm_map(child->pagetable, va, PGSIZE, pa, prot_to_type(PROT_READ | PROT_EXEC, 1));
      }
      sprint("do_fork map code segment at pa:%016lx of parent to child at va:%016lx.\n",
             lookup_pa(parent->pagetable, reg->va), reg->va);
    } else if (reg->seg_type == DATA_SEGMENT || reg->seg_type == HEAP_SEGMENT) {
      for (int j = 0; j < reg->npages; j++) {
        uint64 va = reg->va + j * PGSIZE;
        void* pa = alloc_page();
        memcpy(pa, (void*)lookup_pa(parent->pagetable, va), PGSIZE);
        user_vm_map(child->pagetable, va, PGSIZE, (uint64)pa, prot_to_type(PROT_WRITE | PROT_READ, 1));
      }
    }
    child->mapped_info[i] = *reg;
    child->total_mapped_region++;
  }

  child->status = READY;
  child->trapframe->regs.a0 = 0;
  child->parent = parent;
  insert_to_ready_queue( child );

  return child->pid;
}
```

- 用以下代码替换 `vmm.c`

```c
#include "vmm.h"
#include "riscv.h"
#include "pmm.h"
#include "memlayout.h"
#include "util/string.h"
#include "util/functions.h"
#include "spike_interface/spike_utils.h"

pagetable_t g_kernel_pagetable;

uint64 prot_to_type(int prot, int user) {
  uint64 pte_flags = PTE_V;
  if (prot & PROT_READ) pte_flags |= PTE_R;
  if (prot & PROT_WRITE) pte_flags |= PTE_W;
  if (prot & PROT_EXEC) pte_flags |= PTE_X;
  if (user) pte_flags |= PTE_U;
  return pte_flags;
}

void kern_vm_init() {
  g_kernel_pagetable = (pagetable_t)alloc_page();
  memset((void *)g_kernel_pagetable, 0, PGSIZE);

  extern char _etext[];
  // Map kernel text section [KERN_BASE, _etext] with R-X permissions
  uint64 text_start = (uint64)KERN_BASE;
  uint64 text_end = ROUNDUP((uint64)_etext, PGSIZE);
  // Limit mapping to avoid timeout - only map up to 2MB of kernel code
  if (text_end - text_start > (2 << 20)) {
    text_end = text_start + (2 << 20);
  }
  for (uint64 va = text_start; va < text_end; va += PGSIZE) {
    pte_t* pte = page_walk(g_kernel_pagetable, va, 1);
    if (pte) *pte = PA2PTE(va) | PTE_V | PTE_R | PTE_X;
  }
}

void* user_va_to_pa(pagetable_t page_dir, void* va) {
  uint64 off = (uint64)va - ROUNDDOWN((uint64)va, PGSIZE);
  return (void*)(lookup_pa(page_dir, (uint64)va) + off);
}

void user_vm_map(pagetable_t page_dir, uint64 va, uint64 size, uint64 pa , int perm) {
  for (uint64 i = 0; i < size; i += PGSIZE) {
    pte_t* pte = page_walk(page_dir, va + i, 1);
    assert(pte);
    assert(!(*pte & PTE_V));
    *pte = PA2PTE(pa + i) | perm | PTE_V;
  }
}

void user_vm_unmap(pagetable_t page_dir, uint64 va, uint64 size, int free_pa) {
  for (uint64 i = 0; i < size; i += PGSIZE) {
    pte_t* pte = page_walk(page_dir, va + i, 0);
    if (pte && (*pte & PTE_V)) {
      if (free_pa) free_page((void*)PTE2PA(*pte));
      *pte = 0;
    }
  }
}

pte_t* page_walk(pagetable_t page_dir, uint64 va, int alloc) {
  if (va >= MAXVA) panic("page_walk: va out of range!\n");

  for (int level = 2; level > 0; level--) {
    pte_t* pte = &page_dir[PX(level, va)];
    if (*pte & PTE_V) {
      page_dir = (pagetable_t)PTE2PA(*pte);
    } else {
      if (!alloc || (page_dir = (pagetable_t)alloc_page()) == 0) return 0;
      memset(page_dir, 0, PGSIZE);
      *pte = PA2PTE(page_dir) | PTE_V;
    }
  }
  return &page_dir[PX(0, va)];
}

uint64 lookup_pa(pagetable_t page_dir, uint64 va) {
  pte_t* pte = page_walk(page_dir, va, 0);
  if (pte == 0 || !(*pte & PTE_V) || !(*pte & PTE_U)) return 0;
  return PTE2PA(*pte);
}
```

- 用以下代码替换 `vfs.c`

```c
#include "vfs.h"
#include "pmm.h"
#include "spike_interface/spike_utils.h"
#include "util/string.h"
#include "util/types.h"
#include "util/hash_table.h"
#include "process.h"

#ifndef MAX_FS_TYPE
#define MAX_FS_TYPE 10
#endif

struct dentry *vfs_root_dentry;
struct super_block *vfs_sb_list[MAX_MOUNTS];
struct device *vfs_dev_list[MAX_VFS_DEV];
struct file_system_type *fs_list[MAX_FS_TYPE];
struct hash_table dentry_hash_table;
struct hash_table vinode_hash_table;

int vfs_init() {
  int ret;
  ret = hash_table_init(&dentry_hash_table, dentry_hash_equal, dentry_hash_func,
                            NULL, NULL, NULL);
  if (ret != 0) return ret;

  ret = hash_table_init(&vinode_hash_table, vinode_hash_equal, vinode_hash_func,
                            NULL, NULL, NULL);
  if (ret != 0) return ret;
  return 0; 
}

struct super_block *vfs_mount(const char *dev_name, int mnt_type) {
  struct device *p_device = NULL;
  for (int i = 0; i < MAX_VFS_DEV; ++i) {
    p_device = vfs_dev_list[i];
    if (p_device && strcmp(p_device->dev_name, dev_name) == 0) break;
  }
  if (p_device == NULL) panic("vfs_mount: cannot find the device entry!\n");

  struct file_system_type *fs_type = p_device->fs_type;
  struct super_block *sb = fs_type->get_superblock(p_device);
  hash_put_vinode(sb->s_root->dentry_inode);

  int err = 1;
  for (int i = 0; i < MAX_MOUNTS; ++i) {
    if (vfs_sb_list[i] == NULL) {
      vfs_sb_list[i] = sb;
      err = 0;
      break;
    }
  }
  if (err) panic("vfs_mount: too many mounts!\n");

  if (mnt_type == MOUNT_AS_ROOT) {
    vfs_root_dentry = sb->s_root;
    hash_put_dentry(sb->s_root);
  } else if (mnt_type == MOUNT_DEFAULT) {
    if (!vfs_root_dentry)
      panic("vfs_mount: root dentry not found, please mount the root device first!\n");

    struct dentry *mnt_point = sb->s_root;
    char *dev_name = p_device->dev_name;
    strcpy(mnt_point->name, dev_name);
    mnt_point->parent = vfs_root_dentry;
    hash_put_dentry(sb->s_root);
  } else {
    panic("vfs_mount: unknown mount type!\n");
  }

  return sb;
}

struct dentry *lookup_final_dentry(const char *path, struct dentry **parent,
                                   char *miss_name) {
  char path_copy[MAX_PATH_LEN];
  strcpy(path_copy, path);

  struct dentry *this;
  if (path[0] == '/') {
    this = vfs_root_dentry;
  } else {
    this = current->pfiles->cwd;
  }

  char *token = strtok(path_copy, "/");
  while (token != NULL) {
    if (strcmp(token, ".") == 0) {
      // current directory, do nothing
    } else if (strcmp(token, "..") == 0) {
      // parent directory
      if (this->parent != NULL) {
        this = this->parent;
      }
    } else {
      *parent = this;
      struct dentry *next = hash_get_dentry((*parent), token);
      if (next == NULL) {
        next = alloc_vfs_dentry(token, NULL, *parent);
        struct vinode *found_vinode = viop_lookup((*parent)->dentry_inode, next);
        if (found_vinode == NULL) {
          free_page(next);
          strcpy(miss_name, token);
          return NULL;
        }

        struct vinode *same_inode = hash_get_vinode(found_vinode->sb, found_vinode->inum);
        if (same_inode != NULL) {
          next->dentry_inode = same_inode;
          same_inode->ref++;
          free_page(found_vinode);
        } else {
          next->dentry_inode = found_vinode;
          found_vinode->ref++;
          hash_put_vinode(found_vinode);
        }
        hash_put_dentry(next);
      }
      this = next;
    }
    token = strtok(NULL, "/");
  }
  return this;
}

struct file *vfs_open(const char *path, int flags) {
  struct dentry *parent = NULL;
  char miss_name[MAX_PATH_LEN];
  struct dentry *file_dentry = lookup_final_dentry(path, &parent, miss_name);

  if (!file_dentry) {
    int creatable = flags & O_CREAT;
    if (creatable) {
      char basename[MAX_PATH_LEN];
      get_base_name(path, basename);
      if (strcmp(miss_name, basename) != 0) {
        sprint("vfs_open: cannot create file in a non-exist directory!\n");
        return NULL;
      }
      file_dentry = alloc_vfs_dentry(basename, NULL, parent);
      struct vinode *new_inode = viop_create(parent->dentry_inode, file_dentry);
      if (!new_inode) panic("vfs_open: cannot create file!\n");
      file_dentry->dentry_inode = new_inode;
      new_inode->ref++;
      hash_put_dentry(file_dentry);
      hash_put_vinode(new_inode); 
    } else {
      sprint("vfs_open: cannot find the file!\n");
      return NULL;
    }
  }

  if (file_dentry->dentry_inode->type != FILE_I) {
    sprint("vfs_open: cannot open a directory!\n");
    return NULL;
  }

  int writable = 0, readable = 0;
  switch (flags & MASK_FILEMODE) {
    case O_RDONLY: writable = 0; readable = 1; break;
    case O_WRONLY: writable = 1; readable = 0; break;
    case O_RDWR:   writable = 1; readable = 1; break;
    default:       panic("fs_open: invalid open flags!\n");
  }

  struct file *file = alloc_vfs_file(file_dentry, readable, writable, 0);
  if (file_dentry->dentry_inode->i_ops->viop_hook_open) {
    if (file_dentry->dentry_inode->i_ops->viop_hook_open(file_dentry->dentry_inode, file_dentry) < 0) {
      sprint("vfs_open: hook_open failed!\n");
    }
  }
  return file;
}

ssize_t vfs_read(struct file *file, char *buf, size_t count) {
  if (!file->readable) return -1;
  if (file->f_dentry->dentry_inode->type != FILE_I) return -1;
  return viop_read(file->f_dentry->dentry_inode, buf, count, &(file->offset));
}

ssize_t vfs_write(struct file *file, const char *buf, size_t count) {
  if (!file->writable) return -1;
  if (file->f_dentry->dentry_inode->type != FILE_I) return -1;
  return viop_write(file->f_dentry->dentry_inode, buf, count, &(file->offset));
}

ssize_t vfs_lseek(struct file *file, ssize_t offset, int whence) {
  if (file->f_dentry->dentry_inode->type != FILE_I) return -1;
  if (viop_lseek(file->f_dentry->dentry_inode, offset, whence, &(file->offset)) != 0) return -1;
  return file->offset;
}

int vfs_stat(struct file *file, struct istat *istat) {
  istat->st_inum = file->f_dentry->dentry_inode->inum;
  istat->st_size = file->f_dentry->dentry_inode->size;
  istat->st_type = file->f_dentry->dentry_inode->type;
  istat->st_nlinks = file->f_dentry->dentry_inode->nlinks;
  istat->st_blocks = file->f_dentry->dentry_inode->blocks;
  return 0;
}

int vfs_disk_stat(struct file *file, struct istat *istat) {
  return viop_disk_stat(file->f_dentry->dentry_inode, istat);
}

int vfs_link(const char *oldpath, const char *newpath) {
  struct dentry *parent = NULL;
  char miss_name[MAX_PATH_LEN];
  struct dentry *old_file_dentry = lookup_final_dentry(oldpath, &parent, miss_name);
  if (!old_file_dentry || old_file_dentry->dentry_inode->type != FILE_I) return -1;

  parent = NULL;
  struct dentry *new_file_dentry = lookup_final_dentry(newpath, &parent, miss_name);
  if (new_file_dentry) return -1;

  char basename[MAX_PATH_LEN];
  get_base_name(newpath, basename);
  if (strcmp(miss_name, basename) != 0) return -1;

  new_file_dentry = alloc_vfs_dentry(basename, old_file_dentry->dentry_inode, parent);
  int err = viop_link(parent->dentry_inode, new_file_dentry, old_file_dentry->dentry_inode);
  if (err) return -1;
  hash_put_dentry(new_file_dentry);
  return 0;
}

int vfs_unlink(const char *path) {
  struct dentry *parent = NULL;
  char miss_name[MAX_PATH_LEN];
  struct dentry *file_dentry = lookup_final_dentry(path, &parent, miss_name);
  if (!file_dentry || file_dentry->dentry_inode->type != FILE_I || file_dentry->d_ref > 0) return -1;

  struct vinode *unlinked_vinode = file_dentry->dentry_inode;
  int err = viop_unlink(parent->dentry_inode, file_dentry, unlinked_vinode);
  if (err) return -1;

  hash_erase_dentry(file_dentry);
  free_vfs_dentry(file_dentry);
  unlinked_vinode->ref--; 
  if (unlinked_vinode->nlinks == 0) {
    assert(unlinked_vinode->ref == 0);
    hash_erase_vinode(unlinked_vinode);
    free_page(unlinked_vinode);
  }
  return 0;
}

int vfs_close(struct file *file) {
  if (file->f_dentry->dentry_inode->type != FILE_I) return -1;
  struct dentry *dentry = file->f_dentry;
  struct vinode *inode = dentry->dentry_inode;
  if (inode->i_ops->viop_hook_close) inode->i_ops->viop_hook_close(inode, dentry);
  dentry->d_ref--;
  if (dentry->d_ref == 0) {
    hash_erase_dentry(dentry);
    free_vfs_dentry(dentry);
    inode->ref--;
    if (inode->ref == 0) {
      if (viop_write_back_vinode(inode) != 0) panic("vfs_close: free inode failed!\n");
      hash_erase_vinode(inode);
      free_page(inode);
    }
  }
  file->status = FD_NONE;
  return 0;
}

struct file *vfs_opendir(const char *path) {
  struct dentry *parent = NULL;
  char miss_name[MAX_PATH_LEN];
  struct dentry *file_dentry = lookup_final_dentry(path, &parent, miss_name);
  if (!file_dentry || file_dentry->dentry_inode->type != DIR_I) return NULL;
  struct file *file = alloc_vfs_file(file_dentry, 1, 0, 0);
  if (file_dentry->dentry_inode->i_ops->viop_hook_opendir) {
    if (file_dentry->dentry_inode->i_ops->viop_hook_opendir(file_dentry->dentry_inode, file_dentry) != 0) return NULL;
  }
  return file;
}

int vfs_readdir(struct file *file, struct dir *dir) {
  return viop_readdir(file->f_dentry->dentry_inode, dir, &(file->offset));
}

int vfs_mkdir(const char *path) {
  struct dentry *parent = NULL;
  char miss_name[MAX_PATH_LEN];
  struct dentry *file_dentry = lookup_final_dentry(path, &parent, miss_name);
  if (file_dentry) return -1;
  char basename[MAX_PATH_LEN];
  get_base_name(path, basename);
  if (strcmp(miss_name, basename) != 0) return -1;
  struct dentry *new_dentry = alloc_vfs_dentry(basename, NULL, parent);
  struct vinode *new_inode = viop_mkdir(parent->dentry_inode, new_dentry);
  if (!new_inode) { free_page(new_dentry); return -1; }
  new_dentry->dentry_inode = new_inode;
  new_inode->ref++;
  hash_put_dentry(new_dentry);
  hash_put_vinode(new_inode);
  return 0;
}

int vfs_closedir(struct file *file) {
  if (file->f_dentry->dentry_inode->type != DIR_I) return -1;
  if (file->f_dentry->dentry_inode->i_ops->viop_hook_closedir)
    file->f_dentry->dentry_inode->i_ops->viop_hook_closedir(file->f_dentry->dentry_inode, file->f_dentry);
  file->f_dentry->d_ref--;
  if (file->f_dentry->d_ref == 0) {
    hash_erase_dentry(file->f_dentry);
    struct vinode *inode = file->f_dentry->dentry_inode;
    free_vfs_dentry(file->f_dentry);
    inode->ref--;
    if (inode->ref == 0) {
      viop_write_back_vinode(inode);
      hash_erase_vinode(inode);
      free_page(inode);
    }
  }
  file->status = FD_NONE;
  return 0;
}

struct dentry *alloc_vfs_dentry(const char *name, struct vinode *inode, struct dentry *parent) {
  struct dentry *dentry = (struct dentry *)alloc_page();
  strcpy(dentry->name, name);
  dentry->dentry_inode = inode;
  if (inode) inode->ref++;
  dentry->parent = parent;
  dentry->d_ref = 0;
  return dentry;
}

int free_vfs_dentry(struct dentry *dentry) {
  if (dentry->d_ref > 0) return -1;
  free_page((void *)dentry);
  return 0;
}

int dentry_hash_equal(void *key1, void *key2) {
  struct dentry_key *k1 = key1, *k2 = key2;
  return (strcmp(k1->name, k2->name) == 0 && k1->parent == k2->parent);
}

size_t dentry_hash_func(void *key) {
  struct dentry_key *k = key;
  char *name = k->name;
  size_t hash = 5381;
  int c;
  while ((c = *name++)) hash = ((hash << 5) + hash) + c;
  hash = ((hash << 5) + hash) + (size_t)k->parent;
  return hash % HASH_TABLE_SIZE;
}

struct dentry *hash_get_dentry(struct dentry *parent, char *name) {
  struct dentry_key key = {.parent = parent, .name = name};
  return (struct dentry *)dentry_hash_table.virtual_hash_get(&dentry_hash_table, &key);
}

int hash_put_dentry(struct dentry *dentry) {
  struct dentry_key *key = alloc_page();
  key->name = dentry->name;
  key->parent = dentry->parent;
  return dentry_hash_table.virtual_hash_put(&dentry_hash_table, key, dentry);
}

int hash_erase_dentry(struct dentry *dentry) {
  struct dentry_key key = {.parent = dentry->parent, .name = dentry->name};
  return dentry_hash_table.virtual_hash_erase(&dentry_hash_table, &key);
}

// Vinode Hash helpers
int vinode_hash_equal(void *key1, void *key2) {
  struct vinode_key *k1 = key1, *k2 = key2;
  return (k1->sb == k2->sb && k1->inum == k2->inum);
}

size_t vinode_hash_func(void *key) {
  struct vinode_key *k = key;
  return ((size_t)k->sb + k->inum) % HASH_TABLE_SIZE;
}

struct vinode *hash_get_vinode(struct super_block *sb, int inum) {
  struct vinode_key key = {.sb = sb, .inum = inum};
  return (struct vinode *)vinode_hash_table.virtual_hash_get(&vinode_hash_table, &key);
}

int hash_put_vinode(struct vinode *vinode) {
  struct vinode_key *key = alloc_page();
  key->sb = vinode->sb;
  key->inum = vinode->inum;
  return vinode_hash_table.virtual_hash_put(&vinode_hash_table, key, vinode);
}

int hash_erase_vinode(struct vinode *vinode) {
  struct vinode_key key = {.sb = vinode->sb, .inum = vinode->inum};
  return vinode_hash_table.virtual_hash_erase(&vinode_hash_table, &key);
}

struct vinode *default_alloc_vinode(struct super_block *sb) {
  struct vinode *vinode = (struct vinode *)alloc_page();
  vinode->sb = sb;
  vinode->ref = 0;
  return vinode;
}

struct file *alloc_vfs_file(struct dentry *dentry, int readable, int writable, int offset) {
  struct file *file = alloc_page();
  file->f_dentry = dentry;
  dentry->d_ref += 1;
  file->readable = readable;
  file->writable = writable;
  file->offset = 0;
  file->status = FD_OPENED;
  return file;
}

void get_base_name(const char *path, char *base_name) {
  char path_copy[MAX_PATH_LEN];
  strcpy(path_copy, path);
  char *token = strtok(path_copy, "/");
  char *last_token = NULL;
  while (token != NULL) {
    last_token = token;
    token = strtok(NULL, "/");
  }
  strcpy(base_name, last_token);
}
```

- 用以下代码替换 `proc_file.h`

```c
#ifndef _PROC_FILE_H_
#define _PROC_FILE_H_

#include "spike_interface/spike_file.h"
#include "util/types.h"
#include "vfs.h"

int do_open(char *pathname, int flags);
int do_read(int fd, char *buf, uint64 count);
int do_write(int fd, char *buf, uint64 count);
int do_lseek(int fd, int offset, int whence);
int do_stat(int fd, struct istat *istat);
int do_disk_stat(int fd, struct istat *istat);
int do_close(int fd);

int do_opendir(char *pathname);
int do_readdir(int fd, struct dir *dir);
int do_mkdir(char *pathname);
int do_closedir(int fd);

int do_link(char *oldpath, char *newpath);
int do_unlink(char *path);

int do_ccwd(char *pathname);
int do_rcwd(char *pathname);

void fs_init(void);

typedef struct proc_file_management_t {
  struct dentry *cwd;
  struct file opened_files[MAX_FILES];
  int nfiles;
} proc_file_management;

proc_file_management *init_proc_file_management(void);

void reclaim_proc_file_management(proc_file_management *pfiles);

#endif
```

- 用以下代码替换 `proc_file.c`

```c
#include "proc_file.h"
#include "hostfs.h"
#include "pmm.h"
#include "process.h"
#include "ramdev.h"
#include "rfs.h"
#include "riscv.h"
#include "spike_interface/spike_file.h"
#include "spike_interface/spike_utils.h"
#include "util/functions.h"
#include "util/string.h"

void fs_init(void) {
  vfs_init();
  if( register_hostfs() < 0 ) panic( "fs_init: cannot register hostfs.\n" );
  struct device *hostdev = init_host_device("HOSTDEV");
  vfs_mount("HOSTDEV", MOUNT_AS_ROOT);
  if( register_rfs() < 0 ) panic( "fs_init: cannot register rfs.\n" );
  struct device *ramdisk0 = init_rfs_device("RAMDISK0");
  rfs_format_dev(ramdisk0);
  vfs_mount("RAMDISK0", MOUNT_DEFAULT);
}

proc_file_management *init_proc_file_management(void) {
  proc_file_management *pfiles = (proc_file_management *)alloc_page();
  pfiles->cwd = vfs_root_dentry;
  pfiles->nfiles = 0;
  for (int fd = 0; fd < MAX_FILES; ++fd) pfiles->opened_files[fd].status = FD_NONE;
  sprint("FS: created a file management struct for a process.\n");
  return pfiles;
}

void reclaim_proc_file_management(proc_file_management *pfiles) {
  free_page(pfiles);
}

struct file *get_opened_file(int fd) {
  struct file *pfile = NULL;
  for (int i = 0; i < MAX_FILES; ++i) {
    pfile = &(current->pfiles->opened_files[i]);
    if (i == fd) break;
  }
  if (pfile == NULL) panic("do_read: invalid fd!\n");
  return pfile;
}

int do_open(char *pathname, int flags) {
  struct file *opened_file = vfs_open(pathname, flags);
  if (opened_file == NULL) return -1;
  int fd = 0;
  if (current->pfiles->nfiles >= MAX_FILES) panic("do_open: no file entry for current process!\n");
  struct file *pfile;
  for (fd = 0; fd < MAX_FILES; ++fd) {
    pfile = &(current->pfiles->opened_files[fd]);
    if (pfile->status == FD_NONE) break;
  }
  memcpy(pfile, opened_file, sizeof(struct file));
  ++current->pfiles->nfiles;
  return fd;
}

int do_read(int fd, char *buf, uint64 count) {
  struct file *pfile = get_opened_file(fd);
  if (pfile->readable == 0) panic("do_read: no readable file!\n");
  char buffer[count + 1];
  int len = vfs_read(pfile, buffer, count);
  buffer[count] = '\0';
  strcpy(buf, buffer);
  return len;
}

int do_write(int fd, char *buf, uint64 count) {
  struct file *pfile = get_opened_file(fd);
  if (pfile->writable == 0) panic("do_write: cannot write file!\n");
  return vfs_write(pfile, buf, count);
}

int do_lseek(int fd, int offset, int whence) {
  struct file *pfile = get_opened_file(fd);
  return vfs_lseek(pfile, offset, whence);
}

int do_stat(int fd, struct istat *istat) {
  struct file *pfile = get_opened_file(fd);
  return vfs_stat(pfile, istat);
}

int do_disk_stat(int fd, struct istat *istat) {
  struct file *pfile = get_opened_file(fd);
  return vfs_disk_stat(pfile, istat);
}

int do_close(int fd) {
  struct file *pfile = get_opened_file(fd);
  return vfs_close(pfile);
}

int do_opendir(char *pathname) {
  struct file *opened_file = vfs_opendir(pathname);
  if (opened_file == NULL) return -1;
  int fd = 0;
  struct file *pfile;
  for (fd = 0; fd < MAX_FILES; ++fd) {
    pfile = &(current->pfiles->opened_files[fd]);
    if (pfile->status == FD_NONE) break;
  }
  if (pfile->status != FD_NONE) panic("do_opendir: no file entry for current process!\n");
  memcpy(pfile, opened_file, sizeof(struct file));
  ++current->pfiles->nfiles;
  return fd;
}

int do_readdir(int fd, struct dir *dir) {
  struct file *pfile = get_opened_file(fd);
  return vfs_readdir(pfile, dir);
}

int do_mkdir(char *pathname) {
  return vfs_mkdir(pathname);
}

int do_closedir(int fd) {
  struct file *pfile = get_opened_file(fd);
  return vfs_closedir(pfile);
}

int do_link(char *oldpath, char *newpath) {
  return vfs_link(oldpath, newpath);
}

int do_unlink(char *path) {
  return vfs_unlink(path);
}

int do_ccwd(char *pathname) {
  struct dentry *parent = NULL;
  char miss_name[MAX_PATH_LEN];
  struct dentry *dir_dentry = lookup_final_dentry(pathname, &parent, miss_name);
  if (!dir_dentry || dir_dentry->dentry_inode->type != DIR_I) return -1;
  current->pfiles->cwd = dir_dentry;
  return 0;
}

int do_rcwd(char *pathname) {
  struct dentry *this = current->pfiles->cwd;
  if (this == vfs_root_dentry) {
    strcpy(pathname, "/");
    return 0;
  }
  char stack[MAX_MOUNTS][MAX_DENTRY_NAME_LEN];
  int top = 0;
  // Add counter to prevent infinite loop in case of circular dentry chain
  int count = 0;
  while (this != vfs_root_dentry && this != NULL && count < MAX_MOUNTS) {
    strcpy(stack[top++], this->name);
    this = this->parent;
    count++;
  }
  if (count >= MAX_MOUNTS) {
    panic("do_rcwd: circular dentry chain detected!\n");
  }
  pathname[0] = '\0';
  for (int i = top - 1; i >= 0; i--) {
    strcat(pathname, "/");
    strcat(pathname, stack[i]);
  }
  return 0;
}
```

- 用以下代码替换 `syscall.c`

```c
#include <stdint.h>
#include <errno.h>
#include "util/types.h"
#include "syscall.h"
#include "string.h"
#include "process.h"
#include "util/functions.h"
#include "pmm.h"
#include "vmm.h"
#include "sched.h"
#include "proc_file.h"
#include "spike_interface/spike_utils.h"

ssize_t sys_user_print(const char* buf, size_t n) {
  assert( current );
  char* pa = (char*)user_va_to_pa((pagetable_t)(current->pagetable), (void*)buf);
  sprint(pa);
  return 0;
}

ssize_t sys_user_exit(uint64 code) {
  sprint("User exit with code:%d.\n", code);
  free_process( current );
  schedule();
  return 0;
}

uint64 sys_user_allocate_page() {
  void* pa = alloc_page();
  uint64 va;
  if (current->user_heap.free_pages_count > 0) {
    va =  current->user_heap.free_pages_address[--current->user_heap.free_pages_count];
  } else {
    va = current->user_heap.heap_top;
    current->user_heap.heap_top += PGSIZE;
    current->mapped_info[HEAP_SEGMENT].npages++;
  }
  user_vm_map((pagetable_t)current->pagetable, va, PGSIZE, (uint64)pa,
         prot_to_type(PROT_WRITE | PROT_READ, 1));
  return va;
}

uint64 sys_user_free_page(uint64 va) {
  user_vm_unmap((pagetable_t)current->pagetable, va, PGSIZE, 1);
  current->user_heap.free_pages_address[current->user_heap.free_pages_count++] = va;
  return 0;
}

ssize_t sys_user_fork() {
  sprint("User call fork.\n");
  return do_fork( current );
}

ssize_t sys_user_yield() {
  current->status = READY;
  insert_to_ready_queue( current );
  schedule();
  return 0;
}

ssize_t sys_user_open(char *pathva, int flags) {
  char* pathpa = (char*)user_va_to_pa((pagetable_t)(current->pagetable), pathva);
  return do_open(pathpa, flags);
}

ssize_t sys_user_read(int fd, char *bufva, uint64 count) {
  int i = 0;
  while (i < count) {
    uint64 addr = (uint64)bufva + i;
    uint64 pa = lookup_pa((pagetable_t)current->pagetable, addr);
    uint64 off = addr - ROUNDDOWN(addr, PGSIZE);
    uint64 len = count - i < PGSIZE - off ? count - i : PGSIZE - off;
    uint64 r = do_read(fd, (char *)pa + off, len);
    i += r; if (r < len) return i;
  }
  return count;
}

ssize_t sys_user_write(int fd, char *bufva, uint64 count) {
  int i = 0;
  while (i < count) {
    uint64 addr = (uint64)bufva + i;
    uint64 pa = lookup_pa((pagetable_t)current->pagetable, addr);
    uint64 off = addr - ROUNDDOWN(addr, PGSIZE);
    uint64 len = count - i < PGSIZE - off ? count - i : PGSIZE - off;
    uint64 r = do_write(fd, (char *)pa + off, len);
    i += r; if (r < len) return i;
  }
  return count;
}

ssize_t sys_user_lseek(int fd, int offset, int whence) {
  return do_lseek(fd, offset, whence);
}

ssize_t sys_user_stat(int fd, struct istat *istat) {
  struct istat * istatpa = (struct istat*)user_va_to_pa((pagetable_t)(current->pagetable), istat);
  return do_stat(fd, istatpa);
}

ssize_t sys_user_disk_stat(int fd, struct istat *istat) {
  struct istat * istatpa = (struct istat*)user_va_to_pa((pagetable_t)(current->pagetable), istat);
  return do_disk_stat(fd, istatpa);
}

ssize_t sys_user_close(int fd) {
  return do_close(fd);
}

ssize_t sys_user_opendir(char *pathva) {
  char* pathpa = (char*)user_va_to_pa((pagetable_t)(current->pagetable), pathva);
  return do_opendir(pathpa);
}

ssize_t sys_user_readdir(int fd, struct dir *dirva) {
  struct dir * dirpa = (struct dir*)user_va_to_pa((pagetable_t)(current->pagetable), dirva);
  return do_readdir(fd, dirpa);
}

ssize_t sys_user_mkdir(char *pathva) {
  char* pathpa = (char*)user_va_to_pa((pagetable_t)(current->pagetable), pathva);
  return do_mkdir(pathpa);
}

ssize_t sys_user_closedir(int fd) {
  return do_closedir(fd);
}

ssize_t sys_user_link(char * vfn1, char * vfn2){
  char * pfn1 = (char*)user_va_to_pa((pagetable_t)(current->pagetable), (void*)vfn1);
  char * pfn2 = (char*)user_va_to_pa((pagetable_t)(current->pagetable), (void*)vfn2);
  return do_link(pfn1, pfn2);
}

ssize_t sys_user_unlink(char * vfn){
  char * pfn = (char*)user_va_to_pa((pagetable_t)(current->pagetable), (void*)vfn);
  return do_unlink(pfn);
}

ssize_t sys_user_rcwd(char *pathva) {
  char pathpa[MAX_PATH_LEN];
  int ret = do_rcwd(pathpa);
  if (ret < 0) return ret;
  char *user_pathpa = (char*)user_va_to_pa((pagetable_t)(current->pagetable), pathva);
  strcpy(user_pathpa, pathpa);
  return 0;
}

ssize_t sys_user_ccwd(char *pathva) {
  char *pathpa = (char*)user_va_to_pa((pagetable_t)(current->pagetable), pathva);
  return do_ccwd(pathpa);
}

long do_syscall(long a0, long a1, long a2, long a3, long a4, long a5, long a6, long a7) {
  switch (a0) {
    case SYS_user_print: return sys_user_print((const char*)a1, a2);
    case SYS_user_exit: return sys_user_exit(a1);
    case SYS_user_allocate_page: return sys_user_allocate_page();
    case SYS_user_free_page: return sys_user_free_page(a1);
    case SYS_user_fork: return sys_user_fork();
    case SYS_user_yield: return sys_user_yield();
    case SYS_user_open: return sys_user_open((char *)a1, a2);
    case SYS_user_read: return sys_user_read(a1, (char *)a2, a3);
    case SYS_user_write: return sys_user_write(a1, (char *)a2, a3);
    case SYS_user_lseek: return sys_user_lseek(a1, a2, a3);
    case SYS_user_stat: return sys_user_stat(a1, (struct istat *)a2);
    case SYS_user_disk_stat: return sys_user_disk_stat(a1, (struct istat *)a2);
    case SYS_user_close: return sys_user_close(a1);
    case SYS_user_opendir: return sys_user_opendir((char *)a1);
    case SYS_user_readdir: return sys_user_readdir(a1, (struct dir *)a2);
    case SYS_user_mkdir: return sys_user_mkdir((char *)a1);
    case SYS_user_closedir: return sys_user_closedir(a1);
    case SYS_user_link: return sys_user_link((char *)a1, (char *)a2);
    case SYS_user_unlink: return sys_user_unlink((char *)a1);
    case SYS_user_rcwd: return sys_user_rcwd((char *)a1);
    case SYS_user_ccwd: return sys_user_ccwd((char *)a1);
    default: panic("Unknown syscall %ld \n", a0);
  }
}
```

- 用以下代码替换 `syscall.h`

```c
#ifndef _SYSCALL_H_
#define _SYSCALL_H_

#define SYS_user_base 64
#define SYS_user_print (SYS_user_base + 0)
#define SYS_user_exit (SYS_user_base + 1)
#define SYS_user_allocate_page (SYS_user_base + 2)
#define SYS_user_free_page (SYS_user_base + 3)
#define SYS_user_fork (SYS_user_base + 4)
#define SYS_user_yield (SYS_user_base + 5)
#define SYS_user_open (SYS_user_base + 17)
#define SYS_user_read (SYS_user_base + 18)
#define SYS_user_write (SYS_user_base + 19)
#define SYS_user_lseek (SYS_user_base + 20)
#define SYS_user_stat (SYS_user_base + 21)
#define SYS_user_disk_stat (SYS_user_base + 22)
#define SYS_user_close (SYS_user_base + 23)
#define SYS_user_opendir  (SYS_user_base + 24)
#define SYS_user_readdir  (SYS_user_base + 25)
#define SYS_user_mkdir    (SYS_user_base + 26)
#define SYS_user_closedir (SYS_user_base + 27)
#define SYS_user_link   (SYS_user_base + 28)
#define SYS_user_unlink (SYS_user_base + 29)
#define SYS_user_rcwd (SYS_user_base + 30)
#define SYS_user_ccwd (SYS_user_base + 31)

long do_syscall(long a0, long a1, long a2, long a3, long a4, long a5, long a6, long a7);

#endif
```

