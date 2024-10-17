- 原文: [Keeping your Laravel applications DRY with single action classes](https://medium.com/@remi_collin/keeping-your-laravel-applications-dry-with-single-action-classes-6a950ec54d1d)
- 作者: [Rémi Collin](https://medium.com/@remi_collin?source=post_page-----6a950ec54d1d--------------------------------)

# 讓 Action 幫你遵守 DRY 原則

講到應用程式架構，一個老問題就是「這段程式碼放哪」。 Laravel 很彈性，這問題不好回答。邏輯要寫在 Model、Controller，還是其他地方？

如果你的應用程式只有一個入口，把邏輯寫在 Controller 裡沒問題。但現在，很多端點共用功能是很常見的。

例如，大多數應用程式都有使用者註冊表單，它會呼叫 Controller，然後根據成功或失敗回傳對應的畫面。如果有行動 app，可能還會有專門註冊使用者的 `API` 端點回傳 `JSON`。開發初期也很常用 artisan 命令來建立使用者。

```php
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

程式碼重複看似無害，但如果邏輯變複雜，例如要寄 email 通知新使用者，你就得記得兩個 Controller 都要寄。所以，要保持 DRY（Don't Repeat Yourself，不要重複自己），就要把程式碼移到別的地方。

常見的答案是：「使用 service 然後從 Controller 呼叫它」。好，但 service 怎麼組織？要建立一個 UserService，把所有跟使用者有關的邏輯都放進去，然後注入到所有需要的地方嗎？還是有其他方法？

## 避免讓 service 成為 god class

一開始，你可能會想建立一個 class，把所有套用到特定 Model 的程式碼都放進去。像這樣：

```php
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

看起來不錯：從任何 Controller 都可以呼叫使用者建立／刪除方法，然後用任何方式處理結果。那，這樣有什麼問題？問題是，我們很少會單獨處理單一 Model。

假設建立使用者帳號時，也會建立一個新的部落格。照現在的做法，我們得建立一個 `BlogService` 類別，把它作為 `UserService` 類別的依賴注入：

```php
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

可以預見，隨著應用程式變大，我們會有幾十個服務類別，有些類別會有五六個其他 `Service` 作為依賴，最後就會變成我們想避免的 `Spaghetti code`。

## 介紹 single action 物件

如果不用一個有很多方法的 service ，而是把它拆成很多個 class 呢？這是我最近每個專案都用的方法，效果很好。

首先，我們不要用「**service**」這個詞，它太廣泛了，我們把新 class 叫做「**action**」，然後定義它們是什麼，可以做什麼：

- 每個 action 的的名稱要能清楚表達它的功能，例如: `CreateOrder`, `ConfirmCheckout`, `DeleteProduct` 與 `AddProductToCart` ......等等。
- Action 應該只會有一個 public method，最好都用同一個名稱，例如: `handle()` 或 `execute()`。這會讓往後實作某些 pattern 會容易許多(例如 adapter pattern)。
- Action 不能跟 **request** 與 **response** 綁在一起，也不能處理 **request** 與回傳 **response** 相關的資料，這是 Controller 的責任。
- 所有的 Action 可以被其他 action 所依賴
- 必須透過拋出 `Exception` 來強制執行商業規則。如果有任何事阻止它執行或回傳預期值，就拋出 Exception，讓呼叫者（或 Laravel 的 `ExceptionHandler`）決定怎麼處理。

## 建立 CreateUser 的 action

現在，我們用前面的例子，用 Action class（叫做 `CreateUser`）來重構它。

```php
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
你可能會問，如果 email 已經有人用了，為什麼要拋出 Exception？ 這不是應該在 `request` 驗證處理嗎？ 當然可以。不過，在 action 裡面強制執行商業規則是個好主意。這樣邏輯比較清楚，除錯也比較簡單。

以下是使用 action 重構之後的 controller:

```php
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

現在，不管我們怎麼改使用者註冊流程，`API` 和 `Web` 版本都會跟著變。乾淨俐落。

## 注入 action

假設我們需要一個 action 來匯入 1000 個使用者帳號到我們的應用中。我們可以為這個匯入的邏輯依賴於 `CreateUser` 這個 action：

```php
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

很乾淨吧？我們可以把 `CreateUser` 注入到 `Collection::map()` 裡重複使用，然後回傳包含所有新建立使用者的 `Collectioin`。
我們還可以改進它，例如 email 重複時回傳 `Null Object`，或是把資訊寫到 Log，但你應該懂了。

## 使用修飾模式

現在，假設我們要把每個新註冊的使用者記錄到檔案裡。我們可以把程式碼寫在 action 裡面，也可以用 `裝飾模式`。

```php
class LogCreateUser extends CreateUser
{
    public function execute(array $data) : User
    {
        Log::info("A new user has registered : " . $data['email']);
        return parent::execute($data);
    }
}
```

然後，透過使用 Laravel 的 IoC 容器，我們可以將 `LogCreateUser` class 綁定到 `CreateUser` 上，這樣每次需要 `CreateUser` 時都可以將 `LogCreateUser` 注入。

```php
// AppServiceProvider.php
public function register()
{
    $this->app->bind(CreateUser::class, LogCreateUser::class);
}
```

這樣就可以透過環境變數，來輕鬆控制要不要記錄 Log。

```php
// AppServiceProvider.php
public function register()
{
    if (config('user.log_registration')) {
        $this->app->bind(CreateUser::class, LogCreateUser::class);
    }
}
```

## 總結

一開始，這種方法看起來好像有很多 class 。當然，使用者註冊只是個簡單的例子，目的是讓文章簡短易懂。一旦專案變複雜，它的價值就顯現出來了，因為你知道你的程式碼在哪，界限也很明確。

## 使用 single action 的好處

- 使用小型物件可以避免程式碼重複(`DRY` 原則)，提高重複使用性。符合 **SOLID** 原則。
- 容易單獨測試各種情況。
- 名稱清楚，容易在大型專案中找到程式碼。
- 容易使用裝飾模式。
- 整個專案一致：避免程式碼散落在 Controller、Model 等地方。

當然，這是根據我這幾年用 Laravel 的經驗和我做過的專案，整理出來的方法。這對我來說很有效，我現在連中小型專案都用它。

我很想與你交流想法, 特別是如果你有著不同的方法的話。
