---
title: "Golang Memory Management (based on 1.12.5)"
slug: "golang-memory-management"
date: 2019-07-02T00:03:00+08:00
draft: false
image: "/content/images/2019/07/DSC05572.jpg"
tags: ["golang"]
---

# Golang Memory Management (based on 1.12.5)
此文是我在看過許多 Golang 記憶體管理的文章後，再搭配 souce code 所整理出來的。有點像是利用 souce code 來驗證我是否理解網路上所搜尋到的文章。  
Golang 基於 [TCMalloc (Thread-Caching Malloc)](http://goog-perftools.sourceforge.net/doc/tcmalloc.html) 加以修改，改為適合自己的 allocator。  
TCMalloc 有幾項特點：  

- 盡可能先分配一大塊記憶體，以降低分配及釋放的 overhead
- 將這塊記憶體切成各種大小的 span，以降低 memory fragmentation
- 分配特別小的物件時優化，以提升記憶體使用率
- Thread 裡有 cache，使用時可以不必 lock 以提升效率

# Terminology
Golang memory management 裡有許多角色，此段落先簡單介紹各角色功用，下段落會詳細各角色的結構及彼此之間的關係。  
下圖為大致架構圖，可一邊閱讀一邊參閱：  

![](/content/images/2019/07/golang_memory_architecture.png)

## mheap
`mheap` 在整個 Golang runtime 裡只會存在一個，Golang 所用到的所有記憶體都要來向 `mheap` 申請。  
而 `mheap` 自身擁有的記憶體不夠時，會再向 OS 申請，每次申請至少 1MB 的記憶體，減少向 OS 申請的 overhead。  
`mheap` 在管理記憶體時是以 page 為單位在管理的，這裡的 page 並不是 OS 層級的 page。Golang 的一頁 page 大小為 8k bytes。

## arena
`arena` 便是 Golang 的 heap，一個 `arena` 大小為 64MB (不同平台、架構有所不同，此文皆以 Linux-64bit 為主)。heap 裡有許多個 `arena`，數量會隨著需求而慢慢增加或減少。  

## heapArena
每一個 `arena` 都會有一相對應的 `heapArena`，這裡存放了一些 `arena` 的 metadata，像是 `bitmap` 或是 `spans`。  
在 `mheap` 裡的 `heapArena` 是一個二維 array，在大多數架構下，第一層通常只有一個 element。  
`heapArena` 並不存在 `arena` 裡，所以稱他 off-heap。  

## spans
`spans` 紀錄 `arena` 裡每一個 page 是屬於哪一個 `mspan` 的。

## bitmap
`bitmap` 的存在是為了幫助 GC (garbage collect)，讓 GC 知道哪些是可以回收、哪些不行，亦提供該 object 是不是 pointer，GC 才能知道要不要繼續 scan。  
此文不會對 Golang GC 有太多著墨，預計會在下一篇文章講解 GC。

## mcentral
`mcentral` 用來管理 `mspan`，一個 `mcentral` 負責一種 size class 的 `mspan`，而目前總共有 67 種 sizes。  
當 `mcentral` 無 `mspan` 可用時，便會向 `mheap` 申請新的 `mspan` 加入。
`mheap` 裡總共有 134 (67*2) 個 `mcentral`，這是因為每一個 `mcentral` 又分為 scan 及 non-scan 兩種，前者表示 `mspan` 裡有存放 pointer，後者表示有存放 pointer。  
這是為了加速 GC 的 mark 階段，在 non-scan 裡，一定不會有 pointer，所以不需一個一個往下檢查。

## mspan
`mspan` 由數頁 page 所組成，每一個 `mspan` 都會對應到一個 size class，用來決定該如何切割該 `mspan` 裡的 page。其 `mspan` 是 double-linked list。  
page 經過切割後變為 object，此 object 才是真正分配記憶體的最小單位。

## mcache
每個 P (Golang scheduler 的 logic processor)，會綁定一個 `mcache`，由於同一時間只會有一個 G (goroutine) 在 P 上跑，所以在存取時並不需要 lock。  
當 `mcache` 上的 `mspan` 已經沒有空間了，便會向 `mcentral` 申請一個有空間的 `mspan` 來替換舊的 `mspan`。

## span class
目前共有 67 種 size classes，每種 class 訂出了其 span 多大、其 object 多大。  
而每一種 size 又會分為 scan 及 noscan 兩種，所以共有 137 種 span class。  
每個 `mcentral` 負責一種 span class，其包含的所有 `mspan` 將會按照其 size 劃分。  

# Component
## Virtual Memory Layout
Golang 的 virtual memory layout 長相如下：  

![](/content/images/2019/07/golang_memory_layout.png)

其中 `spans` 及 `bitmap` 功用如上段落所述。  
`heapArena` 會包含整個可以用的 address space，而 allocator 會盡量讓整組 `arena` 是連續的，這是為了讓較大的 `mspan` (或是 object) 能跨 `arena`。

## mheap
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/mheap.go#L31)  

下列是 `mheap` 結構，僅列出此文會講到的幾個 fields：  
{{< highlight go>}}
// mheap.go
type mheap struct {
	// mheap 只會有一個，當同時有多個角色來向 mheap 要求記憶體時，便需要 lock
	lock      mutex

	// 一個 mTreap node 只會存放一個 span
	// free 表示是乾淨的 span，而 scav 表示被汙染的 span
	// 被汙染的意思是其中有 page 歸還給 OS 了
	free      mTreap // free and non-scavenged spans
	scav      mTreap // free and scavenged spans

    // allspans 會存放所有產生出來的 mspan
	allspans []*mspan // all spans out there

    // arenas 是個二維 slice，其型態是 heapArena，裡面存放關於 arena 的 metadata
    // 每一個 arena 會有一個相對應的 heapArena
    //
    // 要 access 時，要用 arenaIndex 來計算 index
    // 不在 Golang heap 的 address space，其對應的 heapArena 會是 nil
    // 
    // 大多數情況下，第一層 (L1) 通常只會有一個，而會有多個第二層 (L2)
	arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

	// allArena 可以用來遍歷整個 address space
	allArenas []arenaIdx

	// central 是長度 137 (67*2) 的 array，其中 index 為 spanClass
	// 各 central 管理各自 size 的 mspan
	central [numSpanClasses]struct {
		mcentral mcentral
		pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
	}

	// 下列為各物件的 allocator，在 mheap.init() 會呼叫各 allocator 的 init
	// 在需要時，會再呼叫各 allocator 的 alloc()
	spanalloc             fixalloc // allocator for span*
	cachealloc            fixalloc // allocator for mcache*
	treapalloc            fixalloc // allocator for treapNodes*
	specialfinalizeralloc fixalloc // allocator for specialfinalizer*
	specialprofilealloc   fixalloc // allocator for specialprofile*
	speciallock           mutex    // lock for special record allocators.
	arenaHintAlloc        fixalloc // allocator for arenaHints
}

type heapArena struct {
	// bitmap 裡每 2 個 bits 會對應到 arena 裡一個 pointer 大小的記憶體
	// 也就是一個 byte 會對應到 4 個 pointer 大小的記憶體
	// 每個 2-bit entry 裡，0 代表不需理會，1 代表活的 pointer
	bitmap [heapArenaBitmapBytes]byte

	// spans 將 page ID 對應到 arena 裡的 mspan
	// 對於已經分配的 mspan，其 page 都會指向該 mspan
	// 對於還未分配的 mspan，只有第一個及最後一個 page 會指向該 mspan，中間的 page 會指向任意的 mspan
	// 還沒被分配的 page 則會對應到 nil
	spans [pagesPerArena]*mspan
}

{{< /highlight >}}

## mspan
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/mheap.go#L316)  

下列是 `mspan` 結構，僅列出此文會講到的幾個 fields：  
{{< highlight go>}}
// mheap.go
type mspan struct {
	// 指向下一個 mspan 的 pointer (在 mcentral 裡，mspan 是以 double-linked list 存放)
	next *mspan     // next span in list, or nil if none

	// 指向前一個 mspan 的 pointer
	prev *mspan     // previous span in list, or nil if none
	list *mSpanList // For debugging. TODO: Remove.

	// 該 mspan 第一個 byte 的起始位址，即是在 arena 的起始位址
	startAddr uintptr // address of first byte of span aka s.base()

	// 該 mspan 擁有多少 page
	npages    uintptr // number of pages in span

	// freeindex 用來指出該 mspan 要從何處開始掃描找出下一個可用的 object，其 index 為 0 至 nelems
	// 每次 allocation 時，會從 freeindex 開始掃描，遇到 0 代表該 object 是可用的
	// freeindex 會被調整到該位置，這樣下次掃描便會從此次分配的 object 的後面開始掃描
    // 若 freeindex 等於 nelem，則代表該 mspan 已經沒有可用的 objects 了
    // freeindex 前的 object 都是已分配的，freeindex 之後的 object 可能分配了，也可能還未分配
    freeindex uintptr
    
    // allocCache 為 allocBits 的 cache，並且從 freeindex 開始
    // allocCache 裡面存了 allocBits 的補數，因此可以直接計算 allocCache 最後面有幾個 0 (也就是在第幾個 bit 遇到第一個 1) 來使用
    // (我對 allocCache 還未了解透徹，目前的認知是，allocCache 涵蓋的 mspan 範圍只有一部份，但存取速度較快，
    //  可以快速找出在此範圍內可用的 object。若找不到，便得乖乖找出該 mspan 裡可用的 object 位置)
    allocCache uint64

    // allocBits 用來標記 object 是否已被分配，其內容是指向 mspan's mark 的 pointer
    allocBits  *gcBits
	
	// 該 mspan 擁有多少 object
	nelems uintptr // number of object in the span.

	// 已經分配出去的 object 數量
	allocCount  uint16     // number of allocated objects

	// 該 mspan 的 span class
	spanclass   spanClass  // size class and noscan (uint8)
}
{{< /highlight >}}

## mcentral
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/mcentral.go#L20)  

下列是 `mcentral` 結構，僅列出此文會講到的幾個 fields：  
{{< highlight go>}}
// mcentral.go
type mcentral struct {
	// 每一種 span class 只會有一個 mcentral，當有多個 mcache 向 mcentral 申請新的 mspan 時，便需要 lock
	lock      mutex

	// 該 mcentral 屬於的 span class
	spanclass spanClass

	// 還有可分配 object 的 mspan list
	nonempty  mSpanList // list of spans with a free object, ie a nonempty free list

	// 已經沒有可分配 object 的 mspan list
	empty     mSpanList // list of spans with no free objects (or cached in an mcache)
}
{{< /highlight >}}

## mcache
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/mcache.go#L19)  

下列是 `mcache` 結構，僅列出此文會講到的幾個 fields：  
{{< highlight go>}}
// mcache.go
type mcache struct {
	// tiny 指向目前的 tiny block，若沒有則為 nil
	tiny             uintptr

	// tinyoffset 為此 tiny block 分配到哪裡了
	tinyoffset       uintptr

	// alloc 為 mspan array，其 index 為 span class，需要存取時，便取出相對應的 mspan
	alloc [numSpanClasses]*mspan // spans to allocate from, indexed by spanClass
}
{{< /highlight >}}

## size class
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/sizeclasses.go#L6)

下表為 Golang 所有的 size class，下列解釋各 column 意義：  

- class：此 class 的編號，分類時皆使用此編號作為 index
- bytes/obj：此 class 一個 object 的大小
- bytes/span：此 class 一個 mspan 的大小
- objects：此 class 的 object 數量
- tail waste：此 class 因無法完整分配所造成的浪費，例如 class 4，浪費了 32 bytes = 8192 - 48*170
- max waste：此 class 最多可能浪費的空間，計算公式為：((object size - (前一級的 object size + 1)) * object 數量 + tail waste) / mspan size，例如 class 4 為 31.52% = ((48 - (32 + 1)) * 170 + 32) / 8192

{{< highlight go>}}
// sizeclasses.go
// class  bytes/obj  bytes/span  objects  tail waste  max waste
//     1          8        8192     1024           0     87.50%
//     2         16        8192      512           0     43.75%
//     3         32        8192      256           0     46.88%
//     4         48        8192      170          32     31.52%
//     5         64        8192      128           0     23.44%
//     6         80        8192      102          32     19.07%
//     7         96        8192       85          32     15.95%
//     8        112        8192       73          16     13.56%
//     9        128        8192       64           0     11.72%
//    10        144        8192       56         128     11.82%
//    11        160        8192       51          32      9.73%
//    12        176        8192       46          96      9.59%
//    13        192        8192       42         128      9.25%
//    14        208        8192       39          80      8.12%
//    15        224        8192       36         128      8.15%
//    16        240        8192       34          32      6.62%
//    17        256        8192       32           0      5.86%
//    18        288        8192       28         128     12.16%
//    19        320        8192       25         192     11.80%
//    20        352        8192       23          96      9.88%
//    21        384        8192       21         128      9.51%
//    22        416        8192       19         288     10.71%
//    23        448        8192       18         128      8.37%
//    24        480        8192       17          32      6.82%
//    25        512        8192       16           0      6.05%
//    26        576        8192       14         128     12.33%
//    27        640        8192       12         512     15.48%
//    28        704        8192       11         448     13.93%
//    29        768        8192       10         512     13.94%
//    30        896        8192        9         128     15.52%
//    31       1024        8192        8           0     12.40%
//    32       1152        8192        7         128     12.41%
//    33       1280        8192        6         512     15.55%
//    34       1408       16384       11         896     14.00%
//    35       1536        8192        5         512     14.00%
//    36       1792       16384        9         256     15.57%
//    37       2048        8192        4           0     12.45%
//    38       2304       16384        7         256     12.46%
//    39       2688        8192        3         128     15.59%
//    40       3072       24576        8           0     12.47%
//    41       3200       16384        5         384      6.22%
//    42       3456       24576        7         384      8.83%
//    43       4096        8192        2           0     15.60%
//    44       4864       24576        5         256     16.65%
//    45       5376       16384        3         256     10.92%
//    46       6144       24576        4           0     12.48%
//    47       6528       32768        5         128      6.23%
//    48       6784       40960        6         256      4.36%
//    49       6912       49152        7         768      3.37%
//    50       8192        8192        1           0     15.61%
//    51       9472       57344        6         512     14.28%
//    52       9728       49152        5         512      3.64%
//    53      10240       40960        4           0      4.99%
//    54      10880       32768        3         128      6.24%
//    55      12288       24576        2           0     11.45%
//    56      13568       40960        3         256      9.99%
//    57      14336       57344        4           0      5.35%
//    58      16384       16384        1           0     12.49%
//    59      18432       73728        4           0     11.11%
//    60      19072       57344        3         128      3.57%
//    61      20480       40960        2           0      6.87%
//    62      21760       65536        3         256      6.25%
//    63      24576       24576        1           0     11.45%
//    64      27264       81920        3         128     10.00%
//    65      28672       57344        2           0      4.91%
//    66      32768       32768        1           0     12.50%
{{< /highlight >}}

# Allocating Flow
Golang 將記憶體分層存取，也依 size 分級存取。對於特別大及特別小的亦有另外的處理方式。  
依據 size 及是否為 pointer 分為三種 allocating 流程：

- Small size：`<= 32KB` 的 object
- Tiny size：`< 16 bytes` 且不為 pointer 的 object
- Large size：`> 32KB` 的 object

## Small Size
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/malloc.go#L928)

Small size 分配流程大致如下：  

1. 根據需求確定其 size class 及 span class
2. 試著從 `mcache` 找可用的 object，有的話便直接分配
3. 若沒有則向 `mcentral` 要一個 `mspan`
4. 若 `mcentral` 也沒有，則會再向 `mheap` 申請

上述行為到第 2 步為止都不需 lock，第 3、4 步都需要 lock，分別 lock 住 `mcentral` 及 `mheap`。

下列是 small size allocating 的部份內容：
{{< highlight go>}}
// malloc.go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	// 處理 GC 相關事情

	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			// tiny size 分配流程
		} else {
			// 根據要分配的 size 找出合適的 size class
			var sizeclass uint8
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
			} else {
				sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
			}

			// 將 size round up 為此 class 的 object size
			size = uintptr(class_to_size[sizeclass])

			// 根據 size class 及是否能 scan 決定 span class
			spc := makeSpanClass(sizeclass, noscan)

			// 根據 span class 取出 mcache 裡的 mspan
			span := c.alloc[spc]

			// 利用 mspan.allocCache 試著找出可用的 object
			v := nextFreeFast(span)
			if v == 0 {
				// 找不到的話，便試著從整個 mspan 找出可用的 object
				// 若沒有的話，則向 mcentral 要一個新的 mspan 並替換舊的 mspan
				v, span, shouldhelpgc = c.nextFree(spc)
			}
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(unsafe.Pointer(v), size)
			}
		}
	} else {
		// large size 分配流程
	}

	// 後續處理

	return x
}

