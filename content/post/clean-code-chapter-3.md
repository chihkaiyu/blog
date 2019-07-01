---
title: "Clean Code Digest - Chapter 3 函式"
slug: "clean-code-chapter-3"
date: 2018-09-27T23:18:56+08:00
draft: false
image: "/content/images/2018/09/DSC04988.jpg"
tags: ["digest"]
---

# Chapter 3 函式
# TL; DR
我認為此章節所想表達的有下列事項：
- 簡短：整章其實都圍繞著「簡短」打轉，像是函式只做一件事，且是同一層抽象概念的事，一直到不要重複自己都是在想辦法盡量讓函式 (程式碼) 簡短。
- 故事：一份程式碼是在描述系統的故事，函式是動詞，類別是名詞，函式訴說著他對系統裡的東西做了什麼事。良好的命名及漂亮的結構最後整潔的結合在一起，便能順暢的說出系統的故事。

# 簡短！
> 關於函式的首要準則就是要簡短。第二項準則就是要比第一項更簡短。
> 每個函式都一清二楚，透露出本身的意圖。每個函式帶領著你至下個函式，這就是經式該有的簡短。

這段直接破題告讓你函式就是要短，而且是毫無研究上的支持，完全是作者自己的經驗談。我們也知道看一個長度 50 行的函式與 500 行的函式在閱讀起來是截然不同的感受。我在拆函式的習慣是滿隨性的，我會將主要流程分為幾個步驟，將各個步驟都寫成一個函式。
這段另外提到了區塊 (blocks) 及縮排 (indenting)，他提到條件敘述應該只有一行，且那行便是函式呼叫。這不僅維持封閉式的簡短，若函式名稱取得好還能用來描述意圖。而這也意味著函式不應該大到包含巢狀結構，因此函式裡的縮排不應該大過一或兩層。但這件事我認為在 Python 裡非常難做到，尤其  Python 又是以縮排作為 blocks 的區別。

# 只做一件事情
> 函式應該做一件事情，它們應該把這件事做好，而且它們應該只做這件事。
> 如果函式只做了函式名稱下「同一層抽象概念」的幾個步驟，那這個函式就算是只作了一件事。
> 做一件事的函式沒有辦法被合理地分成不同的段落。

這段最難拿捏的就是其中的「只做一件事」，該怎麼定義「一件事」？作者以底下程式碼為例：
```go
func RenderPageWithSetupsAndTeardowns(pageData PageData, isSuite bool) string {
    if isTestPage(pageData) {
        includeSetupAndTeardownPages(pageData, isSuite)
    }
    return pageData.getHtml()
}
```
這段程式碼只做一件事嗎？還是做了三件事？
作者的判斷標準在於「同一層抽象概念」。作者給的定義則是：看你是否能夠從函式中提煉出另外一個新函式，但此新函式不能只是重新詮釋原函式的實作而已。
底下程式碼是我的 side project 的其中一個函式，很明顯的可以看到有三個段落，甚至我自己還寫了註解：
```go
func updateAllUsers() error {
	// get users from db
	stmt, err := db.Prepare("select email from User")
	if err != nil {
		return err
	}
	defer stmt.Close()
	rows, err := stmt.Query()
	if err != nil {
		return err
	}
	existedUser := make(map[string]bool)
	for rows.Next() {
		var e string
		err = rows.Scan(&e)
		if err != nil {
			return err
		}
		existedUser[e] = true
	}

	// combine users by email
	userInfos := make(map[string]userInfo)
	err = getAllGitLabUser(userInfos)
	if err != nil {
		return err
	}

	err = getAllSlackUser(userInfos)
	if err != nil {
		return err
	}

	// insert into db
	for email, user := range userInfos {
		// skip already existed row
		if _, exist := existedUser[email]; exist {
			continue
		}
		insert, err := db.Prepare("insert into User(email, slackID, gitlabID, name) values(?, ?, ?, ?)")
		log.Infof("Found new user %v", user.name)
		if err != nil {
			return err
		}
		defer insert.Close()
		res, err := insert.Exec(email, user.slackID, user.gitlabID, user.name)
		if err != nil {
			return err
		}
		lastID, err := res.LastInsertId()
		if err != nil {
			return err
		}
		rowCnt, err := res.RowsAffected()
		if err != nil {
			return err
		}
		log.Debugf("ID = %d, affected = %d", lastID, rowCnt)
	}
	return nil
}
```
上面的函式叫作 `updateAllUsers` 所以可以想像大概流程就是從某個地方讀取資料，再更新到某個地方，那我們應該可以再將各段落拆成函式變成底下的樣子：
```go
func updateAllUsers() error {
	// get users form db
	existedUsers, err := getExistedUsers()
	if err != nil {
		return err
	}

	// combine users by email
	newUsersInfos, err := getNewUsers()
	if err != nil {
		return err
	}

	// insert into db
	err = insertNewUsers(newUsersInfos, existedUsers)
	if err != nil {
		return err
	}
	return nil
}
```
我自己的習慣是會將每個函式當作是 step 1、step 2…的概念，所以第一層函式常常做「不同層抽象概念」的事，子函式便比較像書中所說的「只做一件事」。下一段則會提到函式裡的層次概念。

