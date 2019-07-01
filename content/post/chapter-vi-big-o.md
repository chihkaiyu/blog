---
title: "Chapter VI - Big O"
slug: "chapter-vi-big-o"
date: 2019-03-07T00:24:00+08:00
draft: false
image: "/content/images/2019/03/DSC05004.jpg"
tags: ["digest"]
---

# Chapter VI - Big O

# Big O, Big Theta, and Big Omega
學術上使用 big O, big theta, big omega 來描述執行速度。然而工程上 (或是面試)，人們常會將 big theta 及 big O 混在一起。工程上的 big O 即是學術上的 big theta。  
例如印出一個 array，在學術上說 O(N^2) 是正確的，但工程上我們會說這是 O(N)。  
本書所有例子都會使用工程上的 big O。  

## O (big O)
學術上，big O 用來描述時間的 upper bound。例如印出一個 array 的值，可以說是 O(N)，但也可以說是 O(N^2)、O(N^3)。有點類似小於等於的概念。

## Ω (big omega)
學術上，big omega 等價於 lower bound。例如印出一個 array 的值，可以說是 Ω(N)，也可以說是 Ω(log N) 或 Ω(1)。即是：不會比他 (N) 還更快。

## θ (big theta)
學術上，big theta 表示 big O 及 big omega。例如某個演算法是 θ(N) 當他為 O(N) 且為 Ω(N)。

# Space Complexity
空間複雜度與時間複雜度是相似的，例如我們需要產生一長度為 n 的 array，則將需要 O(n) 的空間。如果需要二維 nxn 的 array，則需要 O(n^2)。  
遞迴的 stack 也必須計算在內，例如底下這段 code，他的時間複雜度為 O(n)，空間複雜度為 O(n)。  
```go
func sum(n int) int {
    if n <= 0 {
        return 0
    }
    return n + sum(n-1)
}
```
每一次呼叫都會增加一層堆疊，並真的有使用到記憶體。
```
sum(4)
    -> sum(3)
        -> sum(2)
            -> sum(1)
                -> sum(0)
```
然而並不是呼叫了 n 次就代表使用了 O(n) 的空間，底下這段 code 的時間複雜度為 O(n) 然而空間複雜度只有 O(1)，因為每一次呼叫並不是同時發生的。  
```go
func pairSumSequence(n int) int {
    sum := 0
    for i := 0; i < n; i++ {
        sum += pairSum(i, i + 1)
    }
}

func pairSum(a, b int) int {
    return a + b
}
```

# Drop the Constants
在某些情形下，O(N) 的 code 是很有可能比 O(1) 的 code 還快的。Big O 只是描述增長的速率，因此我們可以不考慮常數的影響。

# Drop the Non-Dominant Terms
上面提到常數項我們不考慮，那非常數項，又不主要影響的因子呢？  
我們也不考慮。例如 O(N^2 + N^2)，我們會捨棄常數項變成 O(N^2)，那 O(N^2 + N) 呢？既然 N^2 都不捨棄掉了，更何況是 N？  
同理：

- O(N^2 + N) 變成 O(N^2)
- O(N + log N) 變成 O(N)
- O(5*2^N + 1000N^100) 變成 O(2^N)

有些時候我們無法捨棄未知的因子，例如 O(B^2 + A)，我們並不知道 A 與 B 的關係。

# Multi-Part Algorithms: Add vs. Multiply
假設你的演算法有兩個步驟，你哪時候要將時間複雜度相加，哪時候又要相乘呢？  
如下列兩段 code：

- Add the Runtimes: O(A + B)

```go
for _, a := range arrA {
    fmt.Println(a)
}

for _, b := range arrB {
    fmt.Println(b)
}
```

- Multiply the Runtimes: (A*B)

```go
for _, a := range arrA {
    for _, b := range arrB {
        fmt.Println(a + "," + b)
    }
}
```

可得知，你的演算法若是先執行「這個」，完成後再執行「那個」那你就相加；反之，當你每執行一次「這個」的時候，都要執行「那個」，那你就得相乘。

# Amortized Time
可動態增長的 array 在 capacity 滿的時候，會產生一個新的 array，其 capacity 是原本的兩倍，並且會將原本的 element 複製過去新的 array，這會造成 O(N) 的操作。  
但大多數時候新增一個 element 的時間是 O(1)，那要如何計算新增 element 的時間複雜度？  
當 array 滿時候，所需要複製的 element 數為 1, 2, 4, 8, 16, ..., X 次，其時間複雜度便是 1 + 2 + 4 + 8 + ... + X，若從尾巴看回去，則是 X + X/2 + X/4 + X/8 + ... + 1，這約略是 2X。  
因此新增 X 個 element 到 array 裡的時間複雜度是 O(2X)，而分攤到每一次新增便是 O(1)。