// nextFreeFast returns the next free object if one is quickly available.
// Otherwise it returns 0.
func nextFreeFast(s *mspan) gclinkptr {
	theBit := sys.Ctz64(s.allocCache) // Is there a free object in the allocCache?
	if theBit < 64 {
		result := s.freeindex + uintptr(theBit)
		if result < s.nelems {
			freeidx := result + 1
			if freeidx%64 == 0 && freeidx != s.nelems {
				return 0
			}
			s.allocCache >>= uint(theBit + 1)
			s.freeindex = freeidx
			s.allocCount++
			return gclinkptr(result*s.elemsize + s.base())
		}
	}
	return 0
}

// nextFree returns the next free object from the cached span if one is available.
// Otherwise it refills the cache with a span with an available object and
// returns that object along with a flag indicating that this was a heavy
// weight allocation. If it is a heavy weight allocation the caller must
// determine whether a new GC cycle needs to be started or if the GC is active
// whether this goroutine needs to assist the GC.
//
// Must run in a non-preemptible context since otherwise the owner of
// c could change.
func (c *mcache) nextFree(spc spanClass) (v gclinkptr, s *mspan, shouldhelpgc bool) {
	s = c.alloc[spc]
	shouldhelpgc = false
	freeIndex := s.nextFreeIndex()
	if freeIndex == s.nelems {
		// 這個 mspan 已經沒有可用的 object 了
		if uintptr(s.allocCount) != s.nelems {
			println("runtime: s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
			throw("s.allocCount != s.nelems && freeIndex == s.nelems")
        }
        
        // 向 mcentral 申請一個新的 mspan 來替換舊的
		c.refill(spc)
		shouldhelpgc = true
		s = c.alloc[spc]

		freeIndex = s.nextFreeIndex()
	}

	if freeIndex >= s.nelems {
		throw("freeIndex is not valid")
	}

	v = gclinkptr(freeIndex*s.elemsize + s.base())
	s.allocCount++
	if uintptr(s.allocCount) > s.nelems {
		println("s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
		throw("s.allocCount > s.nelems")
	}
	return
}
{{< /highlight >}}
{{< highlight go>}}
// mcache.go
func (c *mcache) refill(spc spanClass) {
	// 取出目前舊的 mspan 並歸還給 mcentral
	s := c.alloc[spc]

	if uintptr(s.allocCount) != s.nelems {
		throw("refill of span with free space remaining")
	}
	if s != &emptymspan {
		// Mark this span as no longer cached.
		if s.sweepgen != mheap_.sweepgen+3 {
			throw("bad sweepgen in refill")
		}
		atomic.Store(&s.sweepgen, mheap_.sweepgen)
	}

	// 從 mcentral 再拿一個新的 mspan
	s = mheap_.central[spc].mcentral.cacheSpan()
	if s == nil {
		throw("out of memory")
	}

	if uintptr(s.allocCount) == s.nelems {
		throw("span has no free space")
	}

	// Indicate that this span is cached and prevent asynchronous
	// sweeping in the next sweep phase.
	s.sweepgen = mheap_.sweepgen + 3

    // 將新的 mspan 放到 mcache 裡
	c.alloc[spc] = s
}
{{< /highlight >}}
{{< highlight go>}}
// mcentral.go
func (c *mcentral) cacheSpan() *mspan {
	// Deduct credit for this span allocation and sweep if necessary.
	spanBytes := uintptr(class_to_allocnpages[c.spanclass.sizeclass()]) * _PageSize
	deductSweepCredit(spanBytes, 0)

	lock(&c.lock)
	traceDone := false
	if trace.enabled {
		traceGCSweepStart()
	}
	sg := mheap_.sweepgen
retry:
	var s *mspan
    // 試著在 nonempty mspan linked-list (還有空間的 mspan) 找出可用的 mspan
	for s = c.nonempty.first; s != nil; s = s.next {
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			c.nonempty.remove(s)
			c.empty.insertBack(s)
			unlock(&c.lock)
			s.sweep(true)
			goto havespan
		}
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// we have a nonempty span that does not require sweeping, allocate from it
		c.nonempty.remove(s)
		c.empty.insertBack(s)
		unlock(&c.lock)
		goto havespan
	}

    // 試著在 empty mspan linked-list (已經沒有空間的 mspan) 找出可用的 mspan
	for s = c.empty.first; s != nil; s = s.next {
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			// 有一個需要 sweep 的 mspan，sweep 該 mspan 再看看是否有可用的 object
			c.empty.remove(s)
            
            // swept spans are at the end of the list
			c.empty.insertBack(s)
            unlock(&c.lock)
            
            // sweep 該 mspan
			s.sweep(true)
			freeIndex := s.nextFreeIndex()
			if freeIndex != s.nelems {
                // 找到可用的 mspan，調整 freeindex 後 goto havespan 段落
				s.freeindex = freeIndex
				goto havespan
			}
			lock(&c.lock)
			// the span is still empty after sweep
			// it is already in the empty list, so just retry
			goto retry
		}
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// already swept empty span,
		// all subsequent ones must also be either swept or in process of sweeping
		break
	}
	if trace.enabled {
		traceGCSweepDone()
		traceDone = true
	}
	unlock(&c.lock)

    // 若從 nonempty 及 empty 的 mspan linked-list 都找不到可用的 mspan，
    // 表示該 mcentral 需要再向 mheap 申請 mspan
	s = c.grow()
	if s == nil {
		return nil
	}
	lock(&c.lock)
	c.empty.insertBack(s)
	unlock(&c.lock)

	// At this point s is a non-empty span, queued at the end of the empty list,
	// c is unlocked.
havespan:
	// 找到 mspan 了，做一些後續處理
	return s
}

