---
title: "Laravel 中使用子查询动态获取关联数据"
subtitle: ""
layout: post
author: "Rucheng Tang"
header-style: text
hidden: false
tags:
  - Laravel
  - Eloquent
---

翻译：[Dynamic relationships in Laravel using subqueries](https://reinink.ca/articles/dynamic-relationships-in-laravel-using-subqueries)，推荐阅读原文。

在构建与数据库交互的 Web 应用程序时，我一直有两个目标：

1. 将数据库查询数保持在最低限度。 
2. 将内存的使用量保持在最低水平。

这些目标会对您的应用程序的性能产生巨大影响。

开发人员通常非常擅长第一个目标。我们知道出现 N+1 查询的问题原因，并使用预加载等技术来限制数据库查询。然而，在第二个目标中不总是最好的 —— 保持内存使用量在最低水平。事实上，有时我们试图以牺牲内存使用为代价来减少数据库查询，弊大于利。

让我分析一下这是如何发生的，以及您可以做些什么来在您的应用中实现这两个目标。


挑战
----

研究以下示例。您的应用中有一个用户页面，其中显示了有关它们的一些信息，包括它们上次登录的日期。这个看似简单的页面实际上呈现了一些有趣的复杂性。

| Name | Email | Last Login |
| ---- | ----- | ---------- |
| Adam Campbell	| adam@hotmeteor.com | Nov 10, 2018 at 12:01pm |
| Taylor Otwell | taylor@laravel.com | Never |
| Jonathan Reinink | jonathan@reinink.ca | Jun 2, 2018 at 5:30am |
| Adam Wathan | adam.wathan@gmail.com | Nov 20, 2018 at 7:49pm |

在这个应用程序中，我们在 `logins` 表中跟踪用户登录信息，因此我们可以对其进行统计报告。以下是基本数据库结构：

```php
Schema::create('users', function (Blueprint $table) {
    $table->increments('id');
    $table->string('name');
    $table->string('email');
    $table->timestamps();
});

Schema::create('logins', function (Blueprint $table) {
    $table->increments('id');
    $table->integer('user_id');
    $table->string('ip_address');
    $table->timestamp('created_at');
});
```

以下是这些表及其关系的相应模型：

```php
class User extends Model
{
    public function logins()
    {
        return $this->hasMany(Login::class);
    }
}

class Login extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

那么我们如何创建上面的用户页面呢？特别是，我们如何获得上次登录日期？最简单的方式可能是执行以下操作：

```php
$users = User::all();

@foreach ($users as $user)
    <tr>
        <td>{{ $user->name }}</td>
        <td>{{ $user->email }}</td>
        <td>
            @if ($lastLogin = $user->logins()->latest()->first())
                {{ $lastLogin->created_at->format('M j, Y \a\t g:i a') }}
            @else
                Never
            @endif
        </td>
    </tr>
@endforeach
```

但是，如果我们是一名优秀的开发人员（而且我们是），我们就会注意到这里的一个问题。我们刚刚创建了一个 N+1 问题。对于我们显示的每个用户，我们现在正在运行一个额外的查询来获取它们的最后一次登录信息。如果我们的页面显示 50 个用户，我们现在总共执行 51 个查询。

```sql
select * from "users";
select * from "logins" where "logins"."user_id" = 1 and "logins"."user_id" is not null order by "created_at" desc limit 1;
select * from "logins" where "logins"."user_id" = 2 and "logins"."user_id" is not null order by "created_at" desc limit 1;
// ...
select * from "logins" where "logins"."user_id" = 49 and "logins"."user_id" is not null order by "created_at" desc limit 1;
select * from "logins" where "logins"."user_id" = 50 and "logins"."user_id" is not null order by "created_at" desc limit 1;
```

让我们看看我们是否可以做得更好。这次让我们使用预加载的方式来获取所有登录记录：

```php
$users = User::with('logins')->get();

@foreach ($users as $user)
    <tr>
        <td>{{ $user->name }}</td>
        <td>{{ $user->email }}</td>
        <td>
            @if ($user->logins->isNotEmpty())
                {{ $user->logins->sortByDesc('created_at')->first()->created_at->format('M j, Y \a\t g:i a') }}
            @else
                Never
            @endif
        </td>
    </tr>
@endforeach
```

该解决方案只需要两次数据库查询。一个用于用户，第二个用于相应的登录记录。成功！

嗯，不完全是。这就是内存问题成为问题的地方。当然，我们已经避免了 N+1 问题，但我们实际上创造了一个更大的内存问题：

| 每个页面用户数 | 50 |
| 平均每个用户的登录记录数 | 250 条 |
| 加载的登录记录总数 |12,500 条 |

我们现在正在加载 12,500 条登录记录，只是为了显示每个用户的最后一次登录信息。这不仅会消耗内存，还会需要额外的计算，因为每条记录都必须初始化为 Eloquent 模型。这是一个相当保守的例子。您很容易遇到导致加载数百万条记录的类似情况。


缓存
----

此时您可能会想，“没什么大不了的，将 `last_login_id` 缓存在 `users` 表中即可”。例如：

```php
Schema::create('users', function (Blueprint $table) {
   $table->integer('last_login_id');
});
```

现在，当用户登录时，我们将创建新的登录记录，然后更新用户的 `last_login_id` 外键。然后我们将在我们的用户模型上创建一个 `lastLogin` 关系，并预先加载该关系。

```php
$users = User::with('lastLogin')->get();
```

这是一个完全有效的解决方案。但请注意，缓存通常并非如此简单。是的，在某些情况下，反规范化是绝对合适的。我只是不喜欢这样使用，因为我的 ORM 存在明显的局限性。我们可以做得更好。


引入子查询
---------

还有另一种方法可以解决这个问题，那就是使用子查询。子查询允许我们在我们的数据库查询（我们的示例中的用户查询）中选择额外的列（属性）。让我们看看我们如何做到这一点。

```php
$users = User::query()
    ->addSelect(['last_login_at' => Login::select('created_at')
        ->whereColumn('user_id', 'users.id')
        ->latest()
        ->take(1)
    ])
    ->withCasts(['last_login_at' => 'datetime'])
    ->get();
    
@foreach ($users as $user)
    <tr>
        <td>{{ $user->name }}</td>
        <td>{{ $user->email }}</td>
        <td>
            @if ($user->last_login_at)
                {{ $user->last_login_at->format('M j, Y \a\t g:i a') }}
            @else
                Never
            @endif
        </td>
    </tr>
@endforeach
```

在这个例子中，我们实际上还没有加载动态关系。那来了。我们正在做的是使用子查询来获取每个用户的最后登录日期作为属性。我们还利用[查询时间转换(query time casting)](https://laravel.com/docs/7.x/eloquent-mutators#query-time-casting)将 `last_login_at` 属性转换为 `Carbon` 实例。

让我们看看实际运行的数据库查询：

```sql
select
    "users".*,
    (
        select "created_at" from "logins"
        where "user_id" = "users"."id"
        order by "created_at" desc
        limit 1
    ) as "last_login_at"
from "users"
```

以这种方式使用子查询允许我们在单个查询中获取用户页面所需的所有信息。这种技术提供了巨大的性能优势，因为我们可以将数据库查询和内存使用量保持在最低限度，而且我们避免了使用缓存。


Scope
-----

在我们进行下一步之前，让我们将新的子查询移动到 `User` 模型的 [scope](https://laravel.com/docs/5.7/eloquent#query-scopes) 内：

```php
class User extends Model
{
    public function scopeWithLastLoginDate($query)
    {
        $query->addSelect(['last_login_at' => Login::select('created_at')
            ->whereColumn('user_id', 'users.id')
            ->latest()
            ->take(1)
        ])->withCasts(['last_login_at' => 'datetime']);
    }
}

$users = User::withLastLoginDate()->get();
```

我喜欢将查询构建器代码隐藏到这样的模型 scope 中。它不仅使控制器更简单，还允许更轻松地重用这些查询。另外，它将帮助我们进行下一步，通过子查询加载动态关系。


通过子查询的动态关系
-----------------

好的，现在是我们一直在构建的部分。使用子查询来获取上次登录日期很好，但是如果我们想要有关上次登录的其它信息怎么办？例如，也许我们还想显示该登录的 IP 地址。我们将如何做到这一点？

一种选择是简单地创建第二个子查询 scope：

{% highlight php %}
{% raw %}
$users = User::withLastLoginDate()
    ->withLastLoginIpAddress()
    ->get();

{{ $user->last_login_at->format('M j, Y \a\t g:i a') }}
({{ $user->last_login_ip_address }})
{% endraw %}
{% endhighlight %}

而且，这肯定会奏效。但是，如果有很多属性，这可能会变得乏味。使用实际的登录模型实例不是更好吗？特别是如果该模型具有内置的附加功能，例如访问器或关系。像这样的东西：

{% highlight php %}
{% raw %}
$users = User::withLastLogin()->get();

{{ $user->lastLogin->created_at->format('M j, Y \a\t g:i a') }}
({{ $user->lastLogin->ip_address }})
{% endraw %}
{% endhighlight %}

输入动态关系。

我们将从定义一个新的 `lastLogin` 所属关系开始。现在通常要使归属关系起作用，您的表需要一列作为外键。在我们的示例中，这意味着我们的 `users` 表上有一个 `last_login_id` 列。然而，由于我们试图避免实际必须反规范化并将该数据存储在 `users` 表中，因此我们将使用子查询来选择外键。Eloquent 不知道这不是一个真正的列，这意味着一切都像以前一样工作。让我们看一下代码：

```php
class User extends Model
{
    public function lastLogin()
    {
        return $this->belongsTo(Login::class);
    }

    public function scopeWithLastLogin($query)
    {
        $query->addSelect(['last_login_id' => Login::select('id')
            ->whereColumn('user_id', 'users.id')
            ->latest()
            ->take(1)
        ])->with('lastLogin');
    }
}

$users = User::withLastLogin()->get();

<table>
    <tr>
        <th>Name</th>
        <th>Email</th>
        <th>Last Login</th>
    </tr>
    @foreach ($users as $user)
        <tr>
            <td>{{ $user->name }}</td>
            <td>{{ $user->email }}</td>
            <td>
                @if ($user->lastLogin)
                    {{ $user->lastLogin->created_at->format('M j, Y \a\t g:i a') }}
                @else
                    Never
                @endif
            </td>
        </tr>
    @endforeach
</table>
```

就这些！这里的最终结果是两个数据库查询。让我们来看看它们：

```sql
select
    "users".*,
    (
        select "id" from "logins"
        where "user_id" = "users"."id"
        order by "created_at" desc
        limit 1
    ) as "last_login_id"
from "users"
```

这个查询与我们之前看到的查询基本完全相同，只是我们选择的是最后一次登录 ID，而不是选择最后一次登录日期。如果我们缓存该值，我们基本上已经获得了 `last_login_id` 列，而实际上不必缓存它。

现在让我们看看第二个查询。这是 Laravel 在我们使用 `with('lastLogin')` 预加载最后一次登录信息时自动运行的查询，你可以看到在我们的 scope 中调用了它。

```sql
select * from "logins" where "logins"."id" in (1, 3, 5, 13, 20 ... 676, 686)
```

我们的子查询允许我们只选择每个用户的最后一次登录。另外，由于我们在 `lastLogin` 中使用了标准的 Laravel 关系，因此我们还将这些记录作为正确的 `Login` Eloquent 模型。最后，我们不再需要查询时间转换，因为登录模型会自动处理 `created_at` 属性。非常好。


延迟加载动态关系
--------------

使用此技术需要注意的一件事是，您不能直接使用延迟加载动态关系。这是因为默认情况下不会添加我们的 scope。

```php
$lastLogin = User::first()->lastLogin; // will return null
```

如果您希望延迟加载工作，您仍然可以做到，需要通过向模型添加全局 scope 来实现：

```php
class User extends Model
{
    protected static function booted()
    {
        static::addGlobalScope('with_last_login', function ($query) {
            $query->withLastLogin();
        });
    }
}
```

我不会经常这样做，因为我通常更喜欢在需要时显式地预先加载我的动态关系。


这可以用 has-one 来完成吗？
------------------------

在我们结束之前的最后一件事。此时您可能想知道我们是否可以通过简单地使用一对一关系来避免所有这些工作。最简洁的答案是不。让我们看看为什么。 您可能会想到的第一种方法是对 has-one 查询进行排序：

```php
class User extends Model
{
    public function lastLogin()
    {
        return $this->hasOne(Login::class)->latest();
    }
}

$lastLogin = User::first()->lastLogin;
```

而且，起初，这实际上似乎提供了预期的结果。访问我们用户的 lastLogin 关系将提供他们正确的 last Login 实例。但是，如果我们查看生成的查询，我们会发现一个问题：

```sql
select * from "logins"
where "logins"."user_id" in (1, 2, 3...99, 100)
order by "created_at" desc
```

它是通过 `user_id` 快速加载登录，但没有设置限制或过滤器。这意味着这不仅会加载最后一次登录，还会加载所有用户的每条登录记录。我们马上回到我们之前看到的 12,500 条登录记录问题。

但是，我们有决心！所以，我们添加一个限制：

```php
public function lastLogin()
{
    return $this->hasOne(Login::class)->latest()->take(1);
}
```

看起来它应该可以工作，但让我们看看生成的查询：

```sql
select * from "logins"
where "logins"."user_id" in (1, 2, 3...99, 100)
order by "created_at" desc
limit 1
```

Laravel 在单个数据库查询中预先加载关系，但我们现在添加了 1 的限制。这意味着我们只会为所有用户返回一条记录。这将是最后一个用户登录的登录记录。所有其他用户的 `lastLogin` 关联关系设置为 `null`。


总结
---

我希望这能让你很好地了解如何使用子查询在 Laravel 中创建动态关系。这是一项强大的技术，可让您将更多工作推入应用程序的数据库层。通过允许您大幅减少执行的数据库查询数量和使用的总内存，这会对性能产生巨大影响。

如果您对本文有任何问题或反馈，请在 Twitter 上向我 ([@reinink](https://twitter.com/reinink)) 发送消息。我很想听听你的意见。
