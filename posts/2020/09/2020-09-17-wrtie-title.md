原文: [Keeping your Laravel applications DRY with single action classes](https://medium.com/@remi_collin/keeping-your-laravel-applications-dry-with-single-action-classes-6a950ec54d1d)
作者: [Rémi Collin](https://medium.com/@remi_collin?source=post_page-----6a950ec54d1d--------------------------------)

在談到應用程式的架構時,經常會被問到像是，"我要把這段程式碼放在哪裡?"這類的問題。
Laravel是一個相當靈活的框架，想要回答這個這個問題並不是這麼容易。
我的商業邏輯是要寫在 model 中? controller 中? 還是寫在其他地方?

當你的應用只有單一進入點時，當然是可以寫在 controller 中。但是現實中，不同邏輯間有共用的功能是非常常見的。

例如，大多數應用程式會有一個帳號註冊的表單，它會使用到一個 controller，並根據成功或失敗的情況去回傳相應的 view。
如果還有給 mobile 使用的部分，這邊可能會有專用的註冊 API 來回傳 JSON 格式的資料。或許也會建立一組於 artisan 的指令來建立帳號，這在早期開發階段也是蠻常見的。

這種重複性的程式碼，可能看起來是並無大礙的。但是，當邏輯繼續延伸下去，例如加上了要寄送 email 通知給新註冊的會員時，你必須要記得在兩個不同的 controller 上面都要加上這段程式碼。
因此如果要繼續遵守 DRY 原則的話，我們就必須要包裝這一段的邏輯‧

對於這種情境，在不同的論壇上可以找到很多類似的作法是，"使用 service 然後在 controller 去呼叫對應的 method"。
好吧，但是我要如何建構 service? 我需要建立一個 UserService 實作有所有關於 user 的邏輯? 我要在所有用到的地方都注入 service? 還是我有其他的地方可以丟? 

## 避免讓 service 成為 god class
首先，我們會想要建立一個獨立的 class，並且將關於特定 model 的程式碼都集合在一起，就像是這樣:

這看起來蠻讓人滿意的，我們可以在不同的 controller 去執行建立或是刪除使用者並且得到結果。但是，這個做法會有什麼問題呢? 答案是，"我們很少只會使用到單一及隔離的 model 去處理邏輯"。

比如說，當一個使用者建立了一個帳號，一個新的 blog 也會一起建立起來。如果依照我們目前的模式，我們必須建立一個 BlogService ，還要將其注入到 UserService 成為其依賴。

結果應該不難猜到，當我們的應用程式越來越龐大時，最終會存在十幾個 service ，其中某幾個 service 可能會依賴其他五或六個 service。最終就會變成我們極力要避免的 Spaghetti code。

## 介紹 single action 物件
那麼，如果我們把 service 拆成好幾個 class，而不是像現在的單一個 class 呢? 這是我在最近幾個 project 中使用的方法，效果十分不錯。

首先，我們先丟到 service 這個太通用且模糊的詞，讓我們引用 action，並且定義一下 action class 可以做的事情:

- 每個 action 的 class name 應該要有一個明確且一眼就可以了解邏輯的命名，例如: CreateOrder, ConfirmCheckout, DeleteProduct 與 AddProductToCart ......等等。
- Action 應該只會有一個 public method，且名稱要統一，例如: handle() 或 execute()。這會讓往後實作某些 pattern 會容易許多(例如 adapter pattern)。
- Action 必須不直接依賴 request 與 response，也不能直接處理任何與 request 與 response 相關的資料，這些都是 controller 要處理的邏輯。
- 所有的 Action 可以被其他 action 所依賴
- 所有的業務必須通過拋出 Exception 來處理錯誤情況，讓 Exception 的 Handler 來統一處理錯誤訊息。

## 總結
剛開始實做這個模式時，會在開始的階段就創建了許多的 action class。為了減少文章的篇幅與容易理解，這邊的示範情境使用了使用者註冊來當範例。
一旦業務邏輯開始複雜許多時，這個模式的價值就開始體現出來了，因為你會知道邏輯都在同一個地方，且切割明確。

## 使用 single action 的好處

- 使用小型物件去管理邏輯可以保持 DRY 的原則、增強可用性與保持 SOILD 的原則。
- 容易針對不同情境進行隔離測試
- 有意義的命名可以使維護的人更好理解邏輯
- 整個專案的一致性，避免程式碼零散在不同的 controller 或是 model 之類。

當然, 這是我根據過去幾年使用Laravel的經驗和我經手處理過的專案所得出的方法. 
這對我來說真的很有效, 我現在甚至在中小型的專案開發中都會使用它.

我很想與你交流想法, 特別是如果你有著不同的方法的話。
