---
title: "在 Laravel 中通过关联关系列进行排序"
subtitle: ""
layout: post
author: "Rucheng Tang"
header-style: text
hidden: false
tags:
  - Laravel
  - Eloquent
---

翻译：[Ordering database queries by relationship columns in Laravel](https://reinink.ca/articles/ordering-database-queries-by-relationship-columns-in-laravel)，推荐阅读原文。

在本文中，我们将探索如何通过 Eloquent 关联关系的值（列）对数据库查询进行排序。例如，可能我们想根据公司名称对用户列表进行排序。

实现起来因关联关系的类型而异。然而，它总是会涉及到按单独的数据库表中的列排序，这就是让它有点棘手的原因，特别是与普通的排序相比。但是，这是很常见的需求。

本文将介绍以下关系类型：

- [Has-one 关联关系](#对-has-one-关联关系列排序)
- [Belongs-to 关联关系](#对-belongs-to-关联关系列排序)
- [Has-many 关联关系](#对-has-many-关联关系列排序)
- [Belongs-to-many 关联关系](#对-belongs-to-many-关联关系排序)


对关联的数据排序
--------------

需要明确的是，我们在这里尝试做的是通过 Eloquent 关联关系的值对数据库查询进行排序。我们不是试图对关系本身的结果进行简单地排序。事实上，您有可能希望对关系本身的结果进行排序，甚至无需从数据库中加载该关系！

然而，以防万一您在阅读本文时想知道如何对关系本身的结果进行排序，您可以使用以下三种方式：

第一种方式，您可以简单地将 order by 语句附加到您的关联关系中：

```php
class Company extends Model
{
    public function users()
    {
        return $this->hasMany(User::class)->orderBy('name');
    }
}
```

现在，每当您调用 `$company->users`（作为集合）或 `$company->users()`（作为查询构建器）时，用户将自动按名称排序。

第二种方式，在延迟预加载的查询语句中进行排序：

```php
class CompaniesController extends Controller
{
    public function show(Company $company)
    {
        $company->load(['users' => function ($query) {
            $query->orderBy('name');
        }]);
        
        return View::make('companies.show', ['company' => $company);
    }
}
```

这种方法使您可以在控制器级别对关联关系进行排序，使用这种方法您可以根据某个条件动态决定是否加载关联数据。

第三种方式，在关联的模型上使用全局范围：

```php
class User extends Model
{
    protected static function booted()
    {
        static::addGlobalScope(fn ($query) => $query->orderBy('name'));
    }
}
```

> 译者注: `booted()` 是在 Laravel7 开始被引入的，Eloquent 在内部首先运行 `boot()` 方法，然后运行 `booted()`。Laravel7 之前前的版本可以使用 `boot()`，不过不要忘记调用 `parent::boot()`：
> 
> ```php
> public static function boot()
> {
>     parent::boot();
>     
>     static::addGlobalScope(fn ($query) => $query->orderBy('name'));
> }
> ```

现在，无论何时运行任何用户数据库查询，包括关系查询，它都会自动按用户名对结果进行排序。

好的，关于对关系数据本身的排序已经清楚了。现在让我们看看根据关系值对父模型进行排序！


对 has-one 关联关系列排序
-----------------------

考虑一个列出用户姓名、电子邮件和公司名称的应用程序。它目前按用户的名字排序，但如果我们想按他们的公司名称排序呢？

| Name | Email | Company |
| ---- | ----- | ------- |
| Adam Wathan | adam.wathan@gmail.com | NothingWorks Inc. |
| Chris Fidao | fideloper@gmail.com | Servers For Hackers |
| Jonathan Reinink | jonathan@reinink.ca | Code Distillery Inc. |
| Taylor Otwell | taylor@laravel.com | Laravel LLC. |

此应用包含一个 `User` 模型，模型下面存在一个 `hasone` company 关联关系，这意味着 company name 存在于 `companies` 表中。

```php
class User extends Model
{
    public function company()
    {
        return $this->hasOne(Company::class);
    }
}
```

实际上，我们可以使用两种方法按它们的公司名称对这些用户进行排序。第一种方式是使用连接：

```php
$users = User::select('users.*')
    ->join('companies', 'companies.user_id', '=', 'users.id')
    ->orderBy('companies.name')
    ->get();
```

让我们来分解下这个查询。

首先，我们只选择 `users` 表中的列，因为默认情况下，Laravel 在使用连接时会选择所有列，包括 `companies` 表中的列。

```php
User::select('users.*')
```

接下来，我们连接 `companies` 表，公司的 `user_id` 等于用户的 `id`。

```php
->join('companies', 'companies.user_id', '=', 'users.id')
```

最后，我们按公司的 `name` 列对记录进行排序。

```php
->orderBy('companies.name')
```

这里是该查询生成的 SQL：

```sql
select users.*
from users
inner join companies on companies.user_id = users.id
order by companies.name asc
```

第二种方式是使用子查询，从 Laravel 6 开始，`orderBy()` 和 `orderByDesc()` 查询构建器方法支持传入 query 参数，而不仅仅是列名。执行此操作时，query 将作为 order by 语句中的子查询执行。

```php
$users = User::orderBy(Company::select('name')
    ->whereColumn('companies.user_id', 'users.id')
)->get();
```

再次，让我们分解这个查询。

首先，在 `orderBy()` 方法中，我们传入一个从 `companies` 表中选择 `name` 的子查询。

```php
Company::select('name')
```

然后我们通过匹配 `companies` 表的 `user_id` 与 `users` 表的 `id` 来限制结果。

```php
->whereColumn('companies.user_id', 'users.id')
```

这里是该查询生成的 SQL：

```sql
select * from users order by (
    select name
    from companies
    where companies.user_id = users.id
) asc
```

虽然第二种方法确实有效，但根据我的经验，对于 has-one 关系的关系，这种方法比连接方法慢得多。以下是我使用 50,000 条用户记录创建的演示应用程序的一些指标：

| 使用连接 | < 1 ms |
| 使用子查询 | 200 ms |

因此，我们在对 has-one 关联的关系进行排序时，一定要首先使用连接方法。


对 belongs-to 关联关系列排序
--------------------------

按 belongs-to 关联的关系进行排序与按 has-one 关联的关系进行排序基本完全相同，只是外键位于相反的表中。为了本文可以作为文档使用，本节将几乎完全复制 has-one 一节的内容。如果您已经阅读过 has-one 一节的内容，您可以直接跳过该节的内容。

考虑一个列出用户姓名、电子邮件和公司名称的应用程序。它目前按用户的名字排序，但如果我们想按他们的公司名称排序呢？

| Name | Email | Company |
| ---- | ----- | ------- |
| Adam Wathan | adam.wathan@gmail.com | NothingWorks Inc. |
| Chris Fidao | fideloper@gmail.com | Servers For Hackers |
| Jonathan Reinink | jonathan@reinink.ca | Code Distillery Inc. |
| Taylor Otwell | taylor@laravel.com | Laravel LLC. |

此应用包含一个 `User` 模型，模型下面存在一个 `belongsTo` company 关联关系，这意味着 company name 存在于 `companies` 表中。

```php
class User extends Model
{
    public function company()
    {
        return $this->belongsTo(Company::class);
    }
}
```

与 has-one 关联的关系一样，我们可以使用两种方法按它们的公司名称对这些用户进行排序。第一种方式是使用连接：

```php
$users = User::select('users.*')
    ->join('companies', 'companies.id', '=', 'users.company_id')
    ->orderBy('companies.name')
    ->get();
```

让我们来分解下这个查询。

首先，我们只选择 `users` 表中的列，因为默认情况下，Laravel 在使用连接时会选择所有列，包括 `companies` 表中的列。

```php
User::select('users.*')
```

接下来，我们连接 `companies` 表，公司的 `id` 等于用户的 `company_id`。

```php
->join('companies', 'companies.id', '=', 'users.company_id')
```

最后，我们按公司的 `name` 列对记录进行排序。

```php
->orderBy('companies.name')
```

这里是该查询生成的 SQL：

```sql
select users.*
from users
inner join companies on companies.id = users.company_id
order by companies.name asc
```

第二种方式是使用子查询，从 Laravel 6 开始，`orderBy()` 和 `orderByDesc()` 查询构建器方法支持传入 query 参数，而不仅仅是列名。执行此操作时，query 将作为 order by 语句中的子查询执行。

```php
$users = User::orderBy(Company::select('name')
    ->whereColumn('companies.id', 'users.company_id')
)->get();
```

再次，让我们分解这个查询。

首先，在 `orderBy()` 方法中，我们传入一个从 `companies` 表中选择 `name` 的子查询。

```php
Company::select('name')
```

然后我们通过匹配 `companies` 表的 `user_id` 与 `users` 表的 `id` 来限制结果。

```php
->whereColumn('companies.id', 'users.company_id')
```

这里是该查询生成的 SQL：

```sql
select * from users order by (
    select name
    from companies
    where companies.id = users.id
) asc
```

同样的，与 has-one 关联的关系一样，虽然子查询方法确实有效，但比连接方法慢得多。以下是我使用 50,000 条用户记录创建的演示应用程序的一些指标：

| 使用连接 | < 1 ms |
| 使用子查询 | 60 ms |

因此，我们在对 belongs-to 关联的关系进行排序时，一定要首先使用连接方法。


对 has-many 关联关系列排序
------------------------

有一个列出用户姓名、电子邮件和上次登录日期的应用程序。它目前是按用户的名字对用户进行排序，但是如果我们想按他们的上次登录日期来排序呢？

| Name | Email | Last login |
| ---- | ----- | ---------- |
|Adam Wathan | adam.wathan@gmail.com | 3 months ago |
|Chris Fidao | fideloper@gmail.com | 8 seconds ago |
|Jonathan Reinink | jonathan@reinink.ca | 6 days ago |
|Taylor Otwell | taylor@laravel.com | A week ago |

（顺便说一句，如果您想知道如何以最有效的方式获取上次登录日期，请务必阅读我的 [动态关联关系](https://reinink.ca/articles/dynamic-relationships-in-laravel-using-subqueries) 文章。）

这个应用程序包含一个具有 `hasMany` logins 关系的 `User` 模型。这意味着，登录信息存在于 `logins` 表中。每次用户登录时，都会在此表中创建一条新记录。

```php
class User extends Model
{
    public function logins()
    {
        return $this->hasMany(Login::class);
    }
}
```

实际上，我们可以使用两种方法对 has-many 关联关系进行排序。这可以通过连接或子查询来完成。让我们从子查询方法开始，因为它更简单。

从 Laravel 6 开始， `orderBy()` 和 `orderByDesc()` 查询构建器方法支持传入 query 参数，而不仅仅是列名。执行此操作时，查询将作为 order by 语句中的子查询执行。

```php
$users = User::orderByDesc(Login::select('created_at')
    ->whereColumn('logins.user_id', 'users.id')
    ->latest()
    ->take(1)
)->get();
```

让我们仔细看看这个子查询。

首先，我们从 `logins` 表中选择 `created_at` 列。

```php
Login::select('created_at')
```

然后我们通过将登录 `user_id` 列与父查询中用户的 `id` 进行比较来限制结果。

```php
->whereColumn('logins.user_id', 'users.id')
```

然后我们调用 `latest()` 方法，该方法对 logins 排序获取最近的记录。

```php
->latest()
```

最后，我们将结果限制为仅一行，因为子查询只能返回单行和单列，但用户（很可能）将拥有多个登录记录。

```php
->take(1)
```

这是为这个查询生成的 SQL，它包括我们在 order by 语句中的登录子查询。

```sql
select * from users order by (
    select created_at
    from logins
    where user_id = users.id
    order by created_at desc
    limit 1
) desc
```

为了更容易重用，我通常会将这个排序子查询使用 model scope 进行封装。例如：

```php
public function scopeOrderByLastLogin($query, $direction = 'desc')
{
    $query->orderBy(Login::select('created_at')
        ->whereColumn('logins.user_id', 'users.id')
        ->latest()
        ->take(1),
        $direction
    );
}
```

现在您可以简单地从您的控制器（或任何你需要的地方）调用这个 scope：

```php
$users = User::orderByLastLogin()->get();
```

很好，现在让我们来看看 join 方法。

```php
$users = User::select('users.*')
    ->join('logins', 'logins.user_id', '=', 'users.id')
    ->groupBy('users.id')
    ->orderByRaw('max(logins.created_at) desc')
    ->get();
```

让我们来分解下这个查询。

首先我们只选择 `users` 表中的列，因为默认情况下，Laravel 在使用连接时会选择所有列，包括 `logins` 表中的列。

```php
User::select('users.*')
```

接下来，我们连接 `logins` 表，其中登录 `user_id` 等于用户的 `id`。

```php
->join('logins', 'logins.user_id', '=', 'users.id')
```

接下来，我们按用户的 `id` 对用户进行分组，因为我们只需要每个用户一行。

```php
->groupBy('users.id')
```

最后，这就是事情变得更有趣的地方，我们对 `max` 登录记录 `created_at` 列进行降序排序，获取用户最近一次的登录信息。

```php
->orderByRaw('max(logins.created_at) desc')
```

这里是该查询生成的 SQL：

```sql
select users.*
from users
inner join logins on logins.user_id = users.id
group by users.id
order by max(logins.created_at) desc
```

您可能想知道为什么我们在这里使用 `max()` 聚合函数。我们不能只按 `created_at` 列排序吗？如：

```php
->orderByDesc('logins.created_at')
```

答案是：不能。

如果没有 `max()` 聚合函数，我们会得到一个语法错误：

> Syntax error or access violation: 1055 Expression #1 of ORDER BY clause is not in GROUP BY clause and contains nonaggregated column 'logins.created_at' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by

这到底是什么意思？

好吧，由于我们连接了登录，我们为每个用户获取了多个记录。他们拥有的每条登录记录对应一行。

当然，我们只希望每个用户有一条记录，这就是我们按用户 `id` 对它们进行分组的原因。

但是，我们然后告诉 MySQL 按 login `created_at` 列对这些分组的行进行排序。但是，如果每个用户有多个行，这意味着我们为每个用户有多个不同的登录 `created_at` 值。 MySQL 如何知道根据 `created_at` 值中的哪一个对用户进行排序？

其实，Mysql 不知道如何排序。这就是为什么我们会得到这个错误。

这里要意识到的重要一点是，当查询执行时，order by 出现在 group by 之后。这意味着 order by 语句在分组的行上执行。而且，与使用 select 语句执行此操作时一样，您必须使用聚合 (group by) 函数来告诉 MySQL 您要使用该分组集中的哪个值。

这就是我们在查询中使用 `max()` 聚合函数的原因。它告诉 MySQL 我们想要分组集中的最新（最大） `created_at` 值。这也适用于其他列类型。例如，您可以使用 `max()` 和 `min()` 函数按字母顺序对字符串列进行排序。有关这些聚合函数的完整列表，请参阅 [MySQL 文档](https://dev.mysql.com/doc/refman/8.0/en/aggregate-functions.html)。

这是我用来记住查询执行顺序的一个简单摘要。

| 顺序 | 关键字 | 作用 |
| ---  | --- - | --- |
| 1	| from | 选择并连接表以获取数据 |
| 2	| where | 过滤数据 |
| 3	| group by | 聚合数据 |
| 4	| having | 过滤聚合的数据 |
| 5	| select | 选择要返回的数据 |
| 6	| order by | 对数据进行排序 |
| 7	| limit/offset | 将数据限制在某些行 |

很好，这两种方法中哪一种最快？联接还是子查询？以下是我使用 50,000 条用户记录创建的演示应用程序的一些指标：

| Using a join | approx. 300ms |
| Using a subquery | approx. 300ms |

如您所见，这里没有决定性的赢家。因此，当按 has-many 关系排序时，我建议尝试这两种方法，看看哪种方法最适合您的用例。

另外，只是一个简短的说明，为了获得一些速度，我必须在登录表上为 `user_id` 和 `created_at` 列创建一个复合索引。

```php
Schema::table('logins', function (Blueprint $table) {
    $table->index(['user_id', 'created_at']);
});
```


对 belongs-to-many 关联关系排序
-----------------------------

思考一个图书馆应用程序，它列出了书的标题、作者以及上次结帐用户和日期。它目前是按书名对书籍进行排序的，但如果我们想按最后结帐日期或结账用户名进行排序怎么办？

| Book | Last Checkout |
| ---- | ------------- |
| Clean Code: A Handbook of Agile Software Craftsmanship <br> Robert C. Martin | Matt Stauffer <br> 6 years ago |
| Patterns of Enterprise Application Architecture <br> Martin Fowler | Freek Van der Herten <br> 8 months ago |
| PHP and MySQL Web Development <br> Luke Welling | Jonathan Reinink <br> 4 years ago |
| Test Driven Development: By Example <br> Kent Beck | Adam Wathan <br> 19 years ago |
| The Pragmatic Programmer: From Journeyman to Master <br> Andy Hunt | Taylor Otwell <br> 6 years ago |
| Working Effectively with Legacy Code <br> Michael C. Feathers | Caleb Porzio <br> 1 day ago |

```php
class Book extends Model
{
    public function user()
    {
        return $this->belongsToMany(User::class, 'checkouts')
            ->using(Checkout::class)
            ->withPivot('borrowed_date');
    }
}

class Checkout extends Pivot
{
    protected $table = 'checkouts';

    protected $casts = [
        'borrowed_date' => 'date',
    ];
}
```

让我们根据最后借用日期开始排序。

我们很容易实现这个排序。因为 `borrowed_date` 存在于我们的 `checkouts` 中间表中。`checkouts` 本质上与 `book` 为 has-many 关系。一本书具有多条结账记录。我们只是碰巧使用 `checkouts` 作为 `books` 和 `users` 之间的多对多关系的数据中间表。

这意味着，如果我们想按 `checkouts` 表上的列进行排序，我们可以使用我们在上面的 [has-many 关系](#对-has-many-关联关系列排序) 部分中介绍的技术来实现。

下面是使用子查询方式进行排序：

```php
$books = Books::orderByDesc(Checkout::select('borrowed_date')
    ->whereColumn('book_id', 'books.id')
    ->latest('borrowed_date')
    ->limit(1)
)->get();
```

您可能想知道，“如果我不使用 `Checkout` 模型，我还能这样做吗？”

答案是肯定的。从 Laravel 6 开始，order by 方法还允许您提供一个闭包，您可以在其中手动编写子查询。

```php
$books = Books::orderByDesc(function ($query) {
    $query->select('borrowed_date')
        ->from('checkouts')
        ->whereColumn('book_id', 'books.id')
        ->latest('borrowed_date')
        ->limit(1);
})->get();
```

这是这两个查询生成的 SQL（它们是相同的）：

```sql
select * from books order by (
    select borrowed_date
    from checkouts
    where book_id = books.id
    order by borrowed_date desc
    limit 1
) desc
```

好的，现在让我们按 belongs-to-many 关系列进行排序。

在我们的示例中，这意味着按 `users` 表中的列进行排序。让我们更新 books 查询以按最后结帐用户 `name` 进行排序。

同样，我们将使用 order by 子查询来实现：

```php
$books = Book::orderBy(User::select('name')
    ->join('checkouts', 'checkouts.user_id', '=', 'users.id')
    ->whereColumn('checkouts.book_id', 'books.id')
    ->latest('checkouts.borrowed_date')
    ->take(1)
)->get();
```

让我们仔细看看这个子查询。

首先，我们从 `users` 表中选择 `name`，因为这是我们想要排序的列。

```php
User::select('name')
```

然后我们 join `checkouts` 表，让其 `user_id` 等于用户的 `id`。我们需要这个连接，因为 `checkouts` 表将 `books` 与 `users` 连接起来。

```php
->join('checkouts', 'checkouts.user_id', '=', 'users.id')
```

接下来，我们将 checkout 的 `book_id` 与 book 的 `id` 进行比较，将结果限制为该图书的结账信息。

```php
->whereColumn('checkouts.book_id', 'books.id')
```

然后我们通过 `borrowed_date` 列对结帐进行排序以获取最新的数据。

```php
->latest('checkouts.borrowed_date')
```

最后，我们取第一条记录，因为子查询只能返回单行和单列，当然，因为我们只关心最后一次结账。

```php
->take(1)
```

这是该查询生成的 SQL：

```sql
select * from books order by (
    select name
    from users
    inner join checkouts on checkouts.user_id = users.id
    where checkouts.book_id = books.id
    order by checkouts.borrowed_date desc
    limit 1
) asc
```

需要注意的是，虽然这种技术在某些情况下非常有用，但如果您要处理数万条记录，这种方法可能不会非常快。因此，如果您遇到这种技术导致性能问题的情况，这可能是引入一些缓存的好时机。

例如，我可能首先将 `last_checkout_id` 外键添加到 `books` 表，作为非规范化的第一步。

```php
Schema::table('books', function (Blueprint $table) {
    $table->foreignId('last_checkout_id')->nullable()->constrained('checkouts');
});
```