func (c *mcentral) grow() *mspan {
	npages := uintptr(class_to_allocnpages[c.spanclass.sizeclass()])
	size := uintptr(class_to_size[c.spanclass.sizeclass()])
	n := (npages << _PageShift) / size

    // 向 mheap 申請 mspan 來使用
	s := mheap_.alloc(npages, c.spanclass, false, true)
	if s == nil {
		return nil
	}

	p := s.base()
	s.limit = p + size*n

	heapBitsForAddr(s.base()).initSpan(s)
	return s
}
{{< /highlight >}}

## Tiny Size
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/malloc.go#L864)

Tiny size 分配流程大致如下：  

1. 將分配大小對齊在 2、4、8 上
2. 若該 `mcache` 有 tiny block 且有合適的空間，便直接分配
3. 若沒有或是空間不合適，則在該 `mcache` 裡尋找其他 `mspan`
4. 若沒有的話，則向 `mcentral` 申請新的 `mspan`
5. 若 `mcentral` 也沒有，則會再向 `mheap` 申請
6. 視情況調整 `mcache` 的 tiny block 及 tinyoffset

上述行為到第 3 步為止都不需 lock，第 4、5 步都需要 lock，分別 lock 住 `mcentral` 及 `mheap`。

下列是 tiny size allocating 的 部份內容：
{{< highlight go>}}
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	// 處理 GC 相關事情
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			off := c.tinyoffset
			// 根據要分配的 size 對齊到 2、4、8
			if size&7 == 0 {
				off = round(off, 8)
			} else if size&3 == 0 {
				off = round(off, 4)
			} else if size&1 == 0 {
				off = round(off, 2)
			}

			// 檢查此 mcache 是否有 tiny block
			// 若有且放的下的話，便放進此 tiny block 並調整 offset
			if off+size <= maxTinySize && c.tiny != 0 {
				x = unsafe.Pointer(c.tiny + off)
				c.tinyoffset = off + size
				c.local_tinyallocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}

			// 從 mcache 拿出新的 mspan
			span := c.alloc[tinySpanClass]

			// 利用 mspan.allocCache 試著找出可用的 object
			v := nextFreeFast(span)
			if v == 0 {
				// 找不到的話，便試著從整個 mspan 找出可用的 object
				// 若沒有的話，則向 mcentral 要一個新的 mspan 並替換舊的 mspan
				v, _, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			
			// 若是新的 tiny block 可能需要更新 mcache 的 tiny 及 tinyoffset
			if size < c.tinyoffset || c.tiny == 0 {
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
			size = maxTinySize
		} else {
			// small size 分配流程
		}
	} else {
		// large size 分配流程
	}

	// 後續處理

	return x
}
{{< /highlight >}}

