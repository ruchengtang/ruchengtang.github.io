---
title: "PHP 错误处理"
subtitle: "不要给调用你代码的人增加麻烦"
layout: post
author: "Rucheng Tang"
header-style: text
hidden: false
tags:
  - PHP 异常
---

我们在程序开发过程中，不光要考虑程序的默认行为，同时还得考虑各种非正常情况。简言之，程序可能会出错，当错误发生时，我们就有责任确保代码照常工作。在我们的日常工作中，会发现许多程序完全由错误处理所占据。所谓占据，并不是说错误处理就是全部。我的意思是几乎无法看明白代码所做的事，因为到处都是凌乱的错误处理代码。如何处理错误呢？

先来一个简单的示例，我们通过 `UserRepository` 获取一个 `User` 实例，然后我们想通知与该用户相关的消息，检查用户是否已经订阅过，以免骚扰用户。

`DbUserRepository` 中提供了一个 `getByName()` 方法，存在用户时，会返回一个 `User` 实例，不存在时则返回 `null`。

```php
interface UserRepository
{
    public function getByName(string $name): ?User;
}
```

这里使用了返回值类型声明，php7 的特性，我们不书写注释，也可以直接通过代码进行交流，不然一定要在注释里声明返回值的类型。

```php
interface UserRepository
{
    /**
     * @param string $name username
     *
     * @return User|null User instance or null if User does not exist
     */
    public function getByName(string $name);
}
```

```php
final class DbUserRepository implements UserRepository
{
    private $db;

    public function __construct(DB $db)
    {
        $this->db = $db;
    }

    public function getByName(string $name)
    {
        $userRecords = $this->db->select('select * from users where name = :name', ['name' => $name]);

        if (0 === $userRecords->count()) {
            return null;
        }

        return $userRecords->first();
    }
}

```

下面这段代码很漂亮，简单易懂的代码。

```php
$user = $userRepository->getByName($username);

if ($user->isSubscribedTo($notification)) {
    $notifier->notify($user, $notification);
}
```

看似没啥问题，我们很快就看到了问题，当查找的用户名称在数据库不存在时，此时程序会报错：

```
PHP Error:  Call to a member function isSubscribedTo() on null ...
```

这个问题很简单，我们很快就知道如何处理：

```php
$user = $userRepository->getByName($username);

if (null !== $user && $user->isSubscribedTo($notification)) {
    $notifier->notify($user, $notification);
}
```

但是现在的代码很丑不易于维护，而且很容易引发问题。

我敢打赌，在你的工作项目中肯定看过下面这段类似的代码

```php
$user = $userRepository->getByName($username);

if (null !== $user) {
    $todo = $todoRepository->get('Book flights');

    if (null !== $todo) {
        $notification = TodoReminder::from($todo, $user);

        if (null !== $notification) {
            if ($user->isSubscribedTo($notification)) {
                $notifier->notify($user, $notification);
            }
        }
    }
}
```

不停地在做检查，`null` 检查泛滥。我们甚至见到过下面这段类似的代码：

```php
class UserService
{
    function login(string $username, string $password)
    {
        if (...) {
            // user does not exist
            return -1;
        }
        
        if (...) {
            // wrong password
            return -2;
        }
        
        if (...) {
            // Too many attempts, please try again later
            return -3;
        }
        
        // ...
        
        // login success
        return 1;
    }
}
```

```php
class UserController
{
    public function login(string $username, string $password)
    {
        $userService = new UserService();
        
        $res = $userService->login($username, $password);
        
        if (-1 === $res) {
            return [
                'code' => '404',
                'message' => 'user does not exist',
            ];
        }
        
        if (-2 === $res) {
            return [
                'code' => '403',
                'message' => 'wrong password',
            ];
        }
        
        if (-3 === $res) {
            return [
                'code' => '403',
                'message' => 'Too many attempts, please try again later',
            ];
        }
        
        // ...
        
        if (1 === $res) {
            return [
                'code' => '200',
                'message' => 'login success',
            ];
        }        
    }
}
```

每当需要调用到 `UserService` 下 `login()`，这即将是一段噩梦的开始。我们现在已经知道这是一场灾难。所以问题不在于检查是否正确，而是我们调用的方法返回的数据除了我们期望的值以外，还返回了 `null` `false` 或数字标识（返回码），这让调用者不得不对各种标识符进行各种检查，这完全搞乱了调用者的代码逻辑。

在 `UserRepository` 的设计上我们应该将 `getByName()` 的输出进行重新设计，以便减轻调用者的负担。

在 Robert C. Martin 写的《Clean Code - 代码整洁之道》一书中，提出了两种选择，一种是抛出异常，另一种是返回一个特殊情况的对象即特例模式（SPECIAL CASE PATTERN），其实也叫 Null Object。

抛出异常
-------

首先我们使用抛出异常的方式将上面的例子改进一下：

```php
interface UserRepository
{
    /**
    * @param string $name
    *
    * @return User
    * @throws UserNotFound
    */
    public function getByName(string $name): User;
}
```

```php
final class DbUserRepository implements UserRepository
{
    private $db;

    public function __construct(DB $db)
    {
        $this->db = $db;
    }

    public function getByName(string $name): User
    {
        $userRecords = $this->db->select('select * from users where name = :name', ['name' => $name]);

        if (0 === $userRecords->count()) {
            throw new UserNotFound();
        }

        return $userRecords->first();
    }
}
```

