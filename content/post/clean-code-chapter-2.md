---
title: "Clean Code Digest - Chapter 2 有意義的命名"
slug: "clean-code-chapter-2"
date: 2018-09-19T22:51:00+08:00
draft: false
image: "/content/images/2018/09/DSC04962.jpg"
tags: ["digest"]
---

# Chapter 2 有意義的命名
# TL; DR
我認為此章節所想表達的有下列事項：
- 意圖：取一個好名稱便能得知此類別、物件、函式、變數的含義，或是它想要做什麼事、被做什麼事
- 一致性：永遠使用同一個字詞來表達相同的抽象概念，同樣語意使用不同字詞會使人誤會
- 區別性：與前一項相反，不相同的東西就讓他能輕易被分別出來
- 社交：使其他人輕鬆維護、開發同一份程式碼，包含面對面的討論

# 讓名稱代表意圖 - 使之名副其實
> 變數、函式或類別的名稱，要能解決大部份的問題。如果一個名稱還需要註解的輔助，那麼這個名稱就不具備展現意圖的能力。

Naming is hard，對於母語不是英文的亞洲人來說，要選一個合適的詞彙並貼切你想表達的意圖就更難了。以往我會盡量避免使用過長的命名，但這本書似乎是鼓勵你這麼做，比起模糊不清，一眼就能看出意圖是更重要的事。
此書舉了一個例子 (我改寫為 Golang)：
```go
func getThem() {
    list1 := []string
    for _, x := range list1 {
        if x[0] == 4 {
            list1 = append(list1, x)
        }
    }
}
```
此段程式碼有許多你必須跳至該變數的宣告處或是賦值處，才能得知他的含義，這導致閱讀程式碼時必須一直中斷思考。作者給了一個改寫過的例子，假設現在要開發一款踩地雷遊戲：
```go
func getFlaggedCells() [][]int {
    flaggedCells := [][]int
    for _, cell := gameBoard {
        if cell[STATUS_VALUE] == FLAGGED {
            flaggedCells = append(flaggedCells, cell)
        }
    }
    return flaggedCells
}
```
上述程式碼將無意義的命名 (`list1`、`x`) 及無意義的常數 (`x[0]`、`4`) 都轉變成一看就能了解的命名，使閱讀者不需自己再將無意義的名稱 mapping 到實際含義。

# 避免誤導
> 程式設計師必須避免留下喪失程式原意的錯誤線索。我們應該避開使用那些與原意圖相違背的字詞。
> 拼寫與意圖相似的字詞就是 `恰當資訊`，而使用與意圖不一致的拼寫就是 `誤導` 了。

我認為要做到避免誤導這件事，除了對自身領域的了解外 (我就從來不知道 `hp`、`aix`、`sco` 是 Unix 平台的名稱)，也還必須對文字有一定的敏感度才行。常常遇到的問題是：英文單字使用錯誤、縮寫取的不好，這是我認為最常造成誤導別人的原因。

# 產生有意義的區別
> 有時候程式設計師僅僅只是為了滿足編譯器或直譯器的規定而寫了某些程式碼。
> 數字序列命名法 (`a1, a2, ..., aN`) 跟意圖命名法是完全背道而馳的。以下列程式為例，如果改用 `source` 和 `destination` 當作參數名稱，這個函式會更容易猜測其意圖。
```go
func copyChars(a1, a2 string) {
    for i, _ := range a1 {
        a2[i] = a1[i]
    }
}
```
> 無意義的字詞是另一種無意義的區別。想像你有一個 `Product` 類別，如果又有另一個叫做 `ProductInfo` 或 `ProductData` 的類別，那就只是讓名稱不同而已，無法使它們代表的意義不同。
> `NameString` 會比 `Name` 這個命名好些嗎？`Name` 有可能會是浮點數嗎？
> 在沒有特別約定的情況下，變數名稱 `moneyAmount` 和 `money` 是沒有區別的，變數名稱 `customerInfo` 和 `customer` 也是沒有區別的。要區別名稱，就請用讀者能辨識出不同之處的區別方式

