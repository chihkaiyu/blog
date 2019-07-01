---
title: "Clean Code Digest - Chapter 6 物件及資料結構"
slug: "clean-code-chapter-6"
date: 2018-10-19T01:15:38+08:00
draft: false
image: "/content/images/2018/10/DSC04477.jpg"
tags: ["digest"]
---

# Chapter 6 物件及資料結構
# TL; DR
我認為此章節所想表達的有下列事項：
- 抽象化：資料需要抽象化隱藏其實作過程。在封裝起來後要依然能操作資料的本質。
- 互補：以資料結構為本位的程式容易增加新函式，但不易增加新的資料結構；以物件為本位的程式容易增加新類別，但不易增加新函式。

# 資料抽象化
> 類別提供了一個抽象介面，讓使用者在不需要知道實現過程的狀態下，還能夠操縱資料的本質
> 嚴謹一點的最好作法是，想辦法找到最能詮釋「資料抽象概念」的方式。
> 最糟糕的作法，則是天真快樂地加上讀取函式及設定函式而已。

作者以座標點及交通工具剩餘油量作為例子：
座標點：
```go
// 具體的座標點
type Point struct {
    X float64
    Y float64
}

// 抽象的座標點
type Point interface {
    getX() float64
    getY() float64
    setCartesian(float64, float64)
    getR() float64
    getTheta() float64
    setPolar(float64, float64)
}
```

在抽象座標點裡，你無法分辨這個實現過程是平面座標還是極座標。這樣的界面除了清楚的表示這是一種資料結構外，還限制了存取的手段。
而在具體座標點裡，可以很清楚的知道這是直角座標，而且必須單獨操作各軸座標。

剩餘油量：
```go
// 具體化的剩餘油量
type FuelTankCapacityInGallons interface {
    getGallonsOfGasoline() float64
}

// 抽象化的剩餘油量
type Vehicle interface {
    getPercentFuelRemaining() float64
}
```

上述例子裡，具體化的剩餘油量很明顯看得出是直接存取某個變數，而抽象化的會回傳剩餘比例，我們並不知道實際內部資料型態如何。

作者想表達的是，即使是變數是 private member，但透過變數的存取函式，程式的實作過程還是會曝露出來。最好的方法是想一個能解釋資料的抽象概念，以該概念來代替存取變數的函式。

# 資料/物件的反對稱性
> 物件將它們的資料在抽象層後方隱藏起來，然後將操縱這些資料的函式曝露在外。
> 資料結構則將資料曝露在外，而且也沒有提供有意義的函式。
> 結構化的程式碼 (使用資料結構的程式碼) 容易添加新的函式，而不需要變動已有的資料結構。
> 物件導向的程式碼，容易添加新的類別，而不用變動已有的函式。

作者以底下程式碼作為例子：
```java
// 程序式圖形
public class Square {
    public Point topLeft;
    public double side;
}

public class Rectangle {
    public Point topLeft;
    public double height;
    public double width;
}

public class Circle {
    public Point center;
    public double radius;
}

public class Geometry {
    public final double PI = 3.141592653589793;

    public double area(Object shape) throws NoSuchShapeException
    {
        if (shape instanceof Square) {
            Square s = (Square)shape;
            return s.side * s.side;
        }
        else if (shape instanceof Rectangle) {
            Rectangle r = (Rectangle)shape;
            return r.height * r.width;
        }
        else if (shape instanceof Circle) {
            Circle c = (Circle)shape;
            return PI * c.radius * c.radius;
        }
        throw new NoSuchShapeException();
    }
}
```

```java
// 多型的圖形
public class Square implements Shape {
    private Point topLeft;
    private double side;

    public double ares() {
        return side*side;
    }
}

public class Rectangle implements Shape {
    private Point topLeft;
    private double height;
    private double width;

    public double area() {
        return height * width;
    }
}

public class Circle implements Shape {
    private Point center;
    private double radius;

    public final double PI = 3.141592653589793;
    public double area() {
        return PI * radius * radius;
    }
}
```

上述程式碼裡，程序式的寫法，在新增一個新的圖形類別會相當麻煩，你必須改變 `Geometry` 裡所有的函式；反過來說，以多型的方式寫，在新增一個新的函式時會相當麻煩。反過來說也是成立的。
而最重要的是，一個成熟的程式設計師應該知道什麼時候使用物件，什麼時候單純使用資料結構。要讓所有東西都變成物件是不可能的 (原文使用神話這個詞)。

# 德摩特爾法則 (The Law of Demeter)
> 模組不該知道「關於它所操縱物件的內部運作」
> 方法不該呼叫「由任何函式所回傳之物件」的方法。只和朋友說話，不跟陌生人聊天。

