---
title: "Clean Code Digest - Chapter 4 註解"
slug: "clean-code-chapter-4"
date: 2018-10-10T14:33:00+08:00
draft: false
image: "/content/images/2018/10/enoshima.jpg"
tags: ["digest"]
---

# Chapter 4 註解
# TL; DR
我認為此章節所想表達的有下列事項：
- 最後一道防線：作者認為註解是最後手段，在寫下註解前，你應該好好的審視程式碼是否能改寫得更好，盡量讓程式碼本身就能表達出足夠的含義
- 好的註解：這些註解都不應該解釋程式碼怎麼運作，而是為什麼要這麼做，或是不這麼做的後果會是什麼
- 壞的註解：可以從 git 得到的資訊就不需要再寫註解了。規定一定要寫下某些註解是很愚蠢的。別使註解干擾或誤導他人。

# Chapter 4 註解
> 「不要替糟糕的程式碼寫註解－－重寫它」－－Brian W. Kernighan and P.J. Plaugher
> 適當地使用註解是用來「彌補我們用程式碼表達意圖的失敗」。
> 一個註解存在越久，事實就越來越偏離當初的程式碼解釋。

此段落其實在告訴你不要寫註解，作者主要有底下兩個理由：
- 寫註解代表你的程式碼不足以表現意圖，你應該修改你的程式碼
- 程式碼總是一直被修改，於是註解與程式碼內容越差越遠，最後註解將會誤導你

# 註解無法彌補糟糕的程式碼
> 整潔具有表達力又極少使用註解的程式碼，遠優於雜亂複雜又滿是註解的程式碼。
> 與其花時間寫註解來解釋你所造成的混亂，不如花時間去整理那堆混亂的程式碼。

# 用程式碼表達你的本意
> 在大部份情況下，你想要寫下的註解，都可以簡單地融入到建立的函式名稱當中。

作者在這裡舉了一個例子：
```go
// Check to see if the employee is eligible for full benefits
if (employee.flags & HOURLY_FLAG) && (employee.age > 65)
```
```go
if employee.isEligibleForFullBenefits()
```

因為程式碼是唯一的真相，註解寫的再詳細清楚都有可能是背離事實的。如果能從程式碼表達出意圖，就再也不需要寫下註解來誤導人。

# 有益的註解
> 真正有益的註解，是你想辦法不寫它的註解。