這裡所要表達的，我認為與上一段是同樣的。如果是同樣的東西，不要畫蛇添足加上許多沒有意義且容易誤導別人的名稱。這小段舉了一個例子：
```go
getActiveAccount()
getActiveAccounts()
getActiveAccountInfo()
```
這個專案的程式設計師要如何知道該呼叫哪個函式才是對的？

# 使用能唸出來的名稱
> 如果你唸不出你取的名稱，你就只能像白痴般用拼音來討論它。因為寫程式是一個社交型活動，能不能發音是相當重要的一件事。
> 比較以下程式碼：
```go
type DtaRcrd102 struct {
    genymdhms time
    modymdhms time
    pszqint   string
}
```
```go
type Customer struct {
    generationTimestamp   time
    modificationTimestamp time
    recordId              string
}
```
這裡提到一個很有趣的概念，「寫程式是一個社交型活動」，所以作者鼓勵你使用能唸出來的變數名稱，在與其他人討論時才能順暢唸下去。其實還有提到另一個原因，「人類對於字詞是很在行的」，我猜大概與人類大腦的運作方式有關。

# 使用可被搜尋的名字
> 也許可以很容易地去搜尋 `MAX_CLASSES_PER_STUDENT`，但如果想搜尋數字 7，就頗為麻煩了。
> 至於我個人的偏好，是只有在宣告小函式的區域變數時，才會使用單一字母的變數。命名的長度應該與其 scope 的大小相對應。
> 例如：
```go
for i := 0; i < 34; i++ {
    s += (t[j]*4)/5
}
```
```go
realDaysPerIdealDay := 4
const WORK_DAYS_PER_WEEK int = 5
sum := 0
for i := 0; i < NUMBER_OF_TASKS; i++ [
    realTaskDays := taskEstimate[j] * realDaysPerIdealDay
    realTaskWeeks := (realdays / WORK_DAYS_PER_WEEK)
    sum += realTaskWeeks
}
```

如果直接使用某個數值，當有人看到時常常會不懂為什麼這裡是這個數字，但若是宣告在變數裡，一來可以藉由變數名稱來表達該數值的意圖，二來也方便快速搜尋。

# 避免編碼
> 編碼已經夠多了，不要再增加我們的負擔。
## 匈牙利標誌法
> 以前的編譯器不會進行資料型態的檢查，所以程式設計師需要使用匈牙利標誌法，來幫助他們記住資料型態。
> 使用較小的類別和簡短的經式是一股趨勢，如此一來，人們使用變數時，通常也看得到這些變數的宣告區塊。

這件事在強型態的語言我可以理解，但像 Python 這種弱型態的語言，我其實不知道為什麼不在變數上加上型態？我有時候也會這麼做，像是用 `pykafka` 從 Kafka 拿出 message 時，其值是 `bytes`，我便會命名 `msg_bytes`。在 `避免誤導` 那段底下有寫著註解：「即便這個容器真的是個 `List`，還是盡可能不要把容器的態加入到名稱當中」，我想這是因為要「產生有意義的區別」吧？

## 成員的字首 (Member Prefixes)
> 你應該使用能將成員變數凸顯或變色的編輯環境，來區分它跟其它程式碼的不同。
> 除此之外，人們會很快學會忽略字首 (或字尾)，只會看名稱中真正有意義的部份，最後，字首成為看不見的雜訊，以及古早程式碼的代表。

## 介面 (Interfaces) 和實作 (Implementations)
> 我偏好讓介面不加額外的修飾，在現今常見的多餘字首 `I`，是個讓人分心的事物。
> 如果必須將介面或實作兩者之一進行名稱編碼，我會選擇將實作編碼，稱之為 `ShapeFactoryImp`，或是醜陋的 `CShapeFactory`，都比對介面編碼好一些。

# 避免思維的轉換
> 讀者不應該還得將你取的名稱在腦中想一遍，翻成他們所熟知的名稱。
> 專業的程式設計師知道「清楚明白才是王道」。

