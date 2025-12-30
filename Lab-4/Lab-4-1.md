# 任务描述

- 在PKE操作系统内核中完善对文件的操作
- 使得它能够正确处理用户进程的打开、创建、读写文件请求
- 使得执行obj/app_file时产生如下输出

```bash
$ spike ./obj/riscv-pke ./obj/app_file
In m_start, hartid:0
HTIF is available!
(Emulated) memory size: 2048 MB
Enter supervisor mode...
PKE kernel start 0x0000000080000000, PKE kernel end: 0x000000008000e000, PKE kernel size: 0x000000000000e000 .
free physical memory address: [0x000000008000e000, 0x0000000087ffffff] 
kernel memory manager is initializing ...
KERN_BASE 0x0000000080000000
physical address of _etext is: 0x0000000080008000
kernel page table is on 
RAMDISK0: base address of RAMDISK0 is: 0x0000000087f37000
RFS: format RAMDISK0 done!
Switch to user mode...
in alloc_proc. user frame 0x0000000087f2f000, user stack 0x000000007ffff000, user kstack 0x0000000087f2e000 
FS: created a file management struct for a process.
in alloc_proc. build proc_file_management successfully.
User application is loading.
Application: obj/app_file
CODE_SEGMENT added at mapped info offset:3
DATA_SEGMENT added at mapped info offset:4
Application program entry point (virtual address): 0x00000000000100b0
going to insert process 0 to ready queue.
going to schedule process 0 to run.
======== Test 1: read host file  ========
read: /hostfile.txt
file descriptor fd: 0
read content: 
This is an apple. 
Apples are good for our health. 
======== Test 2: create/write rfs file ========
write: /RAMDISK0/ramfile
file descriptor fd: 0
write content: 
This is an apple. 
Apples are good for our health. 
======== Test 3: read rfs file ========
read: /RAMDISK0/ramfile
file descriptor fd: 0
read content: 
This is an apple. 
Apples are good for our health. 
======== Test 4: open twice ========
file descriptor fd1(ramfile): 0
file descriptor fd2(ramfile): 1
write content: 
hello world
read content: 
hello world
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
    default:
      panic("Unknown syscall %ld \n", a0);
  }
}
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
    .viop_lookup = rfs_lookup,

    .viop_write_back_vinode = rfs_write_back_vinode,
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
  free_dinode->size = 0;
  free_dinode->type = R_FILE;
  free_dinode->nlinks = 1;
  free_dinode->blocks = 0;

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

# 通关操作

- 用以下代码替换`strap.c`

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
  sprint("handle_page_fault: %lx\n", stval);

  if (mcause != CAUSE_STORE_PAGE_FAULT && mcause != CAUSE_LOAD_PAGE_FAULT) {
    sprint("unknown page fault.\n");
    panic("unsupported page fault.\n");
  }

  if (current == NULL) panic("no current process when page fault happens.\n");

  mapped_region *stack_region = &current->mapped_info[STACK_SEGMENT];
  uint64 fault_page = ROUNDDOWN(stval, PGSIZE);

  if (fault_page >= stack_region->va &&
      fault_page < stack_region->va + stack_region->npages * PGSIZE) {
    panic("page fault triggered on an already mapped page.\n");
  }

  if (fault_page != stack_region->va - PGSIZE) {
    sprint("page fault address %lx is not adjacent to current stack.\n", fault_page);
    panic("currently we only grow stack downward by one page.\n");
  }

  void *new_page = alloc_page();
  if (new_page == NULL) panic("no memory for stack growth.\n");

  user_vm_map(current->pagetable, fault_page, PGSIZE, (uint64)new_page,
              prot_to_type(PROT_READ | PROT_WRITE, 1));
  stack_region->va = fault_page;
  stack_region->npages += 1;
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
      panic("unexpected exception happened.\n");
      break;
  }

  switch_to(current);
}
```

