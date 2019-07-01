---
title: "Clean Code Digest - Chapter 5 編排"
slug: "clean-code-chapter-5"
date: 2018-10-11T00:49:10+08:00
draft: false
image: "/content/images/2018/10/DSC04561.jpg"
tags: ["digest"]
---

# Chapter 5 編排
# TL; DR
我認為此章節所想表達的有下列事項：
- 編排的目的：作者認為編排會影響可讀性，而可讀性會不斷對日後的改變產生影響。甚至整本書其實都是在討論這件事
- 垂直：程式碼由上而下閱讀，應該像報紙一樣，由概要、概念、細節逐一深入。而程式碼之間的垂直距離以宣告的地方、使用的地方、本質來決定
- 水平：水平的空白間隔用來區分事情，例如 `=` 前後加入空白使其更突出，或是用來強調運算子的優先權。另一個用途是縮排，縮排決定了視野，而視野讓程式設計師可以快速替程式區塊作分類

# 編排的目的
> 程式編排是很重要的。
> 程式碼的可讀性，將會對以後每個可能的改變，產生深遠的影響。

這裡提編排是很重要，理由是會影響程式碼可讀性；程式碼可讀性很重要，理由如同前幾章提到的，可讀性會影響該程式碼是否易於修改。
有一派人會主張程式只要能正常運作就好，討論一堆枝微末節的小事根本是浪費時間。但我認為就是要讓程式正常運作，才需要討論這些小事，除非你認為程式碼隨便寫一寫就可以正常運作了。

# 垂直的編排
> 簡短的程式碼往往比大型的程式檔更容易讓人理解。

作者分析了七個 Java 的 open source 專案的程式碼檔案長度，如下圖：
![IMG_20181010_212415](/content/images/2018/10/IMG_20181010_212415.JPG)

細線的上下兩端分別代表最長行數及最短行數，而方塊則是上下半個標準差，所以方塊的中間便是平均值。
作者想傳達的是，要建造一個大型系統，程式碼不必很長就可以辦到，不用動不動就上千行。

## 報紙的啟發
> 在原始檔的最上方，能夠提供高階的概念和演算法，在往下閱讀的時候，程式的細節會慢慢地呈現，直到發現原始檔中最低階的函式及細節為止。