# 類別的命名
> 類別和物件應該使用名詞或名詞片語來命名。
> 命名類別時應該避免 `Manager`、`Processor`、`Data`、`Info`。類別名稱也不應該是動詞。

# 方法的命名
> 方法應該使用動詞或動詞片語來命名。
> 建構子被 overloaded 時，請使用名稱中含有參數資訊的靜態工廠方法。

Algorithms + Data Structures = Programs，所以我們寫程式便是在各種方法 (algorithms) 中操作類別、物件 (data structures)。因此將物件使用名詞命名，而方法使用動詞命名會比較貼近人類的思維。

# 不要裝可愛
> 清楚闡述比起娛樂價值來得重要多了。

# 每個概念使用一種字詞
> 替單一抽象概念挑選一個字詞，並堅持持續地使用它。
> 函式的命名必須要獨一無二且具備一致性，如此你才不必進行額外的瀏覽就能選到正確的方法。

下列名稱代表相同意義，我想這便是此段所想表達的最佳反例了：
```
channel = mallid = MallID = MallId = mid = mall_id = AdvertiserId = AccountId = PublisherId
```

# 別說雙關語
> 避免使用同一個字詞代表兩種不同的目的。

書中舉了一個例子，假設有許多類別的 `add` 方法是用來相加或相連兩個現有的值，然後形成新的值。假設要再寫一個新的類別，此類別有一個方法，會將單一個參數放入一個集合容器裡，可以將之稱為 `add` 嗎？這兩個 `add` 的語意是不同的，所以應該要使用像 `insert` 或 `append` 之類的名稱。
作者用語意來告訴讀者什麼時候該保持一致，什麼時候別造出一個雙關語。我認為若難以區分的話，或許觀察執行完方法的結果會比較容易分別。

# 使用解決方案領域的命名
> 記住閱讀你程式的人也是程式設計師，所以盡量使用 Computer Science 領域的術語。

# 使用問題領域的命名
> 如果沒有程式設計師熟悉的術語可供命名的話，請使用該問題領域 (domain knowledge) 的術語命名。

這類問題可能在非軟體業比較會遇到，例如台積電的系統可能會有相當多晶圓製程的術語。

# 添加有意義的上下文資訊 (Context)
> 只有極少部份的命名能由命名本身了解到足夠的意義，大部份的命名都需放在上下文中，例如放在擁有良好名稱的類別、函式、命名空間裡。
> 當以上方法都無法表達恰當的意義時，在命名中加上字首就是最後必要手段了。

# 別添加無理由的上下文資訊
> 較小的名稱若能清楚地表達涵義，通常好過較長的名稱。盡量減少在名稱上加入不必要的內文資訊。

這段作者舉了一個例子：想像有個虛構的程式叫「Gas Station Deluxe」，在此程式裡把每個類別都加上了 `GSD` 字首，當你輸入了 `G` 再按下 auto complete 鍵，接著你獲得了系統裡所有類別的建議，這份建議表單看似有一哩長。為什麼要讓 IDE 沒辦法幫助你呢？
這部份我想最難拿捏的是，名稱若太短，便不足以表達出意圖，或是不足以與其他命名產生區別。但若太長又會產生無意義的區別。

(這種在每個類別都加上 prefix 的作法，在 Objective C 裡還真的有，若有寫過 iOS 的人便知道，裡面有滿滿的 `NS` 開頭的類別、方法、物件、變數。)

# 總結
> 挑選一個好名稱最困難的點在於，它需要良好的描述技巧及共通的文化背景。

我認為要做好這件事，必須訓練：描述技巧、拿捏分寸、釐清問題本質：
- 描述技巧：挑選合適的詞彙展現其意圖。
- 拿捏分寸：名稱過長過短都不好，一致性與雙關語的區別。
- 釐清問題本質：理解整份程式碼的架構、找出問題本質才有辦法替各類別、函式、變數命一個合適的命名，這些命名便隱含著他們之間的互動。