- 用以下代码替换`mtrap.c`

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
      panic("unexpected exception happened in M-mode.\n");
      break;
  }
}
```

- 用以下代码替换`vmm.c`

```c
#include "vmm.h"
#include "riscv.h"
#include "pmm.h"
#include "util/types.h"
#include "memlayout.h"
#include "util/string.h"
#include "spike_interface/spike_utils.h"
#include "util/functions.h"

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

uint64 prot_to_type(int prot, int user) {
  uint64 perm = 0;
  if (prot & PROT_READ) perm |= PTE_R | PTE_A;
  if (prot & PROT_WRITE) perm |= PTE_W | PTE_D;
  if (prot & PROT_EXEC) perm |= PTE_X | PTE_A;
  if (perm == 0) perm = PTE_R;
  if (user) perm |= PTE_U;
  return perm;
}

pte_t *page_walk(pagetable_t page_dir, uint64 va, int alloc) {
  if (va >= MAXVA) panic("page_walk");

  pagetable_t pt = page_dir;

  for (int level = 2; level > 0; level--) {
    pte_t *pte = pt + PX(level, va);

    if (*pte & PTE_V) {
      pt = (pagetable_t)PTE2PA(*pte);
    } else {
      if( alloc && ((pt = (pte_t *)alloc_page(1)) != 0) ){
        memset(pt, 0, PGSIZE);
        *pte = PA2PTE(pt) | PTE_V;
      } else
        return 0;
    }
  }

  return pt + PX(0, va);
}

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

extern char _etext[];
pagetable_t g_kernel_pagetable;

void kern_vm_map(pagetable_t page_dir, uint64 va, uint64 pa, uint64 sz, int perm) {
  if (map_pages(page_dir, va, sz, pa, perm) != 0) panic("kern_vm_map");
}

void kern_vm_init(void) {
  pagetable_t t_page_dir;

  t_page_dir = (pagetable_t)alloc_page();
  memset(t_page_dir, 0, PGSIZE);

  kern_vm_map(t_page_dir, KERN_BASE, DRAM_BASE, (uint64)_etext - KERN_BASE,
         prot_to_type(PROT_READ | PROT_EXEC, 0));

  sprint("KERN_BASE 0x%lx\n", lookup_pa(t_page_dir, KERN_BASE));

  kern_vm_map(t_page_dir, (uint64)_etext, (uint64)_etext, PHYS_TOP - (uint64)_etext,
         prot_to_type(PROT_READ | PROT_WRITE, 0));

  sprint("physical address of _etext is: 0x%lx\n", lookup_pa(t_page_dir, (uint64)_etext));

  g_kernel_pagetable = t_page_dir;
}

void *user_va_to_pa(pagetable_t page_dir, void *va) {
  uint64 va_val = (uint64)va;
  pte_t *pte = page_walk(page_dir, va_val, 0);
  if (pte == 0) return 0;
  if ((*pte & PTE_V) == 0) return 0;
  if ((*pte & (PTE_R | PTE_W | PTE_X)) == 0) return 0;
  uint64 pa = PTE2PA(*pte) + (va_val & (PGSIZE - 1));
  return (void *)pa;
}

void user_vm_map(pagetable_t page_dir, uint64 va, uint64 size, uint64 pa, int perm) {
  if (map_pages(page_dir, va, size, pa, perm) != 0) {
    panic("fail to user_vm_map .\n");
  }
}