# 每個函式只有一層抽象概念
> 一個函式擁有混合層次的抽象概念總是令人困惑。
> 讀者會無法分辨某個 expression 是一個基本概念還是一個細節。
> 更糟糕的是就像破窗效應一樣，一旦將細節和本質概念混在一起，就會有越來越多的細節雜處於函式裡。

當我在試著修改上一個段落所提到的函式 `updateAllUsers` 時，修改完我還是覺得怪怪的，因為我的函式還是有三個步驟：讀入現有 users、得到新的 users、更新至 db。直到看完這段我才明白這樣做是沒有問題的，因為我一直維持在同一層抽象概念裡。
作者分別舉了高、中、低三個層次的範例，高層次概念：`getHtml()`，中層次概念：`String pagePathName = PathParser.render(pagePath)`，低層次概念：`.append("\n")`。對應到上一個段落的程式碼，我最終修改完的大約會是在中、高層次裡，所以讀起來還算通順並不會覺得混亂。
這裡所要在意的是函式內容有沒有維持在同一層抽象概念，而不是只做一件事。同一個層次便是一件事，而不是呼叫其他函式的次數。
作者將這樣的敘事方式稱為「降層準則」，也就是每個函式後面都緊接著下一層次的抽象概念

# Switch 敘述
> `switch` 本質上總是在做 N 件事情，但我們能確保讓每個 `switch` 都被深埋在較低抽象層次的類別裡。
> 將 `switch` 埋在 Abstract Factory 的底下。

這段開頭便說了要讓 `switch` 簡短是很難的事 (一連串的 `if/else` 也是)，因為他本來就是做超過一件事。避免的方法便是將 `switch` 藏起來且不被重複使用。作者舉了一個例子：
```go
func calculatePay(e Employee) (Money, error) {
    switch v := e.(type) {
    case COMMISSIONED:
        return calculateCommissionedPay(e)
    case HOURLY:
        return calculateHourlyPay(e)
    case SALARIED:
        return calculateSalariedPay(e)
    default:
        return fmt.Errorf("Invalid employee type: %v", v)
    }
}
```
上述程式碼有幾個問題：
1. 它太冗長，如果再加入新的職員型態，這個函式會變得更長。
2. 這個函式做超過一件事情。
3. 有超過一個以上的理由來改變此函式，所以它違反單一職責原則 (SRP)。
4. 當新型態加入此函式就必須改變，所以它違反開放閉合原則 (OCP)。

作者提供的解法是將此 `switch` 敘述埋在 Abstract Factory 底下，不讓任何人看到它。這個工廠會使用 `switch` 產生適當的 `Employee` 實體。
```go
type Employee interface {
	isPayDay() bool
	calculatePay() Money
	deliverPay(pay Money)
}

type EmployeeFactory interface {
	CreateEmployee(r EmployeeRecord) Employee
}

type EmployeeFactoryImpl struct {}

func (e *EmployeeFactoryImpl) CreateEmployee(r EmployeeRecord) Employee {
	switch er := r.(type) {
	case COMMISSIONED:
		return new(CommissionedEmployee)
	case HOURLY:
		return new(HourlyEmployee)
	case SALARIED:
		return new(SalariedEmployee)
	default:
		return fmt.Errorf("Invalid employee type: %v", er)
	}
}

type CommissionedEmployee struct {}
type HourlyEmployee struct {}
type SalariedEmployee struct {}
```

# 使用具描述能力的名稱
> 只要函式越簡短、越集中在做該做的事，就越容易替函式取個具有描述性質的名稱。
> 一個較長但具備描述性質的名稱，比一個較短但難以理解的名稱還要好。
> 使用一致的片語、名詞與動詞來替函式取名。