# Log N Runtimes
以二元搜尋為例，在 N 個 elements 且排序好的 array 裡，要找出 X 的話，每次都挑位於中間的 element 相比，若 X < middle 則繼續往左邊找，反之則是右邊。如此一來每次都能去除一半的 elements。  
假設需要搜尋 k 次才能在長度為 N 的 array 找到 X，則表示 2^k = N，即：  
```
2^4 = 16 -> log2 16 = 4
log2 N = k -> 2^k = N
```
可得知 k 為 O(log N)。  
若你發現有一演算法，在每次處理完都會少掉一半的 elements，那其時間複雜度便很有可能是 O(log N)。  
然而 log 的 base 會有影響嗎？答案是在 big O 的情況下不影響。

# Recursive Runtimes
試問底下這段 code 的時間複雜度為何？  
```go
func f(n int) int {
    if n <= 1 {
        return 1
    }
    return f(n - 1) + f(n - 1)
}
```
若將每一次呼叫都畫成一個 node，可發現他會是一棵二元樹，因此總 node 數便是時間複雜度：2^0 + 2^1 + 2^2 + ... + 2^N，即是 2^(N+1) - 1，其中 N 為該樹的層數 (從 0 開始計算)。

# Examples and Exercises

## Example 3
```go
func printUnorderedPairs(array []int) {
    for i := 0; i < len(array); i++ {
        for j := i; j < len(array); j++ {
            fmt.Println(array[i], array[j])
        }
    }
}
```
這種 pattern 相當常見，但你不能死記這種 pattern 而是要徹底了解他。  
第一次執行會執行 N-1 次，而第二次為 N-2 次，如此下去。最終我們會有：(N-1) + (N-2) + (N-3) + ... + 1。  
總和便是 N(N-1)/2，其時間複雜度即為 `O(N^2)`。

## Example 4
```go
func printUnorderedPairs(arrayA, arrayB []int) {
    for i := 0; i < len(arrayA); i++ {
        for j := 0; j < len(arrayB); j++ {
            fmt.Println(array[i], array[j])
        }
    }
}
```
與 Example 4 不一樣，這次是兩個長度不一樣的 array，其時間複雜度是 `O(A*B)`，其中 A 為 arrayA 的長度，B 為 arrayB 的長度。  

## Example 7
下列哪些等價於 O(N)？

- O(N + P), where P < N/2
- O(2N)
- O(N + log N)
- O(N + M)

我們逐一討論：

- 由於 P < N/2，所以主要影響的因子是 N，所以可以捨棄 P 不看
- O(2N) 即為 O(N)，因為我們可以捨棄常數項
- O(N) 才是主要因子，我們能捨棄 O(log N)
- 我們無從得知 N 與 M 的關係，所以還是 O(N + M)

上述四項只有最後一項是不等價於 O(N)，其餘皆是。

## Example 8
有一演算法，輸入是 array of strings，會對每一 string 排序，最後會再對整個 array 排序，試問其時間複雜度？  
很多人會這樣回答：排序每組字串是 O(N log N)，而我們要對每組字串排序，所以是 O(N*N log N)。最後要排序整個 array，也是 O(N log N)，所以是 O(N^2 log N + N log N)，捨棄非主要因子項，答案是 O(N^2 log N)。  
_**這完全是錯誤的。**_  
問題在於只有一個 N 但卻代表了所有東西。上述的 N 既是 string 長度 (哪個 string？) 又是 array 長度。  
甚至在面試的時候使用 a 和 b，或是 m 和 n 這種組合都不好，太容易忘掉他們分別代表什麼。  
你應該這樣定義：

- 令 s 為最長 string 的長度
- 令 a 為 array 的長度

我們便能逐一分析：

- 對每組 string 排序是 O(s log s)
- 總共有 a 組，所以需要 O(a*s log s)
- 要排序整個 array，其中有 a 組，你可能會說是 O(a log a)，但這是錯的。我們必須把 string 比對也考慮進來，每組 string 比對是 O(s)，總共有 O(a log a) 組比對，所以是 O(a*s log a)

最後加總起來：O(a*s(log a + log s ))。

## Example 11
下列是計算 n 階的 code，請問其時間複雜度為？  
```go
func factorial(n int) {
    if n < 0 {
        return -1
    } else if n == 0 {
        return 1
    } else {
        return n * factorial(n - 1)
    }
}
```
這只是一個單純的 recursion，從 n 到 n-1 到 n-2 一直到 1，所以是單純的 O(n)。  