void user_vm_unmap(pagetable_t page_dir, uint64 va, uint64 size, int free) {
  if (size == 0) return;

  uint64 first = ROUNDDOWN(va, PGSIZE);
  uint64 last = ROUNDDOWN(va + size - 1, PGSIZE);

  for (uint64 addr = first; addr <= last; addr += PGSIZE) {
    pte_t *pte = page_walk(page_dir, addr, 0);
    if (pte == 0 || (*pte & PTE_V) == 0) continue;

    if (free) {
      void *pa = (void *)PTE2PA(*pte);
      free_page(pa);
    }

    *pte = 0;
  }
}
```

- 用以下代码替换`process.c`

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

  procs[i].mapped_info[HEAP_SEGMENT].va = USER_FREE_ADDRESS_START;
  procs[i].mapped_info[HEAP_SEGMENT].npages = 0;
  procs[i].mapped_info[HEAP_SEGMENT].seg_type = HEAP_SEGMENT;

  procs[i].total_mapped_region = 4;

  procs[i].pfiles = init_proc_file_management();

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

  for( int i=0; i<parent->total_mapped_region; i++ ){
    switch( parent->mapped_info[i].seg_type ){
      case CONTEXT_SEGMENT:
        *child->trapframe = *parent->trapframe;
        break;
      case STACK_SEGMENT:
        memcpy( (void*)lookup_pa(child->pagetable, child->mapped_info[STACK_SEGMENT].va),
          (void*)lookup_pa(parent->pagetable, parent->mapped_info[i].va), PGSIZE );
        break;
      case HEAP_SEGMENT: {
        int free_block_filter[MAX_HEAP_PAGES];
        memset(free_block_filter, 0, MAX_HEAP_PAGES);
        uint64 heap_bottom = parent->user_heap.heap_bottom;
        for (int i = 0; i < parent->user_heap.free_pages_count; i++) {
          int index = (parent->user_heap.free_pages_address[i] - heap_bottom) / PGSIZE;
          free_block_filter[index] = 1;
        }

        for (uint64 heap_block = parent->user_heap.heap_bottom;
             heap_block < parent->user_heap.heap_top; heap_block += PGSIZE) {
          if (free_block_filter[(heap_block - heap_bottom) / PGSIZE])
            continue;

          void* child_pa = alloc_page();
          memcpy(child_pa, (void*)lookup_pa(parent->pagetable, heap_block), PGSIZE);
          user_vm_map((pagetable_t)child->pagetable, heap_block, PGSIZE, (uint64)child_pa,
                      prot_to_type(PROT_WRITE | PROT_READ, 1));
        }

        child->mapped_info[HEAP_SEGMENT].npages = parent->mapped_info[HEAP_SEGMENT].npages;
        memcpy((void*)&child->user_heap, (void*)&parent->user_heap, sizeof(parent->user_heap));
        break;
      }
      case CODE_SEGMENT: {
        uint64 code_base = parent->mapped_info[i].va;
        uint64 code_pages = parent->mapped_info[i].npages;
        for (uint64 page = 0; page < code_pages; page++) {
          uint64 va = code_base + page * PGSIZE;
          uint64 pa = lookup_pa(parent->pagetable, va);
          user_vm_map((pagetable_t)child->pagetable, va, PGSIZE, pa,
            prot_to_type(PROT_READ | PROT_EXEC, 1));
        }

        child->mapped_info[child->total_mapped_region].va = code_base;
        child->mapped_info[child->total_mapped_region].npages = code_pages;
        child->mapped_info[child->total_mapped_region].seg_type = CODE_SEGMENT;
        child->total_mapped_region++;
        break;
      }
    }
  }

  child->status = READY;
  child->trapframe->regs.a0 = 0;
  child->parent = parent;
  insert_to_ready_queue( child );

  return child->pid;
}
```

- 用以下代码替换`syscall.c`

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
  return do_fork( current );
}

ssize_t sys_user_yield() {
  current->status = READY;
  insert_to_ready_queue(current);
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
  struct istat * pistat = (struct istat *)user_va_to_pa((pagetable_t)(current->pagetable), istat);
  return do_stat(fd, pistat);
}

ssize_t sys_user_disk_stat(int fd, struct istat *istat) {
  struct istat * pistat = (struct istat *)user_va_to_pa((pagetable_t)(current->pagetable), istat);
  return do_disk_stat(fd, pistat);
}

