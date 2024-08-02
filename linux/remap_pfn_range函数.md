# remap_pfn_range函数分析

## riscv版

mmap:

```c
#define pgprot_noncached pgprot_noncached
static inline pgprot_t pgprot_noncached(pgprot_t _prot)
{
	unsigned long prot = pgprot_val(_prot);

	prot &= ~_PAGE_MTMASK;
	prot |= _PAGE_IO;

	return __pgprot(prot);
}
```

memory.c：remap_pfn_range，实际调用了remap_pfn_range_notrack。

remap_pfn_range_notrack：

```
vma->vm_flags |= VM_IO | VM_PFNMAP | VM_DONTEXPAND | VM_DONTDUMP;
```

设置vm_flags，其中：

VM_IO：

--》

remap_p4d_range-->

remap_pud_range-->

remap_pmd_range-->

remap_pte_range-->

set_pte_at-->

__set_pte_at-->

```
#define _PAGE_PRESENT   (1 << 0)
#define _PAGE_GLOBAL    (1 << 5)    /* Global */

```



```c
static inline void __set_pte_at(struct mm_struct *mm,
	unsigned long addr, pte_t *ptep, pte_t pteval)
{
	if (pte_present(pteval) && pte_exec(pteval))
		flush_icache_pte(pteval); // 会执行这行代码

	set_pte(ptep, pteval);
}

void flush_icache_pte(pte_t pte)
{
	struct page *page = pte_page(pte);

	/*
	 * HugeTLB pages are always fully mapped, so only setting head page's
	 * PG_dcache_clean flag is enough.
	 */
	if (PageHuge(page))
		page = compound_head(page);

	if (!test_bit(PG_dcache_clean, &page->flags)) {
		flush_icache_all();
		set_bit(PG_dcache_clean, &page->flags);
	}
}


/*
 * PageHuge() only returns true for hugetlbfs pages, but not for normal or
 * transparent huge pages.  See the PageTransHuge() documentation for more
 * details.
 */
int PageHuge(struct page *page)
{
	if (!PageCompound(page))
		return 0;

	page = compound_head(page);
	return page[1].compound_dtor == HUGETLB_PAGE_DTOR;
}
```



## aarch64版

remap_pfn_range-->

remap_p4d_range-->

remap_pud_range-->

remap_pmd_range-->

remap_pte_range-->

set_pte_at-->

```c
set_pte_at(mm, addr, pte, pte_mkspecial(pfn_pte(pfn, prot)));--》
static inline void set_pte_at(struct mm_struct *mm, unsigned long addr,
			      pte_t *ptep, pte_t pte)
{
	if (pte_present(pte) && pte_user_exec(pte) && !pte_special(pte))
		__sync_icache_dcache(pte); // 不会执行，验证过了

	__check_racy_pte_update(mm, ptep, pte);

	set_pte(ptep, pte);
}
```

很奇怪，不知道是哪里对缓存进行了刷新。。。
