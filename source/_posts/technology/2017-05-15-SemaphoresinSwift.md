---
layout: post
date: 2017-05-15 10:50:46
title: Swift 中的信號量
category: 技术
tags: 
- 资料
- swift
keywords: swift
description: 转自简书Arco翻译的文章
---
# (譯) Swift 中的信號量 🚦

作者 [Arco_vv](http://www.jianshu.com/u/07d93406ec39) 


> [原文在此](https://medium.com/swiftly-swift/a-quick-look-at-semaphores-6b7b85233ddb)，并經由[作者](https://medium.com/@zntfdr)的同意，擁有翻譯的權利。

首先，如果你對 GCD 或者是隊列調度 (Dispatch Queue) 不夠熟悉，請先讀下這篇來自 [AppCoda](https://medium.com/@appcodamobile) 的[好文章](http://www.appcoda.com/grand-central-dispatch/)。

回到正軌，是時候開始講一些關於信號量 (Semaphores) 的事情了！

![](https://cdn-images-1.medium.com/max/1600/1*8ZCGzvA6DjfR9JoamqauoQ.jpeg)

Traffic Light by spaztacular



## 介紹

想象出一個場景：有一群作家只能共用一支筆，因此很明顯在某個時間範圍內，只有一名作家可以使用這支筆。

現在，把這群作家想象成我們程式碼中的線程，而那支筆就是我們共享的資源(它可以是任何東西，一個文件，一個變量，或者有做某事的權利)。

因此我們的問題就在於，如何確保我們的資源是互斥的 [(mutually exclusive)](https://en.wikipedia.org/wiki/Mutual_exclusion) 呢？

![](https://cdn-images-1.medium.com/max/1600/1*nfAYVSYFMB874-z4sfJ_YQ.jpeg)

Streetless traffic light by Sylvain Bourdos



## 實現我們的訪問資源控制器 (Resource Control Access)

有些人或許會開始想：我只要有一個 resourceIsAvailable 的布林值，依據 true/false 來控制不就行了？

```
if resourceIsAvailable {
    resourceIsAvailable = false
    useResource()
    resourceIsAvailable = true
} else {
    // resource is not available, wait or do something else
}
```

這樣實現的問題就在於：當處於並發時，我們并不能保證知曉究竟是哪一條線程將執行下一步，無論它的優先權是什麼。

## 例子

想象一下，我們有兩個線程，線程 A 和線程 B，來實現以上的代碼，來共享一個互斥的資源：

* 線程 A 讀取 if 條件並且發現這個資源是可用的。
* 但在下一行 (resourceIsAvailable = true) 被執行之前，處理器轉向了線程 B，它也讀取了 if 條件。
* 這時候我們就有兩個線程都相信這個資源是可被使用的，並且他們都去執行了 *useResource* 閉包。

離開 GCD 來寫線程安全的代碼并不是一個簡單的任務。

![](https://cdn-images-1.medium.com/max/1600/1*p54pBislRafckGffcDqRdA.png)

At the lights by petemc



## 思考信號量是怎麼工作的

通過三個步驟：

1. 當我們想使用一個共享的資源，我們對其信號量發出一個請求；
2. 一旦信號量給了我們綠燈，我們就能假定這個資源是我們的並且我們能夠使用它；
3. 一旦這個資源不再被需要，我們再給這個信號量發送一個示意 (signal)，允許它將這個資源分配給其他的線程。

當有且只有一個資源，並其只能被一個線程在任意時間使用，你就可以將其請求/示意 (request/signal)，思考為加鎖/解鎖 (lock/unlock)。

![](https://cdn-images-1.medium.com/max/1600/1*-_owdkyNPRUQS5a5yjdEkA.jpeg)

Robots/Traffic Light by mallix



## 這個場景的背後發生了什麼

### 結構：

一個信號量由兩部分組成：

* 一個計數器 (counter) 來讓信號量知曉有多少線程可以使用這些資源；
* 一個 *先入先出隊列 (FIFO)* 用於追蹤等待資源的線程；

### 資源請求：wait()

當信號量接收到一個請求，它會先檢查它的計數器是否超過0：

* 如果是，信號量會將計數器減量，并給這個線程綠燈；
* 如果不是，則將這個線程放在隊列的最後；

### 資源釋放：signal()

當信號量接收到一個示意，他會先檢查它的先入先出隊列是否有其他線程：

* 如果有，信號量會拉出第一個線程，并給這個線程綠燈；
* 如果不是，則給它的計數器增量；

### 警告：忙碌等待

當一個線程發送一個 *wait()* 的資源請求給信號量，該線程會被 *凍結 (freeze)* 直到信號量告訴它綠燈。

⚠️如果你在主線程上干這事，整個 App 都會凍結住！⚠️

![](https://cdn-images-1.medium.com/max/1600/1*3GANzX3n1uEiuhXE49fcrg.jpeg)

STOP Traffic Lights & Sunset by eyecmore



## 在 Swift 中通過 GCD 使用信號量

寫一些代碼吧！

### 宣告

聲明一個信號量是很簡單的事情：

```
let semaphore = DispatchSemaphore(value: 1)
```

參數 *value* 代表有多少線程可以經信號量創建後來訪問該資源。

### 資源請求

**請求** 信號量的資源，我們只要調用如下：

```
semaphore.wait()
```

需要注意的是信號量并不會在物理上給我們任何東西。資源需要在線程自己的域 (scope) 中準備完畢，我們只在請求和釋放的調用中使用這個資源。

一旦信號量給予我們他的許可，線程就會恢復他的正常執行，并考慮如何使用他的資源。

### 資源釋放

**釋放** 這個資源：

```
semaphore.signal()
```

只要我們向信號量示意後，我們就不再被允許使用資源，直到我們再次請求。

## 信號量 Playgrounds

我們跟隨 [AppCoda](https://medium.com/@appcodamobile) [文章](http://www.appcoda.com/grand-central-dispatch/) 的一些例子，來看看信號量的表現吧！

> 提示：這是 Xcode 的 Playground 例子。Swift Playgrounds 還尚未支持記錄這件事情。等待 WWDC17 吧！

在這些 playgrounds 中，我們擁有兩個線程，其中一個有較高的優先權。他們都要打印10次一個 emoji 以及遞增的數字。

### Semaphore-less Playground

```
import Foundation
import PlaygroundSupport

let higherPriority = DispatchQueue.global(qos: .userInitiated)
let lowerPriority = DispatchQueue.global(qos: .utility)

func asyncPrint(queue: DispatchQueue, symbol: String) {
  queue.async {
    for i in 0...10 {
      print(symbol, i)
    }
  }
}

asyncPrint(queue: higherPriority, symbol: "🔴")
asyncPrint(queue: lowerPriority, symbol: "🔵")

PlaygroundPage.current.needsIndefiniteExecution = true
```

正如你所想象的那樣，擁有更高優先權的線程也同樣會在大部分時間先完成事情：

![](https://cdn-images-1.medium.com/max/1600/1*OjtJO8-44tStXpRS8y1N-A.png)

Semaphore-less Playground



### Semaphore Playground

在這個例子中，我們仍然會使用原先的代碼，但我們會給一個線程在一個時間內印出 emoji + 數字 序列的權利。

因此我們需要決定一個信號量以及更新我們的 *asyncPrint* 函數：

```
import Foundation
import PlaygroundSupport

let higherPriority = DispatchQueue.global(qos: .userInitiated)
let lowerPriority = DispatchQueue.global(qos: .utility)

let semaphore = DispatchSemaphore(value: 1)

func asyncPrint(queue: DispatchQueue, symbol: String) {
  queue.async {
    print("\(symbol) waiting")
    semaphore.wait()  // requesting the resource

    for i in 0...10 {
      print(symbol, i)
    }

    print("\(symbol) signal")
    semaphore.signal() // releasing the resource
  }
}

asyncPrint(queue: higherPriority, symbol: "🔴")
asyncPrint(queue: lowerPriority, symbol: "🔵")

PlaygroundPage.current.needsIndefiniteExecution = true
```

我還增加了一對 *print* 指令表明在我們的任務中每一個線程的實際狀態。

![](https://cdn-images-1.medium.com/max/1600/1*g7SMrR7svWNetOqjSGIEYA.png)

Semaphore Playground



正如你所見，當一個線程開始打印它的序列時，其他線程都必須等待，直到最開始的那個完畢。然後信號量會接收到來自第一個線程的 *示意* ，有且在這個時候，第二個線程才能開始打印它自己的序列。

第二個線程在哪裡打印 *wait()* 的請求并不是重點，重點是它都必須等待直到其他的線程已經完成。

### 優先權倒置 (Priority Inversion)

現在我們理解了每件事情的發生，請看一下以下的記錄：

![](https://cdn-images-1.medium.com/max/1600/1*eCFBl9XpF6JYX1b8xwD26w.png)

Priority Inversion



在上面的代碼的情況下，處理器決定先執行了低優先權的線程。

當這種情況發生時，高優先權線程必須等待低優先權線程的完成。實際上確實會發生這種情況。這個問題在於低優先權線程後面有高優先權線程在等待，這種現象叫做 [優先權倒置 (Priority Inversion)](https://en.wikipedia.org/wiki/Priority_inversion)

> 譯者注：優先權倒置，又稱優先權反轉、優先權逆轉、優先權翻轉，是一種不希望發生的任務調度狀態。在該種狀態下，一個高優先級任務間接被一個低優先級任務所搶先(preemtped)，使得兩個任務的相對優先級被倒置。
> 這往往出現在一個高優先級任務等待訪問一個被低優先級任務正在使用的臨界資源，從而阻塞了高優先級任務；同時，該低優先級任務被一個次高優先級的任務所搶先，從而無法及時地釋放該臨界資源。這種情況下，該次高優先級任務獲得執行權。
> 在多數個案，發生優先權倒置並不導致直接傷害──高優先權任務的延遲執行不被察覺，最終，低優先權任務釋放共享資源。雖然，亦存在很多情況優先權倒置會導致嚴重問題。
> —— 摘自維基百科

信號量并不屬於這種情況，因為實際上任何人都可以呼叫 *call()* 函數 (而並不是因為這個線程正在使用這個資源的原因)。

### 線程飢餓 (Thread Starvation)

讓我們想象出一個更糟糕的情況：在我們的高和低優先權的線程中，有1000個以上的中優先權的線程。

如果我們有一個像之前所提到的 *優先權倒置* 的情況，高優先權線程就必須等待低優先權線程，但在更多時候，處理器會執行中優先權的線程，因為和低優先權線程相比，他們有跟高的權利。

在這個劇本中，我們的高優先權線程正在餓死 CPU 的時間（這也就是線程飢餓的概念）。

### 解決方案

我認為，信號量在使用的時候，最好保證每個訪問的線程都處於同樣的優先權當中。如果你認為這沒辦法解決你的問題，我推薦你看一下其他的解決之道，例如 [Regions](https://en.wikipedia.org/wiki/Critical_section) 和 [Monitors](https://en.wikipedia.org/wiki/Monitor_(synchronization))。

## Playground 死鎖 (Deadlock Playground)

這次我們使用兩個線程來使用兩個互斥的資源 "A" 和 "B"。

如果兩種資源可以分開來使用，那麼為每一個資源定義一個信號量則是有意義的。如果不行的話，一個信號量就足夠管理兩個資源了。

在這裡，我想用先前一種情況 (2資源，2信號量) 作為例子：高優先權的線程將先使用資源 "A"，然後是資源 "B"；而我們的低優先權線程將會先使用資源 "B"，然後是資源 "A"。

代碼如下：

```
import Foundation
import PlaygroundSupport

let higherPriority = DispatchQueue.global(qos: .userInitiated)
let lowerPriority = DispatchQueue.global(qos: .utility)

let semaphoreA = DispatchSemaphore(value: 1)
let semaphoreB = DispatchSemaphore(value: 1)

func asyncPrint(queue: DispatchQueue, symbol: String, firstResource: String, firstSemaphore: DispatchSemaphore, secondResource: String, secondSemaphore: DispatchSemaphore) {
  func requestResource(_ resource: String, with semaphore: DispatchSemaphore) {
    print("\(symbol) waiting resource \(resource)")
    semaphore.wait()  // requesting the resource
  }

  queue.async {
    requestResource(firstResource, with: firstSemaphore)
    for i in 0...10 {
      if i == 5 {
        requestResource(secondResource, with: secondSemaphore)
      }
      print(symbol, i)
    }

    print("\(symbol) releasing resources")
    firstSemaphore.signal() // releasing first resource
    secondSemaphore.signal() // releasing second resource
  }
}

asyncPrint(queue: higherPriority, symbol: "🔴", firstResource: "A", firstSemaphore: semaphoreA, secondResource: "B", secondSemaphore: semaphoreB)
asyncPrint(queue: lowerPriority, symbol: "🔵", firstResource: "B", firstSemaphore: semaphoreB, secondResource: "A", secondSemaphore: semaphoreA)

PlaygroundPage.current.needsIndefiniteExecution = true
```

如果我們足夠幸運，會發生以下的情況：

![](https://cdn-images-1.medium.com/max/1600/1*_ASgiqbV_o9caE7M7hNBpQ.png)

Happened one



簡單來說，高優先權線程將會被第一個資源所服務，然後是第二個資源。只有在那完成之後，處理器才會轉移去處理低優先權的線程。

但是，如果我們運氣比較差，同樣也會發生這種情況：

![](https://cdn-images-1.medium.com/max/1600/1*cVvGM-1NRH7kouSRu2mSRQ.png)

Happened two



兩個線程都沒有完成他們的任務！讓我們回顧一下當前狀態：

* 高優先權線程正在等待那個被低優先權線程持有的資源 "B"。
* 低優先權線程正在等待那個被高優先權線程持有的資源 "A"。

兩個線程都在等待那個不可能到來的資源：歡迎來到[線程死鎖](https://en.wikipedia.org/wiki/Deadlock)的世界！

### 解決方案

想要避免[死鎖](https://en.wikipedia.org/wiki/Deadlock)并不是一件容易的事情。最好的解決方案就是通過你的代碼——編寫出不可能發生這種情況的代碼——[阻止這種情況的發生](https://en.wikipedia.org/wiki/Deadlock_prevention_algorithms)。

舉例來說，在其他 OS 系統中，一個死鎖線程可能會被殺死 (為了釋放他所佔有的所有資源)，并因此而希望其他線程可以繼續執行他們的任務。

或者你可以試試看使用 [Ostrich_Algorithm](https://en.wikipedia.org/wiki/Ostrich_algorithm) 😆。

![](https://cdn-images-1.medium.com/max/1600/1*Nmcb2GTIk-PO0TNPNPD8Mw.jpeg)

Let Me Keep My Memories by Thomas Hawk



## 總結

信號量是一個棒的一個概念，在許多應用中可以非常方便的使用。只是你要小心：過馬路的時候要看看左右兩邊。

* * *

### 作者

[Federico](https://twitter.com/zntfdr) is a Bangkok-based Software Engineer with a strong passion for Swift, Minimalism, Design, and iOS Development.

### 譯者

[Arco](http://www.jianshu.com/u/07d93406ec39)，iOS 開發，目前正在台灣工作。

* * *

經由譯者許可才可進行轉發。



