---
title: "6.S081 Pgtbl Lab"
categories:
  - 6.S081
tags:
  - Lab
toc: true # 是否启用内容索引
---
# Pgtbl Lab

  In this lab , we will explore the user page tables and kernel page table ,and modify or create a process‘s kernel page table to help simplify the functions that copy data from user space into the kernel space.

  The lab have three parts. Part 1  is simpler relatively,we need to print the **valid** `pte` in three-level page table. Part 2 and 3 can be seen as one part .In part 2 ,we need to copy a process’s page table which is identical to kernel page table ,and in part 3 ,we need to add user mapping to the process’s kernel page table .

## Part 1 print a page table

  In this part ,we need to print the first process ‘ page table ,so we can see the figure 1 ,which is the use address space .The user address space have several parts : `text`,`data` ,`guard page` , `stack`,`trampoline` and `trapframe` .

  From the function `freewalk` .which is recursively free page-table pages ,we can find that we need to recursively print the page table. And we have three-level depth page table , so we need a variable to record the depth of the recursion in the helper function.

```c
// Recursively free page-table pages.
// All leaf mappings must already have been removed.
void freewalk(pagetable_t pagetable){
  // there are 2^9 = 512 PTEs in a page table.
  for (int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if ((pte & PTE_V) && (pte & (PTE_R | PTE_W | PTE_X)) == 0){
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    }
    else if (pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void *)pagetable);
}
```

  We can have the function like these.

```c
//heplerfunction for vmprint
void helpervmprint(pagetable_t pagetable, int level){
  if (level > 2)
    return;
  for (int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if ((pte & PTE_V)){
      //this pte pointer to a lower-level page table
      uint64 child = PTE2PA(pte);
      for (int j = 0; j <= level; j++){
        printf("..");
        if (j != level)
          printf(" ");
      }
      printf("%d: pte %p pa %p\n", i, pte, child);
      helpervmprint((pagetable_t)child, level + 1);
    }
  }
}
//function to help print the contents of a page table
void vmprint(pagetable_t pagetable){
  printf("page table %p\n", pagetable);
  helpervmprint(pagetable, 0);
}
```

  We can get the output like this. In the top-level page table ,we have two entry. The first entry corresponding 1GB address. In the bottom page table ,we have three entries  ,corresponding the `text and data` ,`guard page` and `stack`.  The second entry in top-level page table ,  corresponding the `trampoline` and ` tramframe` . We have two interesting points.

* The reason about the `text and data` are mapped together not respectively only for simplicity.
* The `trampoline`  and `tramframe` are mapped highest in va , but in page table ,they are in entry `255`,not in `511`,because  although the `riscv`  uses 39-bits,it actually uses 38 bits ( because  if we use the 39th bit ,the 40th 41th 42th ..etc should be set **All for simplicity! But still support for the future!**) , so in the top-level page table ,we only have 8 bits, the highest bit is `255`.

> ```
> page table 0x0000000087f6e000
> ..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
> .. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
> .. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
> .. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
> .. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
> ..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
> .. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
> .. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
> .. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
> ```

We actually can print the `PTE_FLAG` for each valid pte.

## Part 2  A kernel page table per process 

  The lab is vert time-consuming ,because we are doing kernel programming  which is difficult to track the bug  and easy to make error. The points we should pay attention to :

* The `xv6` code is specialized for  kernel page table 
* the kernel page table  init  `kvminit`  happens in `pricinit ` and `virto_disc`.
* the memory free-up

In part 3 , we will add user mapping in process's kernel page table  to allow the kernel to dereference  the user pointer. This scheme rely on the **user virtual memory range not overlapping the range of the `va` that the kernel address uses for its own instructions and data .**
       Xv6 uses virtual addresses that start at zero for user address spaces, and luckily the kernel's memory starts at higher addresses. **However, this scheme does limit the maximum size of a user process to be less than the kernel's lowest virtual address.** After the kernel has booted, that address is 0xC000000 in xv6, the address of the `PLIC` registers .You'll need to modify xv6 to prevent user processes from growing larger than the PLIC address.
        Through the calculating , the` kernel base` is in  entry 2 .And the `PLIC` is in entry zero ,anything above the `PLIC`.So any user mapping below the ` PLIC ` will be added into process's kernel page table .And anything above that will be identical to the kernel page table.
         So the entry 1-511 we can share the same page table with the kernel page table . The entry zero should be unique in process's kernel page table. This is called `share` solution. 
        We could have the naive  `copy` solution , when we switch in  process , we copy the ,when we switch out ,we free up the page table .

### The `init` of process kernel page table

```c
//share the 1-511 entry and map the entry zero  for process 's kernel page table init

```



### The clean of process kernel page table

When we free the  process’s kernel page table ,because entry 1-511 we share the same thing with the kernel page table ,so we do not need to free that memory . So we do free the lower level page table corresponding the entry zero. We can get `medium level` page table through the `pagetable[0]` ,  and free any valid pte  and corresponding `bottom level` page table .

```

```







**The course lab site** :[MIT 6.S081 pgtbl](https://pdos.csail.mit.edu/6.828/2020/labs/pgtbl.html)