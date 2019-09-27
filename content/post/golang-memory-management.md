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

## mspan
`mspan` 是 Golang 記憶體管理最小單位，由數頁 page 所組成，每一個 `mspan` 都會對應到一個 span class，用來決定該如何切割該 `mspan` 裡的 page。其 `mspan` 是 double-linked list。  
page 經過切割後變為 object，此 object 才是真正分配記憶體的最小單位。

## mcache
每個 P (Golang scheduler 的 logic processor)，會綁定一個 `mcache`，由於同一時間只會有一個 G (goroutine) 在 P 上跑，所以在存取時並不需要 lock。  
當 `mcache` 上的 `mspan` 已經沒有空間了，便會向 `mcentral` 申請一個有空間的 `mspan` 來替換舊的 `mspan`。

## mcentral
`mcentral` 用來管理 `mspan`，一個 `mcentral` 負責一種 span class 的 `mspan`。  
當 `mcentral` 無 `mspan` 可用時，便會向 `mheap` 申請新的 `mspan` 加入。
目前總共有 64 種 sizes，又根據是否存放 pointer 分為 scan 及 noscan 兩種。所以 `mheap` 裡總共有 134 (67*2) 個 `mcentral`。  
這是為了加速 GC 的 mark 階段，在 non-scan 裡，一定不會有 pointer，所以不需一個一個往下檢查。

## mheap
`mheap` 在整個 Golang runtime 裡只會存在一個，Golang 所用到的所有記憶體都要來向 `mheap` 申請。  
而 `mheap` 自身擁有的記憶體不夠時，會再向 OS 申請，每次申請至少 1MB 的記憶體，減少向 OS 申請的 overhead。  
`mheap` 在管理記憶體時是以 page 為單位在管理的，這裡的 page 並不是 OS 層級的 page。Golang 的一頁 page 大小為 8k bytes。

## arena
Golang 的 heap 是由一堆 `arena` 對齊記憶體位址所組成的。一個 `arena` 大小為 64MB (不同平台、架構有所不同，此文皆以 Linux-64bit 為主)。  

## heapArena
每一個 `arena` 都會有一相對應的 `heapArena`，這裡存放了一些 `arena` 的 metadata，像是 `bitmap` 或是 `spans`。  
在 `mheap` 裡的 `heapArena` 是一個二維 array，在大多數架構下，第一層通常只有一個 element。  

## span class
目前共有 67 種 size classes，每種 class 訂出了其 span 多大、其 object 多大。  
而每一種 size 又會分為 scan 及 noscan 兩種，所以共有 134 種 span class。  
每個 `mcentral` 負責一種 span class，其包含的所有 `mspan` 將會按照其規格劃分。  

# Component
## Virtual Memory Layout
Golang 的 virtual memory layout 長相如下：  

![](/content/images/2019/07/golang_memory_layout.png)

Golang 的 heap 是由一堆 arena 所組成，每個 arena 會對齊。而每一個 arena 會有一個相對應的 `heapArena` (`mheap.arenas`)，裡面存放 `spans` 及 `bitmap`，是 arena 的 metadata。  
arena 根據不同 CPU 架構、OS 會有所不同，常見的 linux 64-bit 目前是 64MB。  
在增加 arena 時，會盡量試著讓 arena 是連續的，如此在分配較大的 `mspan` 時便能橫跨 arena。

## mspan
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/mheap.go#L316)  

