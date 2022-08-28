- 原文: [Keeping your Laravel applications DRY with single action classes](https://medium.com/@remi_collin/keeping-your-laravel-applications-dry-with-single-action-classes-6a950ec54d1d)
- 作者: [Rémi Collin](https://medium.com/@remi_collin?source=post_page-----6a950ec54d1d--------------------------------)

# 讓 Action 幫你遵守 DRY 原則

當談到應用程式的架構時，經常會被問到的像是: "我要把這段程式碼放在哪裡?" 之類的問題。

還有像是: 我的商業邏輯是要寫在 model 中、controller 中、還是寫在其他地方? 就算是靈活的框架的 Laravel，想要明確給予這些問題的答案並不是這麼容易。

當你的應用只有單一進入點時，當然是可以寫在 controller 中。但是現實中，不同邏輯間有共用的功能是非常常見的。

例如，大多數應用程式會有一個帳號註冊的表單，它會使用到一個 controller，並根據成功或失敗的情況去回傳相應的 view。
如果還有給 mobile 使用的部分，這邊可能會有專用的註冊 `API` 來回傳 `JSON` 格式的資料。或許也會建立一組於 `artisan` 的指令來建立帳號，這在早期開發階段也是蠻常見的。

``` php
class UserController
{
    public function create(CreateUserRequest $request)
    {
        $user = User::create($request->all());

        return view('user.created', ['user' => $user]);
    }
}

class UserApiController
{
    public function create(CreateUserRequest $request)
    {
        $user = User::create($request->all());

        return response()->json($user);
    }
}
```

這種重複性的程式碼，可能看起來是並無大礙的。但是，當邏輯繼續延伸下去，例如加上了要寄送 email 通知給新註冊的會員時，你必須要記得在兩個不同的 controller 上面都要加上這段程式碼。
因此如果要繼續遵守 `DRY` 原則的話，我們就必須要包裝這一段的邏輯‧

對於這種情境，在不同的論壇上可以找到很多類似的作法是，"使用 service 然後在 controller 去呼叫對應的 method"。
好吧，但是我要如何建構 service? 我需要建立一個 `UserService` 實作有所有關於 user 的邏輯? 我要在所有用到的地方都注入 service? 還是我有其他的地方可以丟? 

## 避免讓 service 成為 god class
首先，我們會想要建立一個獨立的 class，並且將關於特定 model 的程式碼都集合在一起，就像是這樣:

``` php
class UserService
{
    public function create(array $data) : User
    {
        $user = new User;
        $user->email = $data['email'];
        $user->password = $data['password'];
        $user->save();

        return $user;
    }

    public function delete($userId) : bool
    {
        $user = User::findOrFail($userId);        
        $user->delete();

        return true;
    }
}

```

這看起來蠻讓人滿意的，我們可以在不同的 controller 去執行建立或是刪除使用者並且得到結果。但是，這個做法會有什麼問題呢? 答案是，"我們很少只會使用到單一及隔離的 model 去處理邏輯"。

比如說，當一個使用者建立了一個帳號，一個新的 blog 也會一起建立起來。如果依照我們目前的模式，我們必須建立一個 `BlogService` ，還要將其注入到 `UserService` 成為其依賴。

``` php
class UserService
{
    protected $blogService;

    public function __construct(BlogService $blogService)
    {
        $this->blogService = $blogService;
    }

    public function create(array $data) : User
    {
        $user = new User;
        $user->email = $data['email'];
        $user->password = $data['password'];
        $user->save();

        $blog = $this->blogService->create();
        $user->blogs()->attach($blog);           

        return $user;
    }
    
    ...
}
```

結果應該不難猜到，當我們的應用程式越來越龐大時，最終會存在十幾個 service ，其中某幾個 service 可能會依賴其他五或六個 service。最終就會變成我們極力要避免的 Spaghetti code。

## 介紹 single action 物件
那麼，如果我們把 service 拆成好幾個 class，而不是像現在的單一個 class 呢? 這是我在最近幾個 project 中使用的方法，效果十分不錯。

首先，我們先丟掉 **service** 這個太通用且模糊的詞，讓我們引用 **action**，並且定義一下 action class 可以做的事情:

- 每個 action 的 class name 應該要有一個明確且一眼就可以了解邏輯的命名，例如: `CreateOrder`, `ConfirmCheckout`, `DeleteProduct` 與 `AddProductToCart` ......等等。
- Action 應該只會有一個 public method，且名稱要統一，例如: `handle()` 或 `execute()`。這會讓往後實作某些 pattern 會容易許多(例如 adapter pattern)。
- Action 必須不直接依賴 **request** 與 **response**，也不能直接處理任何與 **request** 與 **response** 相關的資料，這些都是 controller 要處理的邏輯。
- 所有的 Action 可以被其他 action 所依賴
- 所有的業務必須通過拋出 `Exception` 來處理錯誤情況，讓 `Exception` 的 `ExceptionHandler` 來統一處理錯誤訊息。

## 建立 CreateUser 的 action class
現在，讓我們用新的 `CreateUser` action 來重構前面說到的範例。

``` php
class CreateUser
{
    protected $createBlog;

    public function __construct(CreateBlog $createBlog)
    {
        $this->createBlog = $createBlog;
    }

    public function execute(array $data) : User
    {
        $email = $data['email'];

        if (User::whereEmail($email)->first()) {
            throw new EmailNotUniqueException("$email should be unique.");
        }

        $user = new User;
        $user->email = $data['email'];
        $user->password = $data['password'];
        $user->save();

        $blog = $this->createBlog->execute();
        $user->blogs()->attach($blog); 

        return $user;
    }
}
```

你可能會想說，為什麼要在已經取得 Email 的情況時拋出異常？ 這不是應該由 request 的階段就進行驗證嗎？
當然該讓驗證器去處理類似的資料檢查。然而，在 action 中處理業務規則是個好主意，這使得業務邏輯更容易理解，除錯也更加容易。

以下是使用 action 重構之後的 controller:

``` php
class UserController
{
    public function create(CreateUserRequest $request, CreateUser $action)
    {
        $user = $action->execute($request->all());

        return view('user.created', ['user' => $user]);
    }
}

class UserApiController
{
    public function create(CreateUserRequest $request, CreateUser $action)
    {
        $user = $action->execute($request->all());

        return response()->json($user);
    }
}
```

現在，無論我們對使用者註冊流程進行了任何的修改，都會在 `API` 和 `Web` 的版本同時實現，保持程式碼乾淨整潔。

## 注入 action

假設我們需要一個 action 來匯入 1000 個使用者帳號到我們的應用中。我們可以為這個匯入的邏輯依賴於 `CreateUser` 這個 action‧

``` php
class ImportUser
{
    protected $createUser;

    public function __construct(CreateUser $createUser)
    {
        $this->createUser = $createUser;
    }

    public function execute(array $rows) : Collection
    {
        return collect($rows)->map( function(array $row) {
            try{
                return $this->createUser->execute($row);
            } catch (EmailNotUniqueException $e) {
                // Deal with duplicate users
            }
        });
    }
}
```

很乾淨，是吧？ 我們可以通過將 `CreateUser` 注入到 `Collection::map()` 中來重複使用，然後返回一個包含了所有剛建立的使用者帳號的 Collectioin。 
我們可以稍微改變它，當郵件重複時，我們可以返回一個 `Null Object` ，或者把這些訊息丟到 Log 裡，隨機應變。

## 使用修飾模式

現在假設我們想將每個新註冊的使用者記錄到一個檔案中。 我們可以把這段程式碼放到 action 本身之中，但我們也可以使用`裝飾模式`。

``` php
class LogCreateUser extends CreateUser
{
    public function execute(array $data) : User
    {
        Log::info("A new user has registered : " . $data['email']);

        return parent::execute($data);
    }
}
```

然後，透過使用 Laravel 的 IoC 容器，我們可以將 `LogCreateUser` class 綁定到 `CreateUser` 上，這樣在每次使用 `CreateUser` 時都可以將 `LogCreateUser` 注入。

``` php
// AppServiceProvider.php

public function register()
{
    $this->app->bind(CreateUser::class, LogCreateUser::class);
}   
```

這樣就可以透過環境變數，來切換是否需要啟用 Log 機制。

``` php
// AppServiceProvider.php

public function register()
{
    if (config('user.log_registration')) {
        $this->app->bind(CreateUser::class, LogCreateUser::class);
    }
}   
```


## 總結
剛開始實做這個模式時，會在開始的階段就創建了許多的 action class。為了減少文章的篇幅與容易理解，這邊的示範情境使用了使用者註冊來當範例。
一旦業務邏輯開始複雜許多時，這個模式的價值就開始體現出來了，因為你會知道邏輯都在同一個地方，且切割明確。

## 使用 single action 的好處

- 使用小型物件去管理邏輯可以保持 `DRY` 的原則、增強可用性與保持 **SOILD** 的原則。
- 容易針對不同情境進行隔離測試
- 有意義的命名可以使維護的人更好理解邏輯
- 整個專案的一致性，避免程式碼零散在不同的 controller 或是 model 之類。

當然, 這是我根據過去幾年使用Laravel的經驗和我經手處理過的專案所得出的方法. 
這對我來說真的很有效, 我現在甚至在中小型的專案開發中都會使用它.

我很想與你交流想法, 特別是如果你有著不同的方法的話。