ssize_t sys_user_close(int fd) {
  return do_close(fd);
}

long do_syscall(long a0, long a1, long a2, long a3, long a4, long a5, long a6, long a7) {
  switch (a0) {
    case SYS_user_print:
      return sys_user_print((const char*)a1, a2);
    case SYS_user_exit:
      return sys_user_exit(a1);
    case SYS_user_allocate_page:
      return sys_user_allocate_page();
    case SYS_user_free_page:
      return sys_user_free_page(a1);
    case SYS_user_fork:
      return sys_user_fork();
    case SYS_user_yield:
      return sys_user_yield();
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
    default:
      panic("Unknown syscall %ld \n", a0);
  }
}
```

- 用以下代码替换`rfs.c`

```c
#include "rfs.h"

#include "pmm.h"
#include "ramdev.h"
#include "spike_interface/spike_utils.h"
#include "util/string.h"
#include "vfs.h"

const struct vinode_ops rfs_i_ops = {
    .viop_read = rfs_read,
    .viop_write = rfs_write,
    .viop_create = rfs_create,
    .viop_lseek = rfs_lseek,
    .viop_disk_stat = rfs_disk_stat,
    .viop_lookup = rfs_lookup,

    .viop_write_back_vinode = rfs_write_back_vinode,
};

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

int rfs_format_dev(struct device *dev) {
  struct rfs_device *rdev = rfs_device_list[dev->dev_id];

  struct super_block *super = (struct super_block *)rdev->iobuffer;
  super->magic = RFS_MAGIC;
  super->size =
      1 + RFS_MAX_INODE_BLKNUM + 1 + RFS_MAX_INODE_BLKNUM * RFS_DIRECT_BLKNUM;
  super->nblocks = RFS_MAX_INODE_BLKNUM * RFS_DIRECT_BLKNUM;
  super->ninodes = RFS_BLKSIZE / RFS_INODESIZE * RFS_MAX_INODE_BLKNUM;

  if (rfs_w1block(rdev, RFS_BLK_OFFSET_SUPER) != 0)
    panic("RFS: failed to write superblock!\n");

  struct rfs_dinode *p_dinode = (struct rfs_dinode *)rdev->iobuffer;
  for (int i = 0; i < RFS_BLKSIZE / RFS_INODESIZE; ++i) {
    p_dinode->size = 0;
    p_dinode->type = R_FREE;
    p_dinode->nlinks = 0;
    p_dinode->blocks = 0;
    p_dinode = (struct rfs_dinode *)((char *)p_dinode + RFS_INODESIZE);
  }

  for (int inode_block = 0; inode_block < RFS_MAX_INODE_BLKNUM; ++inode_block) {
    if (rfs_w1block(rdev, RFS_BLK_OFFSET_INODE + inode_block) != 0)
      panic("RFS: failed to initialize empty inodes!\n");
  }

  struct rfs_dinode root_dinode;
  root_dinode.size = 0;
  root_dinode.type = R_DIR;
  root_dinode.nlinks = 1;
  root_dinode.blocks = 1;
  root_dinode.addrs[0] = RFS_BLK_OFFSET_FREE;

  if (rfs_write_dinode(rdev, &root_dinode, 0) != 0) {
    sprint("RFS: failed to write root inode!\n");
    return -1;
  }

  int *freemap = (int *)rdev->iobuffer;
  memset(freemap, 0, RFS_BLKSIZE);
  freemap[0] = 1;

  if (rfs_w1block(rdev, RFS_BLK_OFFSET_BITMAP) != 0) {
    sprint("RFS: failed to write bitmap!\n");
    return -1;
  }

  sprint("RFS: format %s done!\n", dev->dev_name);
  return 0;
}

int rfs_r1block(struct rfs_device *rfs_dev, int n_block) {
  return dop_read(rfs_dev, n_block);
}