## Large Size
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/malloc.go#L947)  

Large size 分配流程相對簡單，跳過 `mcache` 及 `mcentral` 直接透過 `mheap` 向 OS 申請需要的空間。

下列是 large size allocating 的 部份內容：
{{< highlight go>}}
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	// 處理 GC 相關事情
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			// tiny size 分配流程
		} else {
			// small size 分配流程
		}
	} else {
		var s *mspan
		shouldhelpgc = true
		systemstack(func() {
			// 跳過 mcache、mcentral，直接透過 mheap 向 OS 申請
			s = largeAlloc(size, needzero, noscan)
		})
		s.freeindex = 1
		s.allocCount = 1
		x = unsafe.Pointer(s.base())
		size = s.elemsize
	}

	// 後續處理

	return x
}

func largeAlloc(size uintptr, needzero bool, noscan bool) *mspan {
	// print("largeAlloc size=", size, "\n")

	if size+_PageSize < size {
		throw("out of memory")
	}
	npages := size >> _PageShift
	if size&_PageMask != 0 {
		npages++
	}

	// Deduct credit for this span allocation and sweep if
	// necessary. mHeap_Alloc will also sweep npages, so this only
	// pays the debt down to npage pages.
	deductSweepCredit(npages*_PageSize, npages)

	// 直接向 OS 申請
	s := mheap_.alloc(npages, makeSpanClass(0, noscan), true, needzero)
	if s == nil {
		throw("out of memory")
	}
	s.limit = s.base() + size
	heapBitsForAddr(s.base()).initSpan(s)
	return s
}
{{< /highlight >}}

# Reference
- https://speakerdeck.com/emfree/allocator-wrestling
- https://blog.ankuranand.com/2019/02/20/a-visual-guide-to-golang-memory-allocator-from-ground-up/
- https://xenojoshua.com/2019/03/golang-memory/#234-%E5%AF%B9%E8%B1%A1%E5%B0%BA%E5%AF%B8%E5%88%86%E7%BA%A7
- https://cloud.tencent.com/developer/article/1422392
- http://legendtkl.com/2017/04/02/golang-alloc/
- https://faaaar.github.io/2019/06/05/go/memory-allocation/
- https://andrestc.com/post/go-memory-allocation-pt1/
- https://www.cnblogs.com/zkweb/p/7880099.html