![no_code_no_bug](/content/images/2018/10/no_code_no_bug.jpeg)
[Image reference](https://medium.com/@JohanneA/something-bugging-you-how-to-solve-and-avoid-bugs-46f4aa692ee3)

## 法律型的註解
> 內容不應該直接是契約或是法律條款。
> 讓註解去參考一個標準的許可或其他外部文件。

我自己待過的公司還沒有要求需要寫這種型式的註解，open source 我也沒有參與太多。現在的 GitHub 在開一個新專案時，都有選項讓你直接選這個專案要採用什麼樣的 license 了，我不懂的是都已經選了 license 了，應該不需要在每一份程式碼開頭都還加上 reference 吧？

## 資訊型的註解
> 透過註解提供一些基本資訊是非常有用的。

此處作者提了一個例子：
```go
// format matched kk:mm:ss EEE, MMM dd, yyy
pattern, err := regexp.Compile(
    "\\d*:\\d*:\\d* \\w*, \\w* \\d*, \\d*"
)
```

有時候我會也這麼做：例如要 parse 字串時，如果是比較簡單的 case 我便不會寫 regular expression，所以常常是 `strings.Split()` 後直接拿第 `n` 個位置的 element。這時候我便會在上面加一個註解，其內容便是一個實際要被 parsed 的字串範例。
若是用 regular expression 我也會像作者一樣附上一個實際範例或是格式。

## 對意圖解釋的註解
> 你也許不同意程式設計師對於這個問題的解法，但至少你可以了解他想要做什麼

這裡作者 (或是譯者) 同樣使用「意圖」這個字詞我覺得有點不太對。因為前面作者一直提到應該要把意圖放進程式碼、命名裡，但這裡卻又說要用註解解釋意圖。
而作者舉的例子是比較物件時的優先順序，以及在測試裡為了要得到 race condidtion 而產生了 25,000 條 threads。這裡所寫的註解都是為了解釋為什麼要這麼做 (意圖) 的原因，所以我不認為是對意圖的解釋。意圖代表著目的、想做到的事，而這裡的註解則是解釋過程、手段。

## 闡明
> 註解把難解的參數或回傳值翻譯成具有可讀性的文字，這種註解是有幫助的。
> 如果可以讓參數和回傳值自我闡述清楚其本意，會是一個更好的選擇。
> 萬一闡明性的註解並不正確的話，那就冒了相當大的風險。

作者一而再，再而三的強調可以讓程式碼本身表達其意圖，就不要寫註解了。這裡提到的是，如果參數或回傳值是標準函式庫裡的一部份，或是某些你無法修改的程式碼，那這時候加入一些註解就很有用了。

## 對於後果的告誡
> 能警告其他程式設計師會出現某種特殊後果的註解，也是有用的。

這段所講的跟上上段所提到的「對意圖解釋的註解」其實有點類似。例如你可能初始化一個 instance，而原因是該物件不是 thread safe，那麼加入註解警告其他人，同時也是解釋這裡為什麼要這麼做。
或許當下的程式碼不是最好的解法，但至少你可以讓別人理解你當時的想法。

## TODO (待辦事項) 註解
> 不管這樣的 `TODO` 目標是什麼，都不應該成為讓糟糕程式碼留在系統裡的藉口。
> 要定期地審視這些待辦事項，並且刪除已經不再需要的待辦事項。

這段說明了可以在註解加上 `TODO` 來解釋為什麼要使用比較糟的程式碼，或是未來要怎麼改寫。
比起是不是寫下正確的註解，我認為追蹤 `TODO` 事項並持續整理才是更重要的事情。

## 放大重要性
> 某些無關緊要的事情，可以透過註解來放大其重要性。

這段非常簡短，主要想說的東西我認為跟「對於後果的告誡」有點類似。有些看起來非常簡單的操作實際上是很重要的，不能隨便亂修改、刪除。

## 公共 API 裡的 Javadoc
> Javadoc 能像其它任何註解一樣，產生誤導性、非區域性及非誠實性的現象。

# 糟糕的註解
> 通常它們是爛程式的藉口，給爛程式撐腰，或是程式設計師的自言自語，只是為了不充份做出決定找個說詞。

作者其實還有提了一句話：「大部份的註解都屬於這一類」。我回頭看看我寫過的註解，除了 docstring 類的，大部份都比較像是一個 tag，讓我自己用來分區塊，或是回憶這一小段程式碼是做什麼用的。另外就是一些無法明顯看出這段程式碼是要做什麼的時候，我也會加上一些註解解釋。

## 喃喃自語
> 只因為你覺得應該、或是因為開發流程的要求，所以你就寫下一個註解，這是一種亂寫的行為。
> 任何未能與你做好溝通，強迫你去看其他套件的註解，都是一種失敗的註解，也不值得我們浪費空間保留它。

目前我的公司是有使用 `flake8` 的，在一開始 `flake8` 的設置檔只有簡單把單行長度限制改為 100 個字元，所以我的編輯器一天到晚在噴 docstring 的 warning。於是我便開始為了加 docstring 而加 docstring。
像是 `__init__.py` 裡我也沒特別做什麼事，我還是得幫他加一行註解。`class` 裡的 `__init__(self)` 我也必須幫他加，有誰不知道這是一個 constructor？(如果他的建立過程很複雜當然另當別論)。
最後在討論之後，我們決定把 docstring 相關的規定加進 ignore list 裡了。

## 多餘的註解
> 讀一段多餘的註解可能會比讀程式碼花上更多的時間。
> 有點像一個過度誇耀的二手車銷售員，向你擔保根本不需要檢查車蓋裡的東西。

作者在此段給了一個例子：
```java
// Utility method that returns when this.close is true. Throws an exception if the timeout is reached.
public synchronized void waitForClose(final long timeoutMIllis)
throws Exception
{
    if (!closed)
    {
        wait (timeoutMillis);
        if (!closed)
            throw new Exception("MockResponseSender could not be closed");
    }
}
```

雖然我有點看不懂這個例子，但我想作者想表達的是，如果註解無法帶來更多的解釋或好處，那就不要再多寫註解來使讀者更加混亂。

## 誤導型註解
> 有些程式設計師在出於好意的情況下，寫下了不夠精確的註解。

上面程式碼的註解寫說：`當` `this.closed` 為 `true` 時會 return，但實際上則是要再等一段時間才會 return (如果變為 `true` 的話)。

## 規定型註解
> 如果有規定是說，每個函式都必須有一個 Javadoc，或是每個變數都應該要有註解，那這條規定就有夠傻的。

Python 生態裡有滿強烈的 convention，要完整的遵守其實會滿辛苦且毫無意義的。公司到最後也是把一些實在太惱人的規則拿掉了。

## 日誌型註解
> 這些又臭又長的日誌，只會使得模組更雜亂和混淆，它們應該被移除掉。

這裡所說的日誌型註解就如同現在的 commit message 一樣，在很久之前並沒有原始碼管控系統，所以只好在每個模組的開頭加入日誌，以便說明在什麼時間做了什麼更改。
如今若還出現這種類型的註解，我建議你換間公司吧，連版本控制都不使用的公司，沒有待下去的必要。

## 干擾型註解
> 你會發現某些註解毫無用處，就只會干擾我們，它們重新陳述很明顯的事實，又沒有提供任何新的資訊。
> 所以我們都學會了忽略它們，最後當註解周邊的程式碼被修改之後，註解便開始成了謊言。

毫無用處的註解就如同脫了褲子放屁一樣，例如：
```python
class Employee:
    """Employee class."""

    def __init__(self):
        """Employee constructor."""
```

`class` 以及 `__init__` 底下的 docstring 都是有跟沒有一樣註解。作者進一步說這些註解不斷的干擾我們，最後我們便學會了忽略他們，修改程式碼後這些註解便與程式碼的內容不一致而變成謊言了。

## 可怕的干擾
> 有些註解一點意義都沒有。它們只是一些不應該被要求，但卻被要求提供說明文件所造成的多餘性干擾型註解。

作者從一個著名的 open source 擷取了一小段：
```java
/** The Name. */
private String name;

/** The version. */
private String version;

/** The licenceName. */
private String licenceNaame;

/** The version. */
private String info;
```

你有看出哪裡怪怪的嗎？
如果撰寫程式碼的作者，對於他寫的或複製貼上的註解一點也不用心，那你還能期待從註解得到什麼嗎？

## 當你可以使用函式或變數時就別使用註解
> 當你可以使用函式或變數時就別使用註解。

我認為這段的標題就足以說明一切了。
作者以底下例子作為範例：
```go
// does the module from the global list <mod> depend on the
// subsystem we are part of?
if smodule.getDependSubsystems().contains(subSysMod.getSubSystem())
```

```go
moduleDependees := getDependSubsystems()
ourSubsystem := subSysMod.getSubSystem()
if moduleDependees.contains(ourSubSystem)
```

作者認為與期寫一長串的註解，不如將其融入於程式碼當中。讓能程式碼獨立表現其意圖，就不需要再額外寫下程式碼了。

## 位置的標誌物
> 只有當使用效果非常顯著時才使用它們。
> 如果你過度使用，它們就會變成被你忽略的一種背景陪襯物而已。

這類型的東西我認為與 `TODO` 是很類似的，像是一個 index 一樣讓你可以快速找到想找的段落。但我這為必須要是常見且有意義的。

## 右大括號後面的註解
> 這對一個較長且有著深層巢狀結構的函式可能是有意義的。
> 當你覺得需要在結尾右大括號處留下註解，不如試著簡短你的函式來取代這樣的行為。

我自己是滿不喜歡在 `}` 後面加註解的，因為閱讀程式碼是由上而下，等我看到註解時我都把程式碼看完一遍了，總覺得沒有意義。
我比較喜歡把程式碼切成幾個小區塊，如果有需要的話，便在該區塊前加上註解說明。

## 出處及署名
> 沒有必要用小小的署名來汙染你的程式碼。
> 再一次強調，想要儲存此類的資訊，原始碼管控系統會是一個比較好的選擇。

現在大家都使用 git，就算不是至少也會是 SVN 之類的版本控制。但我自己有時候會沒有設定好 git 而導致 git commit 顯示的 username 會不一樣。
以下圖為例，可以很明顯看到 commit 的頭像 icon 就不一樣了，但其實都是我的 commit。其原因是我沒有設好 user 及 email，而 GitLab 看起來是使用 email 去比對他系統內 user。

![gitlab](/content/images/2018/10/gitlab.png)

PS. 設定 git username 及 email 的方法如下：
```bash
git config --local user.name "Chih Kai Yu"
git config --local user.email "kai.chihkaiyu@gmail.com"
```

## 被註解起來的程式碼
> 當別人看到被註解的程式碼時，會沒有勇氣刪除它們。
> 這些被註解掉的程式碼就像是一瓶壞掉的葡萄酒，在瓶底所沉澱的殘渣。

我自己不認為會沒有勇氣刪除它們，但我認為它們相當干擾閱讀程式碼。
作者提到只有在 60 年代時，這些被註解掉的程式碼可能是有用的資訊。很不幸的，現在已經 2018 年了，而且時間也只會往前走，若你還看到有這樣的東西，請先 `git blame` 看清楚是哪個低能兒再毫不猶豫的刪掉它們吧。

## HTML 型式的註解
> 它讓某個地方裡 (編輯器或 IDE 裡) 本來易讀的註解，變得難以閱讀。
> 把這些註解以適當的 HTML 裝飾起來應該是工具的責任，而不是程式設計師的責任。

目前我參與的其中一個 project 裡面就有這樣的註解，其目的是為了讓 [Swagger](https://swagger.io/) 能自動從中產生文件。一來這些 HTML 碼皆是人工刻的，二來的確很擾人。好在目前的編輯器或 IDE 都還能將註解區塊整個隱藏起來，不至於影響程式碼的閱讀。

```csharp
/// <summary>
/// 讀取目錄列表
/// </summary>
/// <remarks>
/// 讀取目錄列表，各目錄資訊包含目錄名稱、商品欄位、商品更新狀態等資訊，
/// 目錄列表主要用於查看目前各目錄的狀態總覽。
/// <table>
/// <tr><th>Parameter</th><th>傳入位置</th><th>Type</th><th>值域限制</th><th>Required</th><th>Description</th><th>Default Value(Behavior)</th></tr>
/// <tr><td>fields</td><td>query string</td><td>string</td><td></td><td>Optional</td><td>欲顯示的目錄欄位，以逗號分隔</td><td>空字串(show all)</td></tr>
/// <tr><td>include_deleted</td><td>query string</td><td>boolean</td><td></td><td>Optional</td><td>是否顯示已下架的目錄</td><td>false</td></tr>
/// <tr><td>page_size</td><td>query string</td><td>integer</td><td>不大於10000的正整數。超過時，設為10000</td><td>Optional</td><td>回傳目錄數量</td><td>10000</td></tr>
/// </table>
/// </remarks>
/// <param name="fields"></param>
/// <param name="include_deleted"></param>
/// <param name="page_size"></param>
```

## 非區域型的資訊
> 如果你非得要寫註解，那就要確保它只會用來描述附近的程式碼。
> 不要在一個區域性的註解裡，提供整個系統的全域資訊。

作者舉了一個例子：
```java
/**
 *  Port on which fitnesse would run. Defaults to <b>8082</b>.
 *  @param finessePort
 */
public void setFitnessePort(int fitnessePort)
{
    this.fitnessePort = fitnessePort;
}
```

上面程式碼的註解是個很可怕的冗餘，它告訴你預設 port 的資訊，與該函式完全離題。想當然爾，這裡也沒有保證含有預設值資訊的程式碼被修改時，這裡的註解也會跟著修改。

## 過多的資訊
> 不要把一些有趣的歷史討論紀錄，或是一些不相關的細節描述放進你的註解裡。

給個 link 或是 keyword 就好，不需要把整份文件放進你的程式碼註解裡。

## 不顯著的關聯
> 註解及被描述的程式碼之間的關聯應是顯而易見的。
> 如果連註解本身都需要額外的解釋，那真是一件遺憾的事。

作者舉了一個例子，該段程式碼是從 apache 取得：
```java
/*
 * start with an array that is big enough to hold all the pixels
 * (plus filter bytes), and an extra 200 bytes for header info
 * /
this.pngBytes = new byte[((this.width + 1) * this.height * 3) + 200];
```

作者提出一些疑問：`plus filter bytes` 是什麼？和 `+1` 有關係嗎？或者跟 `*3` 有關係？一個像素是用一個 byte 表示嗎？為什麼是 `200`？
雖然我有時候會這為這或許跟 domain knowledge 有關，這個在該領域說不定是接近 common sense 的事情。

## 函式的標頭
> 替只做一件事的小型函式挑選一個好一點的名稱，通常比「將註解寫在函式標頭」來得更優。

## 非公共程式碼裡的 Javadoc
> 拘泥在 Javadoc 所要求的額外格式，通常只會產生一些無關緊要又不想要的廢物，分散人們的注意力而已。

有時候真的不需要完完全全照著 `pylint` 或是 `flake8` 的預設 config，團隊內自己討論好，有一個共識即可。
非常不喜歡討論這種事情的人可以試著寫 Golang，因為這一種直接將 format/linter 寫在 compiler 的語言。