下列是 `mspan` 結構，僅列出此文會講到的幾個 fields：  
{{< highlight go>}}
// mheap.go
type mspan struct {
	next *mspan     // 指向下一個 mspan
	prev *mspan     // 指向前一個 mspan
	list *mSpanList // For debugging. TODO: Remove.

	startAddr uintptr // 此 span 第一個 byte 的位置，即 s.base()
	npages    uintptr // 此 span 所擁有的 page 數量

	// freeindex 值為 0 - nelem 之間，用來標示下一個「可能」可用的 object 位址。
	// 在 freeindex 之前的 object 皆已被使用，而之後的 object 可能被使用，也可能還沒。
	// 搭配 allocCache 使用可以快速找出可使用的 object，而不需全部檢查一次
    freeindex uintptr
	nelems uintptr // 此 span 所擁有的 object 數量
    
    // 從 freeindex 位置開始 cache allocBits，存 allocBits 的補數，
	// 因此可以直接使用 ctz (count trailing zero) 快速找出可用的 object。
    allocCache uint64

    allocBits  *gcBits // 標示哪些 object 被分配了，哪些還沒
	gcmarkBits *gcBits // 標示哪些 object 被標記了，哪些還沒
	
	// sweep generation:
	// if sweepgen == h->sweepgen - 2，此 mspan 需要 sweep
	// if sweepgen == h->sweepgen - 1，此 mspan 正在被 swept
	// if sweepgen == h->sweepgen，此 mspan 已經被 swept 了，且準備好被使用了
	// if sweepgen == h->sweepgen + 1，此 mspan 在 sweep 開始前被 cached 了，現在還是被 cached 且需要 sweep
	// if sweepgen == h->sweepgen + 3，此 mspan 被 swept 後被 cached 了，現在還是被 cached
	// h->sweepgen 經過每次 GC 後就增加 2
	sweepgen    uint32
	allocCount  uint16     // 此 span 已被分配的 object 數量
	spanclass   spanClass  // 此 span 的 class
	state       mSpanState // mspaninuse etc
	needzero    uint8      // 分配前是否需要 zeroed
	elemsize    uintptr    // 此 span 一個 object 的大小 computed from sizeclass or from npages
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

## mcache
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/mcache.go#L19)  

下列是 `mcache` 結構，僅列出此文會講到的幾個 fields：  
{{< highlight go>}}
// mcache.go
//go:notinheap
type mcache struct {
	tiny             uintptr // tiny object 的起始位址
	tinyoffset       uintptr // 從 tiny 開始的 offset，即下次分配 tiny 的地方
	local_tinyallocs uintptr // tiny object 的數量

	alloc [numSpanClasses]*mspan // 各式大小的 mspan list，共有 134 種，其 index 是 spanClass
}
{{< /highlight >}}

## mcentral
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/mcentral.go#L20)  

下列是 `mcentral` 結構，僅列出此文會講到的幾個 fields：  
{{< highlight go>}}
// mcentral.go
//go:notinheap
type mcentral struct {
	lock      mutex // 可能同時會有多個 mcache 向 mcentral 要求 mspan，所以需要 lcok
	spanclass spanClass // 表示此 mcentral 是負責哪個 span class
	nonempty  mSpanList // 仍有可用 object 的 mspan double linked list
	empty     mSpanList // 無可用 object 的 mspan double linked list 或是被 cached 在 mcache 了
	nmalloc uint64 // 此 mcentral 所分配的 object 數量
}
{{< /highlight >}}

## mheap
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/mheap.go#L31)  

下列是 `mheap` 結構，僅列出此文會講到的幾個 fields：  
{{< highlight go>}}
// mheap.go
//go:notinheap
type mheap struct {
	lock      mutex // 可能同時有多個 mcentral 向 mheap 請求，所以需要 lock

	// 下列為 mheap 的兩棵樹堆，以 mspan 的 page 數量排序
	free      mTreap // 可用且並非回收的 mspan
	scav      mTreap // 可用且已經回收的 mspan
	
	allspans []*mspan // 所有產生的 mspan

	reclaimIndex uint64 // 下一個要被 reclaim 的 page 位址
	reclaimCredit uintptr // 額外多 reclaim 的 pages，先存在這裡，下次從這裡先扣除

	scavengeCredit uintptr // 額外多 scavenge 的 pages，先存在這裡，下次從這裡先扣除

	// Malloc stats.
	largealloc  uint64                  // 已分配的 large object bytes 數
	nlargealloc uint64                  // 已分配的 large object 數量
	largefree   uint64                  // 已歸還的 large object bytes 數
	nlargefree  uint64                  // 已歸還的 large object 數量
	nsmallfree  [_NumSizeClasses]uint64 // 已歸還的 small object 數量

	// arenas 存放指向 heapArena 的 pointer。
	// arenas 是兩層的 array，目前大多數情況都第一層都只有一個 element，在存取上會較有效率
	// heapArena 儲存各 arena 的 metadata。
	arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

	arenaHints *arenaHint // 當 mheap 要增加 heap arenas 時，會從 arenaHints 開始增加

	allArenas []arenaIdx // allArenas 存放所有 arena 的 index

	// central 存放各種規格的 mcentral，共有 134 種
	central [numSpanClasses]struct {
		mcentral mcentral
		pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
	}

	// 下列為各結構的 allocator，fixalloc 用來分配 heap 以外且大小固定的 object
	spanalloc             fixalloc // allocator for span*
	cachealloc            fixalloc // allocator for mcache*
	treapalloc            fixalloc // allocator for treapNodes*
	arenaHintAlloc        fixalloc // allocator for arenaHints
}

//go:notinheap
type heapArena struct {
	// bitmap 用來表示 arena 哪些位址存放 object，哪些位址是 pointer。
	// 一個 byte 的 bitmap 用來表示 4 個在 arena 的 pointer，即兩個 bits 表示一個 pointer。
	bitmap [heapArenaBitmapBytes]byte

	// spans 將 arena 裡的 page 對應到相對應的 mspan。
	spans [pagesPerArena]*mspan
}
{{< /highlight >}}

# Allocating Flow
Golang 將記憶體分層存取，也依 size 分級存取。對於特別大及特別小的亦有另外的處理方式。  
依據 size 及是否為 pointer 分為三種 allocating 流程：

- Tiny size：`< 16 bytes` 且不為 pointer 的 object
- Small size：`<= 32KB` 的 object
- Large size：`> 32KB` 的 object

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
// malloc.go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	// 處理 GC 相關事情
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			off := c.tinyoffset
			// 根據要分配的 size 對齊 offset
			if size&7 == 0 {
				off = round(off, 8)
			} else if size&3 == 0 {
				off = round(off, 4)
			} else if size&1 == 0 {
				off = round(off, 2)
			}

			// 檢查此 mcache 是否有 tiny block
			// 若有且放的下的話，便放進此 tiny block 並調整 offset 後直接返回 pointer
			if off+size <= maxTinySize && c.tiny != 0 {
				x = unsafe.Pointer(c.tiny + off)
				c.tinyoffset = off + size
				c.local_tinyallocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}

			span := c.alloc[tinySpanClass] // 目前沒有 tiny block 或是不夠用，便從 mcache 拿出 mspan

			v := nextFreeFast(span) // 利用 mspan.allocCache 快裡找出可用的 object
			if v == 0 {
				v, _, shouldhelpgc = c.nextFree(tinySpanClass) // 找不到的話，便試著從整個 mspan 找出可用的 object
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			
			// 根據剩餘的空間決定需不需要替換現在的 tiny block
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

`nextFreeFast` 利用 `allocCache` 這個 function 快速找出可用的 object 位置：  
{{< highlight go>}}
// malloc.go
// nextFreeFast returns the next free object if one is quickly available.
// Otherwise it returns 0.
func nextFreeFast(s *mspan) gclinkptr {
	theBit := sys.Ctz64(s.allocCache) // 從低位開始計算 allocCache 有幾個 0
	if theBit < 64 {
		result := s.freeindex + uintptr(theBit)
		if result < s.nelems { // 若找出來的 bit 還在該 mspan 可用的範圍內，便可直接使用
			freeidx := result + 1
			if freeidx%64 == 0 && freeidx != s.nelems {
				return 0
			}
			s.allocCache >>= uint(theBit + 1) // 調整 allocCache，因為此次會用掉一個 object，所以 +1
			s.freeindex = freeidx // 調整 freeindex
			s.allocCount++
			return gclinkptr(result*s.elemsize + s.base())
		}
	}
	return 0 // 若無法找到可用的 object，便回傳 0 讓外面處理
}
{{< /highlight >}}

若 `nextFreeFast` 找不到，便呼叫 `nextFree`：  
{{< highlight go>}}
// malloc.go
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
	freeIndex := s.nextFreeIndex() // 找出下個可用的 freeIndex
	if freeIndex == s.nelems { // 若 nextFreeIndex 回傳 nelems 代表該 mspan 已無可用的 object
		if uintptr(s.allocCount) != s.nelems {
			println("runtime: s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
			throw("s.allocCount != s.nelems && freeIndex == s.nelems")
        }

		c.refill(spc) // 這個 mspan 已經沒有可用的 object 了，呼叫 refill 向 mcentral 申請一個新的 mspan 來替換舊的
		shouldhelpgc = true
		s = c.alloc[spc]

		freeIndex = s.nextFreeIndex() // 從新的 mspan 找出下一個可用的 freeIndex
	}

	if freeIndex >= s.nelems {
		throw("freeIndex is not valid")
	}

	v = gclinkptr(freeIndex*s.elemsize + s.base()) // 順利找到可用的 object，計算 object 的位址後回傳
	s.allocCount++
	if uintptr(s.allocCount) > s.nelems {
		println("s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
		throw("s.allocCount > s.nelems")
	}
	return
}
{{< /highlight >}}

接著來看看 `nextFreeIndex` 如何找出可用的 `freeIndex`：  
{{< highlight go>}}
// mbitmap.go
// nextFreeIndex returns the index of the next free object in s at
// or after s.freeindex.
// There are hardware instructions that can be used to make this
// faster if profiling warrants it.
func (s *mspan) nextFreeIndex() uintptr {
	sfreeindex := s.freeindex
	snelems := s.nelems

	// 若目前的 freeIndex 為該 mspan 總可用 object 數量時，代表此 mspan 已無可用的 object 了
	if sfreeindex == snelems {
		return sfreeindex
	}
	if sfreeindex > snelems {
		throw("s.freeindex > s.nelems")
	}

	aCache := s.allocCache

	// 下面 loop 利用 allocCache 與 freeIndex 來 iterate 整個 mspan 試著找出可用的 object 位置
	bitIndex := sys.Ctz64(aCache)
	for bitIndex == 64 { // sys.Ctz64 回傳 64 的話，代表 allocCache 裡全部都是 0，代表 allocCache 所涵蓋的部份沒有可用的 object
		// 將 freeIndex 移到下 8 個 bytes
		sfreeindex = (sfreeindex + 64) &^ (64 - 1)
		if sfreeindex >= snelems { // freeIndex 若超過此 mspan 總可用 object 數量，表示該 mspan 已無可用的 object
			s.freeindex = snelems
			return snelems
		}
		whichByte := sfreeindex / 8
		// 根據新的 freeIndex 重新填補 allocCache
		s.refillAllocCache(whichByte)
		aCache = s.allocCache
		bitIndex = sys.Ctz64(aCache)
		// nothing available in cached bits
		// grab the next 8 bytes and try again.
	}
	result := sfreeindex + uintptr(bitIndex)
	if result >= snelems {
		s.freeindex = snelems
		return snelems
	}

	s.allocCache >>= uint(bitIndex + 1) // 調整 allocCache
	sfreeindex = result + 1 // 調整 freeIndex

	if sfreeindex%64 == 0 && sfreeindex != snelems {
		// We just incremented s.freeindex so it isn't 0.
		// As each 1 in s.allocCache was encountered and used for allocation
		// it was shifted away. At this point s.allocCache contains all 0s.
		// Refill s.allocCache so that it corresponds
		// to the bits at s.allocBits starting at s.freeindex.
		whichByte := sfreeindex / 8
		s.refillAllocCache(whichByte)
	}
	s.freeindex = sfreeindex
	return result
}
{{< /highlight >}}

若該 `mspan` 已無可用的 object 便呼叫 `refill` 向 `mcentral` 申請新的 `mspan` 替換舊的：  
{{< highlight go>}}
// mcache.go
func (c *mcache) refill(spc spanClass) {
	s := c.alloc[spc] // 取出目前滿載的 mspan

	if uintptr(s.allocCount) != s.nelems {
		throw("refill of span with free space remaining")
	}
	if s != &emptymspan {
		// 將此 mspan 標記為不再被 mcache cached
		if s.sweepgen != mheap_.sweepgen+3 {
			throw("bad sweepgen in refill")
		}
		atomic.Store(&s.sweepgen, mheap_.sweepgen)
	}

	s = mheap_.central[spc].mcentral.cacheSpan() // 從 mcentral 再拿一個新的 mspan
	if s == nil {
		throw("out of memory")
	}

	if uintptr(s.allocCount) == s.nelems {
		throw("span has no free space")
	}

	// Indicate that this span is cached and prevent asynchronous
	// sweeping in the next sweep phase.
	s.sweepgen = mheap_.sweepgen + 3

	c.alloc[spc] = s // 將新的 mspan 放到 mcache 裡
}
{{< /highlight >}}

接著再看 `cacheSpan` 如何從 `mcentral` 拿出一個新的 `mspan` 放到 `mcache` 裡：  
{{< highlight go>}}
// mcentral.go
func (c *mcentral) cacheSpan() *mspan {
	// Deduct credit for this span allocation and sweep if necessary.
	spanBytes := uintptr(class_to_allocnpages[c.spanclass.sizeclass()]) * _PageSize
	deductSweepCredit(spanBytes, 0)

	lock(&c.lock) // mcentral 是共用的，需要 lock
	traceDone := false
	if trace.enabled {
		traceGCSweepStart()
	}
	sg := mheap_.sweepgen
retry:
	var s *mspan
    // 試著在 nonempty mspan linked-list (還有可用 object 的 mspan) 找出可用的 mspan
	for s = c.nonempty.first; s != nil; s = s.next {
		// sg-2 代表該 mspan 需要 sweep，試著將他的狀態改成 sg-1，即為該 mspan 正在被 swept
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			c.nonempty.remove(s) // 將該 mspan 從 nonempty 移除
			c.empty.insertBack(s) // 將該 mspan insert 到 empty 的尾巴
			unlock(&c.lock) // 找到可用的 mspan 了，可以 unlock mcentral 了
			s.sweep(true) // sweep 該 mspan
			goto havespan
		}
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// 該 mspan 可以直接使用，不需 sweep，直接將他從 nonempty 移除，並 insert 到 empty 的尾巴
		c.nonempty.remove(s)
		c.empty.insertBack(s)
		unlock(&c.lock)
		goto havespan
	}

    // 試著在 empty mspan linked-list (已經沒有空間或被 cached 的 mspan) 找出可用的 mspan
	for s = c.empty.first; s != nil; s = s.next {
		// 與上面一樣，找出需 sweep 的 mspan 並試著改他的狀態
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			c.empty.remove(s)
			c.empty.insertBack(s)
            unlock(&c.lock)
            
            // sweep 該 mspan
			s.sweep(true)

			// 看看 swept 後的 mspan 的 freeIndex 是否可用
			freeIndex := s.nextFreeIndex()
			if freeIndex != s.nelems {
				s.freeindex = freeIndex
				goto havespan
			}
			lock(&c.lock)

			// 該 mspan 在 swept 後還是空的，並且已經在 empty 裡了，直接 retry
			// 這裡我自己也不是很清楚為什麼是直接 retry，而不是繼續檢查 empty list 裡的其他 mspan
			goto retry
		}
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// already swept empty span,
		// all subsequent ones must also be either swept or in process of sweeping
		// 上面是原文註解，但我依然不是很清楚為什麼這裡直接 break。
		// 例如第一個 mspan 是不需要 sweep 的話 (單純滿了)，那便直接 break 不繼續檢查其他 mspan 了嗎？
		break
	}
	if trace.enabled {
		traceGCSweepDone()
		traceDone = true
	}
	unlock(&c.lock)

	s = c.grow() // 在 nonempty 及 empty 都找不到可用的 mspan，需要向 mheap 申請
	if s == nil {
		return nil
	}
	lock(&c.lock)
	c.empty.insertBack(s)
	unlock(&c.lock)

	// At this point s is a non-empty span, queued at the end of the empty list,
	// c is unlocked.
havespan:
	if trace.enabled && !traceDone {
		traceGCSweepDone()
	}
	n := int(s.nelems) - int(s.allocCount)
	if n == 0 || s.freeindex == s.nelems || uintptr(s.allocCount) == s.nelems {
		throw("span has no free objects")
	}
	// Assume all objects from this span will be allocated in the
	// mcache. If it gets uncached, we'll adjust this.
	atomic.Xadd64(&c.nmalloc, int64(n))
	usedBytes := uintptr(s.allocCount) * s.elemsize
	atomic.Xadd64(&memstats.heap_live, int64(spanBytes)-int64(usedBytes))
	if trace.enabled {
		// heap_live changed.
		traceHeapAlloc()
	}
	if gcBlackenEnabled != 0 {
		// heap_live changed.
		gcController.revise()
	}
	freeByteBase := s.freeindex &^ (64 - 1)
	whichByte := freeByteBase / 8
	// Init alloc bits cache.
	s.refillAllocCache(whichByte)

	// Adjust the allocCache so that s.freeindex corresponds to the low bit in
	// s.allocCache.
	s.allocCache >>= s.freeindex % 64

	return s
}
{{< /highlight >}}

上面是 `mcache` 向 `mcentral` 申請新的 `mspan`，如果 `mcentral` 也沒有，便呼叫 `grow` 向 `mheap` 申請：  
{{< highlight go>}}
// mcentral.go
func (c *mcentral) grow() *mspan {
	// 根據該 mcentral 的 spanclass 計算要申請的量
	npages := uintptr(class_to_allocnpages[c.spanclass.sizeclass()])
	size := uintptr(class_to_size[c.spanclass.sizeclass()])
	n := (npages << _PageShift) / size

	s := mheap_.alloc(npages, c.spanclass, false, true) // 向 mheap 申請 mspan 來使用
	if s == nil {
		return nil
	}

	p := s.base()
	s.limit = p + size*n

	heapBitsForAddr(s.base()).initSpan(s)
	return s
}
{{< /highlight >}}

繼續看 `alloc` 這個 function：  
{{< highlight go>}}
// mheap.go
// alloc allocates a new span of npage pages from the GC'd heap.
//
// Either large must be true or spanclass must indicates the span's
// size class and scannability.
//
// If needzero is true, the memory for the returned span will be zeroed.
func (h *mheap) alloc(npage uintptr, spanclass spanClass, large bool, needzero bool) *mspan {
	// Don't do any operations that lock the heap on the G stack.
	// It might trigger stack growth, and the stack growth code needs
	// to be able to allocate heap.
	var s *mspan
	systemstack(func() {
		s = h.alloc_m(npage, spanclass, large) // 呼叫 alloc_m 申請 mspan
	})

	if s != nil {
		if needzero && s.needzero != 0 {
			memclrNoHeapPointers(unsafe.Pointer(s.base()), s.npages<<_PageShift)
		}
		s.needzero = 0
	}
	return s
}
{{< /highlight >}}

繼續看 `alloc_m` 這個 function:  
{{< highlight go>}}
// mheap.go
// alloc_m is the internal implementation of mheap.alloc.
//
// alloc_m must run on the system stack because it locks the heap, so
// any stack growth during alloc_m would self-deadlock.
//
//go:systemstack
func (h *mheap) alloc_m(npage uintptr, spanclass spanClass, large bool) *mspan {
	_g_ := getg()

	// 為了避免 heap 長太快，會先試著先回收一些 page 回來
	if h.sweepdone == 0 {
		h.reclaim(npage)
	}

	lock(&h.lock)
	// transfer stats from cache to global
	memstats.heap_scan += uint64(_g_.m.mcache.local_scan)
	_g_.m.mcache.local_scan = 0
	memstats.tinyallocs += uint64(_g_.m.mcache.local_tinyallocs)
	_g_.m.mcache.local_tinyallocs = 0

	s := h.allocSpanLocked(npage, &memstats.heap_inuse) // 從 mheap 裡分配 mspan
	if s != nil {
		// 成功分配 mspan 的話，便更新該 mspan 的一些資訊
		// Record span info, because gc needs to be
		// able to map interior pointer to containing span.
		atomic.Store(&s.sweepgen, h.sweepgen)
		h.sweepSpans[h.sweepgen/2%2].push(s) // Add to swept in-use list.
		s.state = mSpanInUse
		s.allocCount = 0
		s.spanclass = spanclass
		if sizeclass := spanclass.sizeclass(); sizeclass == 0 {
			s.elemsize = s.npages << _PageShift
			s.divShift = 0
			s.divMul = 0
			s.divShift2 = 0
			s.baseMask = 0
		} else {
			s.elemsize = uintptr(class_to_size[sizeclass])
			m := &class_to_divmagic[sizeclass]
			s.divShift = m.shift
			s.divMul = m.mul
			s.divShift2 = m.shift2
			s.baseMask = m.baseMask
		}

		// Mark in-use span in arena page bitmap.
		arena, pageIdx, pageMask := pageIndexOf(s.base())
		arena.pageInUse[pageIdx] |= pageMask

		// update stats, sweep lists
		h.pagesInUse += uint64(npage)
		if large {
			memstats.heap_objects++
			mheap_.largealloc += uint64(s.elemsize)
			mheap_.nlargealloc++
			atomic.Xadd64(&memstats.heap_live, int64(npage<<_PageShift))
		}
	}
	// heap_scan and heap_live were updated.
	if gcBlackenEnabled != 0 {
		gcController.revise()
	}

	if trace.enabled {
		traceHeapAlloc()
	}

	// h.spans is accessed concurrently without synchronization
	// from other threads. Hence, there must be a store/store
	// barrier here to ensure the writes to h.spans above happen
	// before the caller can publish a pointer p to an object
	// allocated from s. As soon as this happens, the garbage
	// collector running on another processor could read p and
	// look up s in h.spans. The unlock acts as the barrier to
	// order these writes. On the read side, the data dependency
	// between p and the index in h.spans orders the reads.
	unlock(&h.lock)
	return s
}
{{< /highlight >}}

再來重點是 `allocSpanLocked` 這個 function：  
{{< highlight go>}}
// mheap.go
// Allocates a span of the given size.  h must be locked.
// The returned span has been removed from the
// free structures, but its state is still mSpanFree.
func (h *mheap) allocSpanLocked(npage uintptr, stat *uint64) *mspan {
	var s *mspan

	s = h.pickFreeSpan(npage) // 從 mheap 裡找出可用的 mspan，找不到則回傳 nil
	if s != nil {
		goto HaveSpan // 成功找到可用的 mspan，跳到 HaveSpan section
	}
	// 若找不到可用的 mspan，便增加 heap 後再試一次
	if !h.grow(npage) {
		return nil // 增加 heap 失敗，某個地方出了嚴重的錯誤，直接 return nil
	}
	s = h.pickFreeSpan(npage)
	if s != nil {
		goto HaveSpan
	}
	throw("grew heap, but no adequate free span found")

HaveSpan:
	// Mark span in use.
	if s.state != mSpanFree {
		throw("candidate mspan for allocation is not free")
	}
	if s.npages < npage {
		throw("candidate mspan for allocation is too small")
	}

	// First, subtract any memory that was released back to
	// the OS from s. We will re-scavenge the trimmed section
	// if necessary.
	memstats.heap_released -= uint64(s.released())

	// pickFreeSpan 這個 function 會找出最小可容納 npage 的 mspan，所以找到的 mspan 大小可能超過 npage，
	// 此時會將多餘的空間切掉還回去給 heap，只拿需要的大小就好
	if s.npages > npage {
		t := (*mspan)(h.spanalloc.alloc()) // 分配 mspan 的結構
		t.init(s.base()+npage<<_PageShift, s.npages-npage) // 把多出來的空間 initialize 成一個新的 mspan
		s.npages = npage
		h.setSpan(t.base()-1, s)
		h.setSpan(t.base(), t)
		h.setSpan(t.base()+t.npages*pageSize-1, t)
		t.needzero = s.needzero
		// 若 s 是 scanvenged，則 t 也有可能是
		start, end := t.physPageBounds()
		if s.scavenged && start < end {
			memstats.heap_released += uint64(end - start)
			t.scavenged = true
		}
		s.state = mSpanManual // prevent coalescing with s
		t.state = mSpanManual
		h.freeSpanLocked(t, false, false, s.unusedsince) // 將 t (用多出來的空間新成立的 mspan) 還回去 mheap (放入 mheap.free)
		s.state = mSpanFree
	}
	// "Unscavenge" s only AFTER splitting so that
	// we only sysUsed whatever we actually need.
	if s.scavenged {
		// sysUsed all the pages that are actually available
		// in the span. Note that we don't need to decrement
		// heap_released since we already did so earlier.
		sysUsed(unsafe.Pointer(s.base()), s.npages<<_PageShift)
		s.scavenged = false
	}
	s.unusedsince = 0

	h.setSpans(s.base(), npage, s)  // 更新 mheap 裡的 spans 資訊

	*stat += uint64(npage << _PageShift)
	memstats.heap_idle -= uint64(npage << _PageShift)

	//println("spanalloc", hex(s.start<<_PageShift))
	if s.inList() {
		throw("still in list")
	}
	return s
}
{{< /highlight >}}

接著 `pickFreeSpan` 這個 function 來得知 `mheap` 怎麼找出可用的 `mspan`：  
{{< highlight go>}}
// mheap.go
// pickFreeSpan acquires a free span from internal free list
// structures if one is available. Otherwise returns nil.
// h must be locked.
func (h *mheap) pickFreeSpan(npage uintptr) *mspan {
	// free 及 scav 是樹堆 (tree+heap=treap)，其 priority 是 mspan 的 pages
	tf := h.free.find(npage) // 找出最小且可容納 npage 大小的 mspan
	ts := h.scav.find(npage) // 找出最小且可容納 npage 大小的 mspan

	// 看誰給我們的 mspan 不為 nil 且比較接近 npage
	var s *mspan
	if tf != nil && (ts == nil || tf.spanKey.npages <= ts.spanKey.npages) {
		s = tf.spanKey // treap node 裡的 spanKey 就是 mspan 的 pointer
		h.free.removeNode(tf) // 移除該 treap node
	} else if ts != nil && (tf == nil || tf.spanKey.npages > ts.spanKey.npages) {
		s = ts.spanKey
		h.scav.removeNode(ts)
	}
	return s
}
{{< /highlight >}}

最後一個 function 了！當 `mheap` 空間不夠時，會呼叫 `grow` 向 OS 申請空間：  
{{< highlight go>}}
// mheap.go
// Try to add at least npage pages of memory to the heap,
// returning whether it worked.
//
// h must be locked.
func (h *mheap) grow(npage uintptr) bool {
	ask := npage << _PageShift // 將想申請的頁數轉換成 bytes
	v, size := h.sysAlloc(ask) // 向 OS 申請空間
	if v == nil {
		print("runtime: out of memory: cannot allocate ", ask, "-byte block (", memstats.heap_sys, " in use)\n")
		return false
	}

	// Scavenge some pages out of the free treap to make up for
	// the virtual memory space we just allocated. We prefer to
	// scavenge the largest spans first since the cost of scavenging
	// is proportional to the number of sysUnused() calls rather than
	// the number of pages released, so we make fewer of those calls
	// with larger spans.
	h.scavengeLargest(size)

	// Create a fake "in use" span and free it, so that the
	// right coalescing happens.
	s := (*mspan)(h.spanalloc.alloc()) // 分配 mspan 結構
	s.init(uintptr(v), size/pageSize) // 根據得到的位址 initialize 一個新的 mspan
	h.setSpans(s.base(), s.npages, s) // 更新 mheap.spans 的資訊，即哪個 page 對應到哪個 mspan
	atomic.Store(&s.sweepgen, h.sweepgen)
	s.state = mSpanInUse
	h.pagesInUse += uint64(s.npages)
	h.freeSpanLocked(s, false, true, 0) // 將該 mspan 插入 mheap.free 裡
	return true // return true 表示成功向 OS 申請空間
}
{{< /highlight >}}

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

			size = uintptr(class_to_size[sizeclass]) // 將 size round up 為此 class 的 object size
			spc := makeSpanClass(sizeclass, noscan) // 根據 size class 及是否能 scan 決定 span class
			span := c.alloc[spc] // 根據 span class 取出 mcache 裡的 mspan

			v := nextFreeFast(span) // 利用 mspan.allocCache 快裡找出可用的 object (回頭看上面的 tiny size 流程)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc) // 找不到的話，便試著從整個 mspan 找出可用的 object (回頭看上面的 tiny size 流程)
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
{{< /highlight >}}

額外看一下 `makeSpanClass`：  
{{< highlight go>}}
func makeSpanClass(sizeclass uint8, noscan bool) spanClass {
	// 可以看到 size class 與 span class 的轉換方式其實就是將 size class * 2 再根據是否為 pointer 加 0 或 1
	return spanClass(sizeclass<<1) | spanClass(bool2int(noscan))
}
{{< /highlight >}}

## Large Size
[Source code](https://github.com/golang/go/blob/go1.12.5/src/runtime/malloc.go#L947)  

Large size 分配流程相對簡單，跳過 `mcache` 及 `mcentral` 直接透過 `mheap` 向 OS 申請需要的空間。

下列是 large size allocating 的 部份內容：
{{< highlight go>}}
// malloc.go
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
			
			s = largeAlloc(size, needzero, noscan) // 呼叫 largeAlloc
		})
		s.freeindex = 1
		s.allocCount = 1
		x = unsafe.Pointer(s.base())
		size = s.elemsize
	}

	// 後續處理

	return x
}
{{< /highlight >}}

