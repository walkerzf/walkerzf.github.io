---
title: "6.S081 Pgtbl Lab"
categories:
  - 6.S081
tags:
  - Lab
toc: true # 是否启用内容索引
---
# Pgtbl Lab

In this lab , we will explore the user page tables and kernel page table ,and modify or create a process’s kernel page table to help simplify the functions that copy data from user space into the kernel space.

The lab have three parts. Part 1  is simpler relatively,we need to print the **valid** `pte` in three-level page table. Part 2 and 3 can be seen as one part .In part 2 ,we need to copy a process’s page table which is identical to kernel page table ,and in part 3 ,we need to add user mapping to the process’s kernel page table .

## Part 1 print a page table



```c
// Recursively free page-table pages.
// All leaf mappings must already have been removed.
void freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for (int i = 0; i < 512; i++)
  {
    pte_t pte = pagetable[i];
    if ((pte & PTE_V) && (pte & (PTE_R | PTE_W | PTE_X)) == 0)
    {
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    }
    else if (pte & PTE_V)
    {
      panic("freewalk: leaf");
    }
  }
  kfree((void *)pagetable);
}
```