int rfs_w1block(struct rfs_device *rfs_dev, int n_block) {
  return dop_write(rfs_dev, n_block);
}

struct rfs_dinode *rfs_read_dinode(struct rfs_device *rdev, int n_inode) {
  int n_block = n_inode / (RFS_BLKSIZE / RFS_INODESIZE) + RFS_BLK_OFFSET_INODE;
  int offset = n_inode % (RFS_BLKSIZE / RFS_INODESIZE);

  if (dop_read(rdev, n_block) != 0) return NULL;
  struct rfs_dinode *dinode = (struct rfs_dinode *)alloc_page();
  memcpy(dinode, (char *)rdev->iobuffer + offset * RFS_INODESIZE,
         sizeof(struct rfs_dinode));
  return dinode;
}

int rfs_write_dinode(struct rfs_device *rdev, const struct rfs_dinode *dinode,
                     int n_inode) {
  int n_block = n_inode / (RFS_BLKSIZE / RFS_INODESIZE) + RFS_BLK_OFFSET_INODE;
  int offset = n_inode % (RFS_BLKSIZE / RFS_INODESIZE);

  dop_read(rdev, n_block);
  memcpy(rdev->iobuffer + offset * RFS_INODESIZE, dinode,
         sizeof(struct rfs_dinode));
  int ret = dop_write(rdev, n_block);

  return ret;
}

int rfs_alloc_block(struct super_block *sb) {
  int free_block = -1;
  int *freemap = (int *)sb->s_fs_info;
  for (int block = 0; block < sb->nblocks; ++block) {
    if (freemap[block] == 0) {
      freemap[block] = 1;
      free_block = RFS_BLK_OFFSET_FREE + block;
      break;
    }
  }
  if (free_block == -1) panic("rfs_alloc_block: no more free block!\n");
  return free_block;
}

int rfs_free_block(struct super_block *sb, int block_num) {
  int *freemap = (int *)sb->s_fs_info;
  freemap[block_num - RFS_BLK_OFFSET_FREE] = 0;
  return 0;
}

int rfs_add_direntry(struct vinode *dir, const char *name, int inum) {
  struct rfs_device *rdev = rfs_device_list[dir->sb->s_dev->dev_id];
  int n_block = dir->addrs[dir->size / RFS_BLKSIZE];
  if (rfs_r1block(rdev, n_block) != 0) {
    return -1;
  }

  char *addr = (char *)rdev->iobuffer + dir->size % RFS_BLKSIZE;
  struct rfs_direntry *p_direntry = (struct rfs_direntry *)addr;
  p_direntry->inum = inum;
  strcpy(p_direntry->name, name);

  if (rfs_w1block(rdev, n_block) != 0) {
    return -1;
  }

  dir->size += sizeof(struct rfs_direntry);

  if (rfs_write_back_vinode(dir) != 0) {
    return -1;
  }

  return 0;
}

struct vinode *rfs_alloc_vinode(struct super_block *sb) {
  struct vinode *vinode = default_alloc_vinode(sb);
  vinode->i_ops = &rfs_i_ops;
  return vinode;
}

int rfs_write_back_vinode(struct vinode *vinode) {
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
    return -1;
  }

  return 0;
}