接著看 `largeAlloc` 這個 function：  
{{< highlight go>}}
// malloc.go
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

	s := mheap_.alloc(npages, makeSpanClass(0, noscan), true, needzero) // 跳過 mcache、mcentral，直接向 mheap 申請 (回頭看上面的 tiny size 流程)
	if s == nil {
		throw("out of memory")
	}
	s.limit = s.base() + size
	heapBitsForAddr(s.base()).initSpan(s)
	return s
}
{{< /highlight >}}

## Flow Chart
搭配底下的圖片可知整個 Golang 的分配流程是有層級的。事先向 OS 申請一大塊連續的虛擬記憶體，再切割成各種粒度的 mspan 來分配。  
只要能弄懂 tiny allocation 的流程，剩下的 small 及 large 便也能理解了，他們其實都是重覆一樣的動作而已。  
![](/content/images/2019/07/golang-memory-allocating-flow.png)

# Reference
- https://speakerdeck.com/emfree/allocator-wrestling
- https://blog.ankuranand.com/2019/02/20/a-visual-guide-to-golang-memory-allocator-from-ground-up/
- https://xenojoshua.com/2019/03/golang-memory/#234-%E5%AF%B9%E8%B1%A1%E5%B0%BA%E5%AF%B8%E5%88%86%E7%BA%A7
- https://cloud.tencent.com/developer/article/1422392
- http://legendtkl.com/2017/04/02/golang-alloc/
- https://faaaar.github.io/2019/06/05/go/memory-allocation/
- https://andrestc.com/post/go-memory-allocation-pt1/
- https://www.cnblogs.com/zkweb/p/7880099.html
- http://lessisbetter.site/2019/07/06/go-memory-allocation/
- https://programmer.help/blogs/exploration-of-golang-source-code-3-realization-principle-of-gc.html
- https://kknews.cc/zh-tw/code/5rbk9kk.html