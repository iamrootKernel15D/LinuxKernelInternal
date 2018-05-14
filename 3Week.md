# Direct Mapping이 무엇인가?
## 커널 크기가 1GB로 맵핑되는데, 그럼 1GB이하의 컴퓨터에서 커널을 어떻게 맵핑되는가?

결론 부터 말하자면 Direct Mapping 되는 영역은 물리 메모리 크기에 따라 달라질수 있다.

kernel 4.16.8 기준으로 코드를 보아 보면 아래와 같다.
arch/x86/mm/init.c 파일에 존재 하는 void \__init zone_sizes_init(void) 함수에서  zone 의 크기를 정의한다.

```c
void __init zone_sizes_init(void)
{
...
    max_zone_pfns[ZONE_NORMAL]  = max_low_pfn;
...
}
```
ZONE_NORMAL 의 경우 max_low_pfn 으로 정의하고 있다.

그럼 max_low_pfn 값은 어디서 설정되는가?
32비트 기준으로 arch/x86/mm/init_32.c 파일에 존재하는 void \__init find_low_pfn_range(void) 함수를 보면 된다.

```c
/*
 * Determine low and high memory ranges:
 */
void __init find_low_pfn_range(void)
{
    /* it could update max_pfn */
   if (max_pfn <= MAXMEM_PFN)
        lowmem_pfn_init();
    else
        highmem_pfn_init();
}
```

물리메모리가 작을때는 물리 메모리가 ㅈstatic void \__init lowmem_pfn_init(void) 함수를 보면 된다.

```c
/*
 * All of RAM fits into lowmem - but if user wants highmem
 * artificially via the highmem=x boot parameter then create
 * it:
 */
static void __init lowmem_pfn_init(void)
{
    /* max_low_pfn is 0, we already have early_res support */
    max_low_pfn = max_pfn;

    if (highmem_pages == -1)
        highmem_pages = 0;
#ifdef CONFIG_HIGHMEM
    if (highmem_pages >= max_pfn) {
        printk(KERN_ERR MSG_HIGHMEM_TOO_BIG,
            pages_to_mb(highmem_pages), pages_to_mb(max_pfn));
        highmem_pages = 0;
    }
    if (highmem_pages) {
        if (max_low_pfn - highmem_pages < 64*1024*1024/PAGE_SIZE) {
            printk(KERN_ERR MSG_LOWMEM_TOO_SMALL,
                pages_to_mb(highmem_pages));
            highmem_pages = 0;
        }
        
        // 위에서 highmem_pages 의 크기를 결정한 이후에 max_low_pfn 값을 결정한다.
       max_low_pfn -= highmem_pages;
    }
#else
    if (highmem_pages)
        printk(KERN_ERR "ignoring highmem size on non-highmem kernel!\n");
#endif
}

```

그럼 물리메모리가 클 경우 static void \__init highmem_pfn_init(void) 함수를 보면 된다.

```c
static void __init highmem_pfn_init(void)
{
    max_low_pfn = MAXMEM_PFN;
    ...
}

```

___다시 말해서 max_low_pfn 값은 물리 메모리크기에 따라 변할수 있는 값이며 이 값은 ZONE_NORMAL 의 최대 크기가 된다.___

추가로 더 확인하면 좋은것들은 아래와 같다.

arch/x86/include/asm/setup.h
```c
#define MAXMEM_PFN	PFN_DOWN(MAXMEM)
```
include/linux/pfn.h
```c
#define PFN_DOWN(x)	((x) >> PAGE_SHIFT)
```
arch/x86/include/asm/pgtable_32_types.h
```c
#define MAXMEM	(VMALLOC_END - PAGE_OFFSET - \__VMALLOC_RESERVE)
```
arch/x86/include/asm/pgtable_64_types.h
```
c#define MAXMEM			_AC(\__AC(1, UL) << MAX_PHYSMEM_BITS, UL)
```
arch/x86/include/asm/pgtable_32_types.h
```c
/*
 * Define this here and validate with BUILD_BUG_ON() in pgtable_32.c
 * to avoid include recursion hell
 */
#define CPU_ENTRY_AREA_PAGES    (NR_CPUS * 40)

#define CPU_ENTRY_AREA_BASE                     \
    ((FIXADDR_TOT_START - PAGE_SIZE * (CPU_ENTRY_AREA_PAGES + 1))   \
     & PMD_MASK)

#define PKMAP_BASE      \
    ((CPU_ENTRY_AREA_BASE - PAGE_SIZE) & PMD_MASK)

#ifdef CONFIG_HIGHMEM
# define VMALLOC_END    (PKMAP_BASE - 2 * PAGE_SIZE)
#else
# define VMALLOC_END    (CPU_ENTRY_AREA_BASE - 2 * PAGE_SIZE)
#endif
```
arch/x86/include/asm/fixmap.h
```c
#define FIXADDR_TOT_START	(FIXADDR_TOP - FIXADDR_TOT_SIZE)
/*
 * We can't declare FIXADDR_TOP as variable for x86_64 because vsyscall
 * uses fixmaps that relies on FIXADDR_TOP for proper address calculation.
 * Because of this, FIXADDR_TOP x86 integration was left as later work.
 */
#ifdef CONFIG_X86_32
/* used by vmalloc.c, vsyscall.lds.S.
 *
 * Leave one empty page between vmalloc'ed areas and
 * the start of the fixmap.
 */
extern unsigned long __FIXADDR_TOP;
#define FIXADDR_TOP ((unsigned long)__FIXADDR_TOP)
#else
#define FIXADDR_TOP (round_up(VSYSCALL_ADDR + PAGE_SIZE, 1<<PMD_SHIFT) - \
             PAGE_SIZE)
#endif
```
 
# Slob은 정확히 어디서 사용하는가?
 
# FAT, Index, Chain의 명확한 특징은?
 
# block bitmap은 한 블록에 존재해야 하므로 한 블록의 크기가 4KB라면 최대 몇개의 block그룹을 설정하는가?
## 이때, 4096 * 8인데 8은 어디서 나온 것 인가?
 
#가용성을 위해 ext3에는 LVM을 추가했다고 하였다. 이때 가용성은 무엇을 뜻하는가?
 
# C언어에서 try ~ catch와 같은 기능이 존재하는가?
## 없다면 비슷하게 구현가능한가?
 
# eip가 무엇인가?
 
