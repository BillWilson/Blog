# You Don't Really Need Repository Pattern in Laravel

![random sunrise image](https://source.unsplash.com/random/1600x900/?sunrise)

When we decide to use design patterns in development, we hope to achieve several goals:

- Reduce duplicate code
- Reduce bugs 
- Separate data and business logic to make testing easier
- Improve code readability

So before choosing any pattern, the project design should meet the principles above.

Before discussing why, let's first understand Repository Pattern. This is Microsoft's explanation of Repository Pattern:

> Use a repository to separate the logic that retrieves the data and maps it to the entity model from the business logic that acts on the model. The business logic should be agnostic to the type of data that comprises the data source layer. For example, the data source layer can be a database, a SharePoint list, or a Web service. 
> ![](https://learn.microsoft.com/zh-tw/previous-versions/msp-n-p/images/ff649690.4058e458-bd54-4597-845e-6f8b1a21cfc3(en-us,pandp.10).png)
> From: [learn.microsoft.com](https://learn.microsoft.com/zh-tw/previous-versions/msp-n-p/ff649690(v=pandp.10))

The implementation of Repository Pattern perfectly applies the [Dependency Inversion Principle (DIP)](https://en.wikipedia.org/wiki/Dependency_inversion_principle).

In practice, it can be done like this:

```php
# Bind the Interface with the implemented Repository
class AppServiceProvider extends ServiceProvider
{
  public function register(): void
  {
    $this->app->bind(PostRepositoryInterface::class, PostEloquentRepository::class);
    $this->app->bind(AuthorRepositoryInterface::class, AuthorManagementRepository::class);
  }
}

# Then inject where needed  
class PostController extends Controller
{
  public function __invoke(PostRepositoryInterface $repository)
  {
    // ...
    $data = $repository->get();
  }
}
```

However, to meet interchangeability, the repository methods cannot return Eloquent models directly, otherwise the repository will depend on Eloquent. The repository created this way is just a wrapped Eloquent query.

To separate Eloquent from here is the only way to make the repository interchangeable.

## Do You Really Need It?

Remember [YAGNI](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it) we talked about before? In development, try to avoid assuming future features will need certain design patterns. Using non-default tools without proper documentation often buries more landmines for future maintenance. 

[Over-engineering](https://yurenju.medium.com/over-engineering-685ebc009fca) is what kills many project maintainers. For example, the decision on which database to use rarely changes arbitrarily. Even if it needs to change, functional changes are often required. So the scenario where repositories can be switched seamlessly is nearly impossible.

## Repository Pattern Always Lags Behind 

Think about it. After the first stage of the product is launched, the PM finds more features are needed, such as:

We have implemented a method to list all posts under a certain author `all()`. Now the boss sees some successful Silicon Valley companies start paid content, and wants to categorize posts into free ones and premium ones only visible to paid members.

Now we rack our brains to take the challenge. Here are some possible solutions:

- Add a `bool $isPremium = true` parameter to `all()`
- Add a separate method for free users, like `allForFree()` 
- Or create a separate Repository for premium members

What if the requirement evolves to allow `unregistered` users to experience premium functions, so visitors can see those premium-only posts? How will your Repository design evolve?

Now we realize extending the Repository is rather difficult, while keeping up with feature development and avoiding [Code Smells](https://en.wikipedia.org/wiki/Code_smell). In the end we get a huge and complex Repository.

## Solutions

Now back to Laravel projects, what are our options without Repository Pattern?

### Method 1: Use Eloquent Query Builder

Below is an example using custom Query Builder:

```php
// Channel.php
use App\Models\Builder\ChannelBuilder;
use Illuminate\Database\Eloquent\Builder;

class Channel extends Base
{
    // ...
    public static function query(): ChannelBuilder|Builder
    {
        return parent::query();
    }

    public function newEloquentBuilder($query): ChannelBuilder
    {
        return new ChannelBuilder($query);
    }
    // ...
}

// ChannelBuilder.php
class ChannelBuilder extends Builder
{
    use Common;

    // ...
    public function isPublished(): self
    {
        return $this->whereStatus(ChannelStatus::published);
    }
}

// Common.php
trait Common
{
    public function whereStatus(int|BackedEnum $status): self
    {
        if ($status instanceof BackedEnum) {
            $status = $status->value;
        }

        return $this->where('status', $status);
    }
}

// ChannelStatus.php
enum ChannelStatus :int
{
    case pending = 0;
    case published = 1;
    case unpublished = 2;
}
```

In this example, not only can querying follow the [DRY principle](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself), it allows better separation of common queries on the Model. During development, IDE can also find corresponding query methods through static analysis.

Note: Based on above, I don't recommend using Laravel [query scopes](https://laravel.com/docs/10.x/eloquent#query-scopes).

### Method 2: Use Custom Query Components

```php
class LatestEpisodeQuery
{
    public function __constructor(
        private readonly Channel $channel,
        private readonly ?int    $limit = null,
    ): void
    {
        $this->limit ??= config('episodes.limit');
    }

    public function get(): Collection
    {
        return $this
            ->channel
            ->episodes()
            ->isPublished()
            ->isPremium()
            ->latest()
            ->take($this->limit)
            ->get();
    }
}

// usage
(new LatestEpisodeQuery($channel))->get();
```

In this example, common queries can be extracted into `query()`, then combined into complex queries needed. 

### Method 3: Use Actions to Organize Contexts

An Action class can contain simple CRUD, complex multi-model data processing, a combination of CRUD-only repository and business logic.

Actions are highly reusable and composable, but there are no mainstream guidelines. Some rules to try:

- Use Composite pattern, no inheritance
- Use Dependency Injection for decoupling 
- Only one `public method` like `execute()` or `run()`, other methods `private`

See this [article](https://yhhbill.dev/posts/2020-09-17-%E8%AE%93action%E5%B9%AB%E4%BD%A0%E9%81%B5%E5%AE%88dry%E5%8E%9F%E5%89%87/#%E4%BD%BF%E7%94%A8%E4%BF%AE%E9%A3%BE%E6%A8%A1%E5%BC%8F) for examples.

Although this is **Method 3**, I actually use it with `1` and `2` which works well for most scenarios, even slightly complex ones.

## End

In the end, I want to emphasize the above are not just theoretical methods you should know, but can be applied and implemented immediately with real benefits. Please leave comments to discuss more.

## Reference

* Actions - Stitcher.io. (n.d.). https://stitcher.io/blog/laravel-beyond-crud-03-actions
* Bill. (2020, September 17). 讓 Action 幫你遵守 DRY 原則. B2’s Stage. https://yhhbill.dev/posts/2020-09-17-%E8%AE%93action%E5%B9%AB%E4%BD%A0%E9%81%B5%E5%AE%88dry%E5%8E%9F%E5%89%87/ 
* Dike, O. (2022, November 17). Laravel Repository Design Pattern- ( A Quick Demonstration ). Medium. https://blog.devgenius.io/laravel-repository-design-pattern-a-quick-demonstration-5698d7ce7e1f
* 仓库模式被误用了吗?—— laravel-腾讯云开发者社区-腾讯云. (n.d.). https://cloud.tencent.com/developer/article/1491737