作者此處以報紙文章的書寫方式為例：報紙的頭條告訴你整篇報導在談論什麼。接著第一段描述整篇報導的概要，再來慢慢的詳述細節，最後你讀到所有日期、姓名、引言、主張等內容。
我認為的層次大約是：概要 -> 概念 -> 細節。在 [chapter 3 函式](https://chihkaiyu.gitlab.io/2018/09/27/clean-code-chapter-3/)裡有提到降層準則，我認為非常適合以這種模式去分層，同時每一個函式只做同一層次的事情。

## 概念間的垂直空白區隔
> 每一段程式碼都代表一個完整的思緒，應該用空白行來分隔這些思緒。

在閱讀本書前，我自己 coding 時便常常做這件事，例如底下程式碼：
```go
func main() {
	err := initConfig()
	if err != nil {
		log.Fatal(err)
	}

	err = initDatabaseConnection()
	if err != nil {
		log.Fatal(err)
	}

	err = initDatabase()
	if err != nil {
		log.Fatal(err)
	}

	log.Info("Updating users from Slack and GitLab...")
	err = updateAllUsers()
	if err != nil {
		log.Error(err)
	}

	log.Info("Updating GitLab Projects...")
	err = updateAllGitLabProject()
	if err != nil {
		log.Error(err)
	}

	log.Info("Starting cronjob scheduler...")
	err = startCronjob()
	if err != nil {
		log.Error(err)
	}

	log.Info("Starting server...")
	router := gin.Default()
	router.POST("/", handler)

	// user related routers
	router.GET("/user/:email", getUser)
	router.PUT("/user/:email", updateUser)

	// project related routers
	router.GET("/project/:namespace/*path", getProject)
	router.PUT("/project/:namespace/*path", updateProject)

	// update all users or projects
	router.POST("/users", allUser)
	router.POST("/projects", allProject)

	router.Run()
	// TODO: close database connection
}
```

此函式是我用 Golang 寫的一個 web server，在準備接受 request 前得先做一連串的事情，每件事情中間都會有一行空行隔開。若是拿掉空行，便如下：
```go
func main() {
	err := initConfig()
	if err != nil {
		log.Fatal(err)
	}
	err = initDatabaseConnection()
	if err != nil {
		log.Fatal(err)
	}
	err = initDatabase()
	if err != nil {
		log.Fatal(err)
	}
	log.Info("Updating users from Slack and GitLab...")
	err = updateAllUsers()
	if err != nil {
		log.Error(err)
	}
	log.Info("Updating GitLab Projects...")
	err = updateAllGitLabProject()
	if err != nil {
		log.Error(err)
	}
	log.Info("Starting cronjob scheduler...")
	err = startCronjob()
	if err != nil {
		log.Error(err)
	}
	log.Info("Starting server...")
	router := gin.Default()
	router.POST("/", handler)
	// user related routers
	router.GET("/user/:email", getUser)
	router.PUT("/user/:email", updateUser)
	// project related routers
	router.GET("/project/:namespace/*path", getProject)
	router.PUT("/project/:namespace/*path", updateProject)
	// update all users or projects
	router.POST("/users", allUser)
	router.POST("/projects", allProject)
	router.Run()
	// TODO: close database connection
}
```

我自己兩種格式比對後，很明顯發現把空白行拿掉會讓我不知道哪邊會是一個小斷點，有點像是一段文章裡沒有句號的感覺一樣。
我相信，一定會有人自以為是的說：「我覺得兩種都一樣啊。」

## 垂直密度
> 如果垂直空白區分開了各個概念，那麼垂直密度則意味著密切相關的程度。

作者主張越相關的程式碼在垂直距離上要越接近，底下是作者舉的例子：
```java
/**
 * The class name of the reporter listener
 */
private String m_className;

/**
 * The properties of the reporter listener
 */
private List<Property> m_properties = new Arraylist<Property>();

public void addProperty(Property property) {
    m_properties.add(property);
}
```

上述程式碼的兩個變數被註解隔開了，若將註解拿掉則會好讀許多。但我這為這與 syntax highlight 有關，目前的編輯器或 IDE 都會針對註解做不一樣的變色，所以你還是能輕易的過濾出想看的東西。

## 垂直距離
> 因為你試著想要了解整個系統到底在做什麼，但你卻花上大把的時間及力，想辦法找出和記住這些片斷的程式碼放在哪裡。
> 本段所描述的準則，也是避免使用 `protected` 變數的理由之一。

此段落分了三個面向：
- 變數宣告 (Variable Declarations)：變數的宣告應該盡可能靠近變數被使用的地方
- 實體變數 (Instance Variables)：實體變數應該被宣告在類別的上方。最重要的是，實體變數應該被宣告在大家都熟悉的地方
- 相依的函式 (Dependent Functions)：如果某個函式呼叫了另一個函式，那這兩個函式在垂直的編排上要盡可能地接近
- 概念相似性 (Conceptual Affinity)：某些程式碼在概念上有著相似的性質，希望能和其它程式碼盡可能地相近

這裡大概以底下幾種面向作為分類的依據：宣告的地方、使用的地方、本質。

## 垂直的順序
> 我們希望函式呼叫呈現一種向下的相依性。
> 由上往下查看原始碼時，可以依序發現高層模組，再接著找到低層模組。

# 水平的編排
> 我並不反對將最大寬度設在 100 字元，或甚至是 120 字元。

下圖仍是以前面段落所說的七個 open source 專案的分布：
![IMG_20181010_212449](/content/images/2018/10/IMG_20181010_212449.JPG)

可以看到在 80 個字元後線性下降，但 Y 軸的刻度是對數，所以下降幅度其實是相當大的。
Python PEP8 的建議長度是 79 個字元，但我認為這根本辦不到。即使公司的設定是 100 個字，我也常常因為超過 100 個字元的關係，而一直修改程式碼的編排格式。

## 水平的空白間隔和密度
> 我們利用水平的空白來區分事情。
> 空白的另一種用途是強調運算子的優先權。

水平空白在 Python 的 PEP8 裡倒是有很詳盡的規定，我自己也滿認同的，例如運算符號前後加上空白，參數列逗號後加空白等等。
我的想法是，程式碼可以想成是另一種人類的自然語言，它不單只是給電腦看的，同時也是給其他人閱讀的。所以若有適當的空白、符號等，一定能幫助我們閱讀。
試想以下文字你讀起來的感覺是暢快的嗎？

```
水平空白在 Python 的 PEP8 裡倒是有很詳盡的規定 我自己也滿認同的例如運算符號前後加上空白參數列逗號後加空白等等
我的想法是程式碼可以想成是另一種人類的自然語言 它不單只是給電腦看的 同時也是給其他人閱讀的 所以若有適當的空白 符號等 一定能幫助我們閱讀
試想以下文字你讀起來的感覺是暢快的嗎
```

## 水平的對齊
> 這樣的對齊模式一點幫助也沒有。這樣的對齊強調了不該注意的東西。
> 在一連串的宣告裡，你會不自覺被吸引去由上而下閱讀著一連串的變數名稱，而忽略了它們的型態。
> 在一連串的設定敘述裡，你會被吸引由上而下的看著一連串的右值，而忽略了設定運算子。
> 如果我有較長的列表需要被對齊，問題會出在列表的長度，而不是缺乏對齊。

作者以底下程式碼作為例子 (我改寫為 Golang 且內容為我自己的 side project)：
```go
type config struct {
	slackToken   string
	slackURL     string
	slackWebhook string
	gitlabToken  string
	gitlabURL    string
	dbFile       string
	token        string
}

cfg := config{
	slackToken:   slackToken,
	slackURL:     "https://slack.com/api",
	slackWebhook: webhook,
	gitlabToken:  gitlabToken,
	gitlabURL:    gitlabURL,
	dbFile:       dbFile,
	token:        token,
}
```

這種對齊方式是 Golang 的 convention (不這樣對齊一樣可以 compile 過)，其實我滿喜歡宣告時對齊的，但作者似乎不這麼認為。
作者所提到的閱讀順序，以及會忽略的東西，我自己讀起來是順序的確跟作者說的一樣，但我並不會忽略他的型態。

## 縮排
> 一個原始檔是個階層結構，而非大綱結構。
> 程式設計師相當依賴縮排的設計，依據程式碼的縮排層次，來了解這段程式碼是屬於哪個視野，這讓他們可以快速地略過某些視野。

這段我想不需要多說明，沒有縮排的程式碼是多驚人的事情。這件事在 Python 倒是不可能發生就是了，因為 Python 將縮排作為視野 (scope) 的區別，而不是使用 `{}`。

底下程式碼顯示沒有使用縮排是多麼嚇人的一件事：
```go
type config struct {
	slackToken   string
	slackURL     string
	slackWebhook string
	gitlabToken  string
	gitlabURL    string
	dbFile       string
	token        string
}

var cfg config

func initConfig() error { slackToken := os.Getenv("SLACK_TOKEN"); if slackToken == "" { slackToken = "SLACKTOKEN" }; webhook := os.Getenv("SLACK_WEBHOOK"); if webhook == "" { webhook = "WEBHOOK" }; gitlabToken := os.Getenv("GITLAB_TOKEN"); if gitlabToken == "" { gitlabToken = "GITLABTOKEN" }; gitlabURL := os.Getenv("GITLAB_URL"); if gitlabURL == "" { gitlabURL = "GITLABURL" }; dbFile := os.Getenv("DB_FILE"); if dbFile == "" { dbFile = "DBFILE" }; token := os.Getenv("TOKEN"); cfg = config{ slackToken: slackToken, slackURL: "https://slack.com/api", slackWebhook: webhook, gitlabToken: gitlabToken, gitlabURL: gitlabURL, dbFile: dbFile, token: token }; return nil }


func main() { err := initConfig(); if err != nil { log.Fatal(err) }; fmt.Println(cfg) }
```

上面這段程式碼完全可以 compile 過且正確執行。若你接觸到的程式碼有此類，請直接離職。

## 空視野的範圍
> 當無法避免的時候，我會確保空白區塊也被適當的縮排，並且使用大括號將之括起來。

作者提到，他已經被座落在 `while` 迴圈後面同一行的分號愚弄過非常多次了。所以作者非常建議一定要使用括號將空白區塊括起來。

```java
while (dis.read(buf, 0, readBufferSize) != -1)
    ;
```

改成底下這樣子會比較好：
```java
while (dis.read(buf, 0, readBufferSize) != -1) {
};
```

# 團隊的共同準則
> 如果他身處在一個團隊，那他必須以團隊的編排準則為主。
> 不要在原始碼中摻入雜亂的個人風格程式碼，這樣會增加閱讀原始碼的複雜性。

很多東西都只是 convention，但我相當支持一定要有一套準則，準則可以討論、可以修改，但一定要明訂出來讓大家遵守。
有時候總是會聽到一句跟屁話沒兩樣的話：「沒有規則也是一種規則」。
我沒有錢也是很有錢的一種嗎？

# Bob 大叔的編排準則
此處作者給了一份程式碼，是他自己的習慣，其程式碼可以在[這裡](https://github.com/ludwiggj/CleanCode/blob/master/src/clean/code/chapter05/CodeAnalyzer.java)找到。