The Law of Demeter
一個類別 `C` 內的方法 `f`，應該只能呼叫以下事項的方法：
- `C`
- 任何由 `f` 所產生的物件
- 任何當作參數傳遞給 `f` 的物件
- `C` 類別裡實體變數所持有的物件

## 火車事故
> 一連串相連的程式呼叫，通常被認為是一種很懶散的程式風格。

```go
outputDir := ctxt.getOptions().getScratchDir().getAbsolutePath()
```

上述程式碼被稱為火車事故，比較好的作法是將之分割為以下形式：
```go
opts := ctxt.getOptions()
scratchDir := opts.getScratchDir()
outputDir := scratchDir.getAbsolutePath()
```

作者並未說明為什麼這樣子寫不好，只說了句：通常被認為是一種懶散的風格。但這類的風格的確存在，稱之為 [Fluent Interface](https://en.wikipedia.org/wiki/Fluent_interface)。我第一次看到這種風格的程式碼是在 [pyspark 程式碼](https://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.DataFrameWriter.sortBy)裡看到的。第一眼倒是沒有太多負面的感想，甚至還覺得有點容易閱讀。
然而上述範例程式碼是否違反 the law of Demeter 呢？作者給的答案是，取決於 `ctxt`、`Options`、`ScratchDir` 是物件還是資料結構。如果是物件，那它們的內部結構應該隱藏起來，如果能獲知它們的內部資訊，則明顯違反了 the law of Demeter。
如果上述程式碼變為底下形式：
```go
outputDir := ctxt.options.scratchDir.absolutePath
```
如此一來它就只是單純的資料結構，我們便不會懷疑它是否違反 the law of Demeter。但據說此法則仍在爭辯中，並沒有一個結果。

## 混合體
> 它們擁有函式來作一些重要的事，它們也有公共變數或公共存取器、修改器。
> 這類的混合體是兩種世界裡最糟的情況。

作者提到在一個結構化程式裡面操作資料結構稱之為 Feature Envy，意思是某個類別裡的函式對其他類別裡的變數更感興趣，可以想成是類別間的依賴關係切不乾淨。
此類混合體是非常糟糕的設計，作者根本不確定 (或是直接忽略) 它們是否需要函式的或型態的保護。

## 隱藏結構
> 如果 `ctxt` 是一個物件，那我們應該要告訴它去做 `某某事情`；我們不應該還被問到它的內部結構是什麼。

如果 `ctxt`、`Options`、`ScratchDir` 是擁有真實行為的物件，又該怎麼辦？由於物件應該將其內部結構隱藏起來，因此我們不能探索物件內部，那又該怎麼拿到 `scratchDir` 的 `absolutePath` 呢？
作者提出兩個可能的解法：
```go
ctxt.GetAbsolutePathOfScratchDirectoryOption()
```

```go
ctxt.getScratchDirectoryOption().getAbsolutePath()
```

第一個方法會導致 `ctxt` 有許多物件，第二個方法則假設 `getScratchDirectoryOption()` 會回傳資料結構而不是物件。這兩個方法都不是很好。
作者在看完程式碼的上下文後，發現拿 `absolutePath` 只是為了在此目錄下產生一個給定名稱的檔案，於是提出了第三種方法：
```go
bos := ctxt.createScratchFileStream(classFileName)
```

如此 `ctxt` 既能隱藏物件內部結構，又能遵守 the law of Demeter。

# 資料傳輸物件 (Data Transfer Objects, DTO)
> 最佳的資料結構形式，是一個類別裡只有公用變數，沒有任何函式。

此資料結構有時被稱為 DTO，當我們要和資料庫溝通或解析由 socket 傳來的訊息時，是非常有用的結構。在將「資料庫的原始資料」轉換成「應用程式內的物件」時，通常在轉換過程的第一階段會用到。

## 活動記錄
> 活動紀錄是由資料庫表格或資料來源直接轉換而來。
> 開發者會將這類資料結構視為物件，然後加入處理商業法則的方法，最後會產生資料結構與物件的混合物。

作者提到這類的資料結構是一種特別的 DTO，它們擁有公用變數，但通常也有 `save` 與 `find` 來瀏覽的方法。但通常這類的資料結構會被誤用，最終導致產生出混合體。

# 總結
> 物件會曝露它們的行為並隱藏其內部資料，這讓我們在不改變現有行為的情況下，能輕易添加新類型的物件，但添加新行為卻變得困難。
> 資料結構會曝露其資料但不會有顯著行為，這讓我們在現有的資料結構上能輕易的添加新行為，但在現有函式要添加新函式卻變得困難。

優秀的程式設計師能理解其使用情境，在不帶偏頗的情況下選擇最適合的方法來完成。