## Example 12
下列 code 會印出一組 string 所有的排列組合
```go
func permutation(str, prefix string) {
	if len(str) == 0 {
		fmt.Println(prefix)
	} else {
		for i := 0; i < len(str); i++ {
			rem := str[0:i] + str[i+1:]
			permutation(rem, prefix+string(str[i]))
		}
	}
}

func main() {
	permutation("abc", "")
}
```
這題我們可以簡單的計算 `permutation` 被呼叫了幾次，以及每次呼叫時花多少時間。但我們來試著更準確的 upper bound。  

### How many times does permutation get called in its base case?
要呼叫 `permutation` 時，我們就必須從 string 裡挑一個 character 出來，假設有 7 個 characters，則有 7 種選擇。當挑了一個之後，便還有 6 個 characters 可選擇，如此類推則總共有 7 * 6 * 5 * 4 * 3 * 2 * 1，正好是 7!。

### How many times does permutation get called before its base case?
上面只討論了呼叫了 `permutation` 後的 recursion，而我們也必須考慮外面迴圈所呼叫的次數。  
想像有一棵樹表示每次呼叫，則該樹總共會有 n! 葉節點，而每個葉節點貼在長度為 n 的路徑上。  
因此我們得知此樹最多只會有 n * n! 個 nodes。  

### How long does each function call take?
當要印出字串時，需要 O(n)，因為每個 character 都需要被印出來。  
當在做 string 串接時也需要花 O(n)，因為 `rem`、`prefix` 及 `str[i]` 的總和永遠都是會 n。  
因此每個 node 需要 O(n) 的執行時間。

### What is the totoal runtime?
我們會呼叫 `permutation` O(n * n!) 次，而每一次需要 O(n) 的時間，所以執行時間最多不會超過 O(n^2 * n!)。

## Example 14
下列 code 將 0 至 n 的 Fibonacci 數字印出來，他的時間複雜度為何？
```go
func allFib(n int) {
    for i := 0; i < n; i++ {
        fmt.Printf("%v: %v", i, fib(i))
    }
}

func fib(n int) int {
    if n <= 0 {
        return 0
    } else if n == 1 {
        return 1
    }
    return fib(n - 1) + fib(n - 2)
}
```
很多人看到 `fib(n)` 馬上想到 recursion form 是 O(2^n)，而外面又有一層迴圈呼叫了 n 次，所以這段 code 的時間複雜度是 O(n2^n)。  
_**這是錯的。**_  
錯的地方在於 n 是持續變動的：
```
fib(1) -> 2^1 steps
fib(2) -> 2^2 steps
fib(3) -> 2^3 steps
...
fib(n) -> 2^n steps
```
總共有 2^1 + 2^2 + 2^3 + ... + 2^n，其和是 2^(n+1)，因此時間複雜度為 O(2^n)。

## Example 16
下列 code 會將 1 至 n 所有是 2 的次方數的數字印出來，例如 n 是 4，則會印出 1、2、4。那他的時間複雜度為何？  
```go
func powerOf2(n int) int {
    if n < 1 {
        return 0
    } else if n == 1 {
        fmt.Println(1)
        return 1
    } else {
        prev := powerOf2(n / 2)
        curr := prev * 2
        fmt.Println(curr)
        return curr
    }
}
```

有許多方式可以得到此 code 的時間複雜度，如下：

### What It Does
`powerOf2(50)` 實際的呼叫過程是：
```
powerOf2(50)
    -> powerOf2(25)
        -> powerOf2(12)
            -> powerOf2(6)
                -> powerOf2(3)
                    -> powerOf2(1)
                        -> print & reutnr 1
                    print & reutnr 2
                print & reutnr 4
            print & reutnr 8
        print & reutnr 16
    print & reutnr 32
```
可以看出每呼叫一次，其值就少了一半，其時間複雜度為 O(log n)。

### What It Means
此演算法是要計算出所有 1 至 n 的 2 的次方數。  
每一次呼叫 `powerOf2` 都會正好計算出一個數字並將他印出且回傳。所以若此演算法最後印出 13 個數字，則表示 `powerOf2` 被呼叫了 13 次。  
回到問題本身，在 1 至 n 之間，有多少個 2 的次方數的數字，就代表了 `powerOf2` 會被呼叫幾次，即所求的時間複雜度。  
在 1 到 n 之間有 log n 個 2 的次方數，所以其時間複雜度即為 O(log n)。  

### Rate of Increase
我們也可以思考當 n 變大時，其執行時間會如何變化。其實這也是 big O 所代表的意義。  
若 N 從 P 變成 P+1，則 `powerOf2` 的呼叫次數可能根本不會改變。那什麼時候 `powerOf2` 才會增加？答案是當 n 變為兩倍時。  
因此，要求得 `powerOf2` 的時間複雜度，便是計算將 1 不斷以兩倍成長，直到他變成 n，如同此等式：2^x = n，其 x 便是我們所求的時間複雜度。  
而 x 便是 log n，因此時間複雜度為 O(log n)