现在调用者不必研究被调程序返回的各种标识，可以放心的去调用了。因为之前纠结的两个元素，程序正常逻辑和错误处理现在被隔离了，我们可以查看其中任一元素，分别做相应的处理。

现在我们来看看调用者的代码，调用者对异常进行了处理：

```php
try {
    $user = $userRepository->getByName($username);
    
    if ($user->isSubscribedTo($notification)) {
        $notifier->notify($user, $notification);
    }
} catch (UserNotFound $ex) {
    $this->logger->notice('User was not found', ['username' => $username]);
}
```

但是 `try {} catch {}` 块，有点丑，它混乱了正常和错误处理两部分代码。我的建议是将 `try` 和 `catch` 块分离到两个独立的方法中以便代码更整洁。

```php
try {
   $this->notifyUserIfSubscribed($username, $notification);
} catch (\Throwable $ex) {
    $this->log($ex);
}
```

异常处理代码还可以交给最上层做处理，也就是交给应用程序的异常处理中心去集中处理。

特例模式
-------

特例模式（SPECIAL CASE PATTERN）是我们的第二种选择。在我门的示例中，它的工作方式就是创建一个拥有默认行为的类用来替换示例中的用户实体，在 `UserRepository` 下 `getByName()` 中当用户不存在时我们返回一个特例对象，这个特例对象拥有与用户实体一样的接口。现在我们可以安全地移除那些检查代码，因为我们的代码在用户存在或不存在都可以正常运行。

```php
class UnknownUser extends User
{
    public function username(): string
    {
        return 'unknow';
    }
    
    public function isSubscribedTo(Notification $notification): bool
    {
        return false;
    }
}
```

```php
final class DbUserRepository implements UserRepository
{
    private $db;

    public function __construct(DB $db)
    {
        $this->db = $db;
    }

    public function getByName(string $name): User
    {
        $userRecords = $this->db->select('select * from users where name = :name', ['name' => $name]);

        if (0 === $userRecords->count()) {
            return new UnknownUser();
        }

        return $userRecords->first();
    }
}
```

但是可能仍然需要做一些检查，当我们需要检查用户是否为不存在用户时。我们很容易检查到不存在用户：

```php
if ($user instanceof UnknowUser) {
    // do something
}
```

这里推荐使用静态工厂（简单工厂）的方式去做检查：

```php
if ($user === User::unknown()) {
    // do something
}
```

确保创建得到的是用户的一个唯一实例。我们不希望在应用的生命周期内存在多个用户特例的实例。

```php
class User
{
    public static function unknown(): User
    {
        static $unknownUser = null;
        
        if (null === $unknownUser) {
            $unknownUser = new UnknownUser();
        }
        
        return $unknownUser;
    }
}
```

我们现在仅仅可以确保在这个静态工厂方法里只有一个这种情况的特殊实例被创建，我们并不能确保没有其他人创建这个特殊实例。我们可以利用 PHP7 引入的创建匿名类功能（嵌套类）来解决这个问题。

```php
class User
{
    public static function unknown(): User
    {
        static $unknownUser = null;
        
        if (null === $unknownUser) {
            $unknownUser = new class extends User {
                public function username(): string
                {
                    return 'unknow';
                }
                
                public function isSubscribedTo(Notification $notification): bool
                {
                    return false;
                }
            }
        }
        
        return $unknownUser;
    }
}
```

> Returning null from methods is bad, but passing null into methods is worse.（在方法中返回 `null` 值是糟糕的做法，但将 `null` 值传递给方法就更糟糕了）- Robert C. Martin, "Clean Code"

Php 中，只有在代码允许的情况下才可以传递 `null` 值。

示例，订单对象包含正在订购的商品，客户及可选的折扣，`total()` 方法计算该订单合计金额。由于折扣可以不传，因此在计算合计金额的时候不得不做检查，以便正确的计算合计金额。

```php
class Order
{
    public function __construct(
        Product $product,
        Customer $customer,
        ?Discount $discount
    ) {
        $this->product = $product;
        $this->customer = $customer;
        $this->discount = $discount;
    }
    
    public function total(): float
    {
        $price = $this->product->getPrice();
        
        if (null !== $this->discount) {
            $price = $this->discount->apply($price);
        }
        
        return $price;
    }
}
```

```php
final class PremiumDiscount implements Discount
{
    public function apply(float $productPrice): float
    {
        return $productPrice * 0.5;
    }
}
```

所以，我们要如何解决这个问题，以便我们可以直接应用折扣。要想不做检查我们需要限制订单类的折扣参数设为必填，当原价购买商品的时候，我们只需要传递一个特例折扣对象，该对象应用价格不起作用。

```php
class Order
{
    public function __construct(
        Product $product,
        Customer $customer,
        Discount $discount
    ) {
        $this->product = $product;
        $this->customer = $customer;
        $this->discount = $discount;
    }
    
    public function total(): float
    {
        $price = $this->product->getPrice();
        $price = $this->discount->apply($price);
        
        return $price;
    }
}
```

```php
final class NoDiscount implements Discount
{
    public function apply(float $productPrice): float
    {
        return $productPrice;
    }
}
```

```php
$order = new Order($product, $customer, new NoDiscount());
```

现在我们的代码变得更具有可读性了。

总结
---

1. 使用异常而非返回码
2. 不要返回 `null` 值，使用异常或特例
3. 不要传递 `null` 值，使用特例