這裡所要表達的我想與前一章：[有意義的命名](https://chihkaiyu.gitlab.io/2018/09/19/clean-code-chapter-2/)是同樣的概念：表現意圖、一致性。

# 函式的參數
> 函式參數數量越少越好，盡量避免三個以上的參數。
> 參數會混淆讀者，例如 `includeSetupPage()` 比 `includeSetupPageInfo(newPageContent)` 更容易理解。
> 越多參數在撰寫測試越困難，因為要涵蓋所有參數可能組合。
> 輸出型參數比輸入型參數更難理解。

## 單一參數的常見形式
此段落舉了幾個單一參數的常見使用方法：
1. 與該參數有關的問題，例如 `func fileExists("MyFile") bool`
2. 對參數進行某種操作，將該參數轉換成某種東西然後回傳，例如 `dat, _ := ioutil.ReadFile("Myfile")`
3. 事件 (event)，這類型會有一輸入型參數，沒有任何輸出型參數

作者特別指出，如果不符合以上形式，試著不要使用單一參數函式。另外也提到了如果有發生轉換時，轉換後的產物應該出現在回傳值裡，例如 `func transform(in StringBuffer) StringBuffer` 會比 `func transform(out StringBuffer)` 來得好，即便前者只是回傳輸入參數。

## 旗標 (flag) 參數
> 使用旗標參數是一種非常爛的做法。
> 這馬上使函式變得複雜，等同於大聲宣布此函式做了不止一件事。當是 `true` 時做了一件事，`false` 時又做了另一件事。

You heard me. `render(true)` sucks.

## 兩個參數的函式
> 有兩個參數的函式會比單一參數更難理解。
> 有時候兩個參數是恰當的，例如直角座標系上的點，本質上就是需要兩個參數。
> 如果有機制可以將兩個參數轉換成一個參數時，你就應該好好的利用。

越多參數越難理解這是一定的，我自己在看多個參數的的函式時，就像是電腦在 cache 東西一樣，我把參數的名稱、型態、意圖 cache 在我腦中，同時還要看函式的邏輯，大概超過三四個就很容易放棄閱讀了。
作者另外說明了某些情況下使用兩個參數是合理的，像是平面座標上的點。但他提出了另一個觀點，兩個參數是「有序元件組成的單一值」，意思就是兩個數參的順序及組合要是有意義的。即便像是 `assertEquals(expected, actual)` 明顯會有兩個參數的函式，也是常常會搞混 `actual` 及 `expected` 的位置，因為這個函式的先後順序是要經過訓練才能學會的 (我認為就是熟悉度、習慣)。但平面座標的表示方法是我們非常習以為常的，所以如果有一個函式為 `plot(x, y)`，我們非常輕易就能夠理解其意圖，因為這兩個參數是自然的組合，也是自然的順序。

## 三個參數的函式
> 我建議你在建立三個參數的函式之前應該謹慎地思考。
> 介紹一個沒有那麼陰險的三參數函式：`assertEquals(1.0, amount, .001)`，檢查兩個浮點數在某個精確度下是否相等。

雖然作者再三警告不要設計超過兩個參數的函式，但就我看過的 code 以及寫過的 code，這實在是太難達成了。比較常遇到的情形是無法合理的將意圖相近的東西組合成一個類別，導致必須傳一堆參數。
或許先專注在每個函式只做一層抽象概念開始，便能減少參數的數量。

## 物件型態的參數
> 當一個函式需要超過兩個的參數時，很可能需要將其中一些參數包裝在一個類別裡。

作者舉了一個例子：
```go
func makeCircle(x, y, radius double) Circle {}
func makeCircle(center Point, radius double) Circle {}
```
他指出將變數 `x` 及 `y` 包裝在 `Point` 類別看起來像作弊，但其實不然，因為 `x` 與 `y` 是某個概念裡的相似部份，這個概念應該獲得一個屬於它的名稱。

## 參數串列
> 如果可變數量的參數都被同等看待，那它們就和型態為 `List` 的單一參數等價。

此處考慮的面向與物件型態參數是不同的，上面是將相似的東西包裝在一個概念裡，這裡所提到是同質的參數。

## 動詞和關鍵字
> 在單一參數裡，函式和參數要形成一個動詞/名詞的良好配對。
> 關鍵字型式的函式裡，我們將「參數的名稱」編碼加入到函式名稱裡。

函式內容是在描述邏輯與參數的互動，若能用函式的名稱及參數表達出意圖、參數順序性意義在可讀性上便會大大提升。例如：
```go
writeField(name)
assertExpectedEqualsActual(expected, actual)
```
前者告訴我們 `name` 是一個欄位，並將此名稱寫入；後者告訴我們參數順序性的意義。雖然我覺得後者有點太過頭了，現今的 IDE 應該都能 peek 函式的用法、docstring。

# 要無副作用
> 副作用 (Side effects) 就像謊言一樣，你的函式暗地裡偷偷做了其他事。
> 使同類別其他變數產生不可預期的改變；有時候將之轉換成參數傳給其他函式，常會導致奇怪的時空耦合 (temporal coupling) 和順序相依性的問題。

前面一點的段落有提到函式應該只做同一層抽象概念的事，若不這樣做，則會在奇怪的地方 (空間) 或奇怪的時間 (時間) 發生奇怪的事情 (耦合)。作者有提出，若你必須有一個時空耦合，那應該在函式的名稱中說明清楚。

作者舉了以下例子：
```go
func checkPassword(userName string, password string) bool {
    user := UserGateway.findByName(userName)
    if user != User.NULL {
        codedPhrase := user.getPhraseEncodedByPassword()
        phrase := cryptographer.decrypt(codedPhrase, password)
        if phrase == "Valid Password" {
            Session.initialize()
            return true
        }
    }
}
```
上面例子在哪裡發生了時空耦合？答案是 `Session.initialize()`，函式名稱並沒有告訴我檢查密碼同時還會初始化 session。若真的要這麼做，作者建議將函式重新命名為 `checkPasswordAndInitializeSession` 會比較好。

## 輸出型的參數
> 要避免使用輸出型參數，如果你的函式必須要改變物件的某種狀態，就讓該物件改變自己的狀態吧。

這裡提到的是，我們已經習慣參數是輸入到函式裡。當遇到一個輸出型的參數而非輸入型時，我們的思考便會被中斷，被迫去檢查該函式的宣告。
作者提及在物件導向程式設計尚未出現的日子，有時候是需要輸出型的參數，但在物件導向出現後 `this` 便有預謀地扮演了輸出型參數的角色。

# 指令和查詢的分離
> 函式應該要能做某件事或能回答某個問題，但兩者不該同時發生。
> 真正的解決方法是將指令 (command) 和查詢 (query) 分開。

這段提到的概念我非常喜歡，這是我以前從來沒想過的。作者舉了例子：
```go
func set(attribute string, value, string) bool {}
```
上面函式相當簡單，將某個屬性設定為某個值，成功回傳 `true`，失敗則回傳 `false`。再看看底下例子：
```go
if set("username", "unclebob") {}
```
試問上面敘述代表什麼意思？是問 `username` 在之前是否已經被設為 `unclebob` 嗎？還是 `username` 是否成功被設定為 `unclebob`？因為出現了 `if`，所以讓 `set` 「感覺」像是形容詞，但事實上他是個動詞。
即便將函式重新命名為 `setAndCheckIfExists` 也無法增加 `if` 的可讀性，將指令和查詢分開才是正確的做法。例如：
```go
if attributeExists("username") {
    setAttribute("username", "unclebob")
}
```

條件敘述裡放的是條件 (condition)，所以其內容應該要是 query 的結果，將要執行的 command 放進其區塊裡。

# 使用例外處理取代回傳錯誤碼
> 要指令型函式回傳錯誤碼有點違反指令和查詢分離的原則，這代表鼓勵在 `if` 判別處，將指令型函式當作判斷表達式使用。
> 這會導致更深層的巢狀結構。
> 當你回傳一個錯誤碼，就是要求呼叫者馬上處理這個錯誤。

有回傳錯誤碼，你就必須要檢查，否則回傳就沒有意義了不是嗎？但如此檢查就會導致很深的巢狀結構。作者鼓勵你使用「例外處理」取代「錯誤碼」。這與 Python 生態的風格非常像，Python 生態裡鼓勵你使用 EAFP (Easier to Ask for Forgiveness than Permission)，甚至是用例外處理當作流程控制。而另一派是 LBYL (Look Before You Leap)，便是這裡所說的檢查錯誤碼了。
例如我曾寫過一支 Python 程式，是要抓取圖片並簡單影像處理：
```python
def worker(job_queue):
    """Fetch and resize image worker."""
    while 1:
        try:
            # Fetch and resize image here
        except constant.NoMallProductException as err:
            logger.error(err)
        except constant.NoImageURLException as err:
            logger.error(err)
        except constant.FetchImageException as err:
            logger.error(err)
        except constant.ResizeImageException as err:
            logger.error(err)
        except constant.DatabaseException as err:
            logger.error(err)
        except constant.ProductAPIException as err:
            logger.error(err)
        except KeyboardInterrupt:
            logging.info('Stop!')
            sys.exit(0)
        except Exception as err:
            logger.critical(err)
```
但我也聽說在 C# 裡 try/catch 的 overhead 是很大的，我便不知道在這類型的語言下該如果做好這件事。

(附帶一提，這裡我不使用 Golang 作為範例程式碼是因為 Golang 裡並沒有 try/catch 語法。Golang 生態習慣回傳 `error`，再檢查他是否為 `nil`，若不檢查則容易引發 `panic` 而導致程式 crash。所以 Golang 的程式碼裡會出現滿滿的檢查 `error`：
```go
if err != nil {
    // error handling here
}
```

## 提取 Try/Catch 區塊
> Try/catch 區塊本身是難看的。

## 錯誤處理就是一件事
> 函式應該只做一件事，而錯誤處理就是一件事。
> 如果關鍵字 `try` 存在於一個函式裡，它幾乎就是該函式的開頭字眼，而且在 `catch/finally` 區塊後，理當不應有其他程式碼。

作者認為 try/catch 區塊是難看的，他認為應該要將 try/catch 區塊從函式中提取出來。我看完作者舉的例子後，我認為反過來說也是可以：將 try/catch 區塊裡的邏輯放到另一個函式裡，原函式便只處理例外。其例子為 (我改寫為 Python)：
```python
def delete(page):
    try:
        deletePageAndAllReferences(page)
    except Exception as err:
        logError(err)

def deletePageAndAllReferences(page):
    deletePage(page)
    registry.deleteReferences(page.name)
    configKeys.deleteKey(page.name.makeKey())

def logError(e):
    logging.error(err)
```

## Error.java 的依附性磁鐵
> 定義所有錯誤碼的類別或列舉 (enum) 若有所改變，則其它相關的類別就必須重新編譯及重新部署。
> 使用例外處理而非錯誤碼，新的例外可以由例外類別衍生出來，不必被迫重新編譯和重新部署。

作者在這段的註解特別說明一件事，新的例外可以由例外類別衍生出來是開放封閉原則 (OCP) 的範例。開放封閉原則 (Open-Closed Principle) 意思是，程式應該對擴充是開放的，而對修改是封閉的。若你定義了錯誤碼，在要新增錯誤碼時就得修改錯誤碼類別或 enum；反過來說，若你採用例外處理，則你可以很輕易的從例外基礎類例衍生出新的例外類別，這便是擴充。

# 不要重複自己
> 重複程式碼也許是軟體裡所有邪惡的根源。
> DRY (Don't Repeat yourself) 原則。

這段我想非常容易理解，這也大概是剛學習寫程式時能把握的原則：若程式碼重複兩次以上便抽出來寫成函式。一來程式碼不那麼擁擠，二來修改時只要修改一個地方便可。
作者另外提到許多準則或慣例都是為了控制或移除重複程式碼。例如資料庫的 Codd's normal forms 是為了消除資料的重複；物件導向程式設計將程式碼集中到基本類別，避免冗餘。結構化程式設計、剖面導向程式設計 (Aspect Oriented Programming)、元件導向程式設計 (Component Oriented Programming) 都包含一些減少重複程式碼的策略。

# 結構化程式設計
> Dijkstra 說道，每個函式及其裡面的區塊，都應該只有一個進入點及一個離開點。代表一個函式裡只能有一個 `return`，迴圈內不能有任何的 `break` 或 `continue`，而且永遠不可以有 `goto`。
> 如果你能保持函式的短小，那偶爾出現 `return`、`break`、`continue` 並沒有壞處。

# 你要如何寫出這樣的函式？
> 先直接把想法寫下來，然後開始琢磨，直到讀起來很通順。
> 當函式符合本章所提到的準則時，便可以結束這個函式的修改。我並不會從一開始就這麼寫，我也不認為有人可以辦得到。

作者在這段以寫作比喻寫程式，一開始一定是很粗糙且雜亂無章，接下來不斷的修改、重新組織直到你想要的樣子。

# 總結
> 函式是語言裡的動詞，類別則是語言裡的名詞。系統的函式和類別，是需求文件裡名詞和動詞的猜測。
> 永遠不要忘記你真正的目標是描述系統的故事。

程式語言雖然是將人類的想法翻譯給電腦，但同時也是在把你的想法以另一種文字的方式呈現出來，所以必定會有人去閱讀你寫的程式碼。最重要的目標是讓你對於系統的想法能明確傳達給他人知道。