int rfs_update_vinode(struct vinode *vinode) {
  struct rfs_device *rdev = rfs_device_list[vinode->sb->s_dev->dev_id];
  struct rfs_dinode *dinode = rfs_read_dinode(rdev, vinode->inum);
  if (dinode == NULL) {
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

ssize_t rfs_read(struct vinode *f_inode, char *r_buf, ssize_t len,
                 int *offset) {
  if (f_inode->size < (*offset + len)) len = f_inode->size - *offset;

  char buffer[len + 1];

  int align = *offset % RFS_BLKSIZE;
  int block_offset = *offset / RFS_BLKSIZE;
  int buf_offset = 0;

  int readtimes = (align + len) / RFS_BLKSIZE;
  int remain = (align + len) % RFS_BLKSIZE;

  struct rfs_device *rdev = rfs_device_list[f_inode->sb->s_dev->dev_id];

  rfs_r1block(rdev, f_inode->addrs[block_offset]);
  int first_block_len = (readtimes == 0 ? len : RFS_BLKSIZE - align);
  memcpy(buffer + buf_offset, rdev->iobuffer + align, first_block_len);
  buf_offset += first_block_len;
  block_offset++;
  readtimes--;

  if (readtimes >= 0) {
    while (readtimes != 0) {
      rfs_r1block(rdev, f_inode->addrs[block_offset]);
      memcpy(buffer + buf_offset, rdev->iobuffer, RFS_BLKSIZE);
      buf_offset += RFS_BLKSIZE;
      block_offset++;
      readtimes--;
    }

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

ssize_t rfs_write(struct vinode *f_inode, const char *w_buf, ssize_t len,
                  int *offset) {
  int align = *offset % RFS_BLKSIZE;
  int writetimes = (len + align) / RFS_BLKSIZE;
  int remain = (len + align) % RFS_BLKSIZE;

  int buf_offset = 0;
  int block_offset = *offset / RFS_BLKSIZE;

  struct rfs_device *rdev = rfs_device_list[f_inode->sb->s_dev->dev_id];

  if (align != 0) {
    rfs_r1block(rdev, f_inode->addrs[block_offset]);
    int first_block_len = (writetimes == 0 ? len : RFS_BLKSIZE - align);
    memcpy(rdev->iobuffer + align, w_buf, first_block_len);
    rfs_w1block(rdev, f_inode->addrs[block_offset]);

    buf_offset += first_block_len;
    block_offset++;
    writetimes--;
  }

  if (writetimes >= 0) {
    while (writetimes != 0) {
      if (block_offset == f_inode->blocks) {
        f_inode->addrs[block_offset] = rfs_alloc_block(f_inode->sb);
        f_inode->blocks++;
      }

      memcpy(rdev->iobuffer, w_buf + buf_offset, RFS_BLKSIZE);
      rfs_w1block(rdev, f_inode->addrs[block_offset]);

      buf_offset += RFS_BLKSIZE;
      block_offset++;
      writetimes--;
    }

    if (remain > 0) {
      if (block_offset == f_inode->blocks) {
        f_inode->addrs[block_offset] = rfs_alloc_block(f_inode->sb);
        ++f_inode->blocks;
      }
      memcpy(rdev->iobuffer, w_buf + buf_offset, remain);
      rfs_w1block(rdev, f_inode->addrs[block_offset]);
    }
  }

  f_inode->size =
      (f_inode->size < *offset + len ? *offset + len : f_inode->size);

  *offset += len;
  return len;
}

struct vinode *rfs_lookup(struct vinode *parent, struct dentry *sub_dentry) {
  struct rfs_direntry *p_direntry = NULL;
  struct vinode *child_vinode = NULL;

  int total_direntrys = parent->size / sizeof(struct rfs_direntry);
  int one_block_direntrys = RFS_BLKSIZE / sizeof(struct rfs_direntry);

  struct rfs_device *rdev = rfs_device_list[parent->sb->s_dev->dev_id];

  for (int i = 0; i < total_direntrys; ++i) {
    if (i % one_block_direntrys == 0) {
      rfs_r1block(rdev, parent->addrs[i / one_block_direntrys]);
      p_direntry = (struct rfs_direntry *)rdev->iobuffer;
    }
    if (strcmp(p_direntry->name, sub_dentry->name) == 0) {
      child_vinode = rfs_alloc_vinode(parent->sb);
      child_vinode->inum = p_direntry->inum;
      rfs_update_vinode(child_vinode);
      break;
    }
    ++p_direntry;
  }
  return child_vinode;
}

struct vinode *rfs_create(struct vinode *parent, struct dentry *sub_dentry) {
  struct rfs_device *rdev = rfs_device_list[parent->sb->s_dev->dev_id];

  struct rfs_dinode *free_dinode = NULL;
  int free_inum = 0;
  for (int i = 0; i < (RFS_BLKSIZE / RFS_INODESIZE * RFS_MAX_INODE_BLKNUM);
       ++i) {
    free_dinode = rfs_read_dinode(rdev, i);
    if (free_dinode->type == R_FREE) {
      free_inum = i;
      break;
    }
    free_page(free_dinode);
  }

  if (free_dinode == NULL)
    panic("rfs_create: no more free disk inode, we cannot create file.\n" );

  free_dinode->size = 0;
  free_dinode->type = R_FILE;
  free_dinode->nlinks = 1;
  free_dinode->blocks = 0;

  free_dinode->addrs[0] = rfs_alloc_block(parent->sb);

  rfs_write_dinode(rdev, free_dinode, free_inum);
  free_page(free_dinode);

  struct vinode *new_vinode = rfs_alloc_vinode(parent->sb);
  new_vinode->inum = free_inum;
  rfs_update_vinode(new_vinode);

  int result = rfs_add_direntry(parent, sub_dentry->name, free_inum);
  if (result == -1) {
    return NULL;
  }

  return new_vinode;
}

int rfs_lseek(struct vinode *f_inode, ssize_t new_offset, int whence, int *offset) {
  int file_size = f_inode->size;

  switch (whence) {
    case LSEEK_SET:
      if (new_offset < 0 || new_offset > file_size) {
        return -1;
      }
      *offset = new_offset;
      break;
    case LSEEK_CUR:
      if (*offset + new_offset < 0 || *offset + new_offset > file_size) {
        return -1;
      }
      *offset += new_offset;
      break;
    default:
      return -1;
  }
  
  return 0;
}

int rfs_disk_stat(struct vinode *vinode, struct istat *istat) {
  struct rfs_device *rdev = rfs_device_list[vinode->sb->s_dev->dev_id];
  struct rfs_dinode *dinode = rfs_read_dinode(rdev, vinode->inum);
  if (dinode == NULL) {
    return -1;
  }

  istat->st_inum = vinode->inum;
  istat->st_size = dinode->size;
  istat->st_type = dinode->type;
  istat->st_nlinks = dinode->nlinks;
  istat->st_blocks = dinode->blocks;
  free_page(dinode);
  return 0;
}

struct super_block *rfs_get_superblock(struct device *dev) {
  struct rfs_device *rdev = rfs_device_list[dev->dev_id];

  if (rfs_r1block(rdev, RFS_BLK_OFFSET_SUPER) != 0)
    panic("RFS: failed to read superblock!\n");

  struct rfs_superblock d_sb;
  memcpy(&d_sb, rdev->iobuffer, sizeof(struct rfs_superblock));

  struct super_block *sb = alloc_page();
  sb->magic = d_sb.magic;
  sb->size = d_sb.size;
  sb->nblocks = d_sb.nblocks;
  sb->ninodes = d_sb.ninodes;
  sb->s_dev = dev;

  if( sb->magic != RFS_MAGIC ) 
    panic("rfs_get_superblock: wrong ramdisk device!\n");

  struct vinode *root_inode = rfs_alloc_vinode(sb);
  root_inode->inum = 0;
  rfs_update_vinode(root_inode);

  struct dentry *root_dentry = alloc_vfs_dentry("/", root_inode, NULL);
  sb->s_root = root_dentry;

  if (rfs_r1block(rdev, RFS_BLK_OFFSET_BITMAP) != 0)
    panic("RFS: failed to read bitmap!\n");
  void *bitmap = alloc_page();
  memcpy(bitmap, rdev->iobuffer, RFS_BLKSIZE);
  sb->s_fs_info = bitmap;

  return sb;
}
```

