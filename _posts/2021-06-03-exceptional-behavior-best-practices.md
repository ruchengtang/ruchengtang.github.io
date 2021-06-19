---
title: "PHP 处理异常行为的最佳实践"
subtitle: "从组织结构开始，以免日后进行昂贵的重构"
layout: post
author: "Rucheng Tang"
header-style: text
hidden: false
tags:
  - PHP
  - PHP 异常
  - 最佳实践
---

为什么要使用异常？ [《PHP 错误处理》]({% post_url 2021-05-31-handling-exceptional-conditions-in-php %}) 这篇文章详细地介绍了在什么情况下应该使用异常，什么情况下使用特例模式替代异常。很多人也许会觉得使用异常很简单，直接 `throw new` 一个异常即可，事实并非如此，这就是这篇文章要讨论的话题，如何组织异常，避免我们混乱使用异常，从一个噩梦（检查泛滥）到另一个噩梦（异常混乱）。

我们先来看一段简单的代码：

```php
$user = $this->usersGateway->fetchOneById($userId);

if (!$user) {
    throw new \Exception('User with the ID: ' . $userId . ' does not exist');
}
```

使用异常没有这么简单。


自定义异常类型
------------

上面这个例子的主要问题是抛出的异常类型没有丰富的语义，我们必须阅读整个消息才能理解该异常的目的。另外，抛出的异常类型是基础异常，`\Exception` 是所有异常的基类，调用者很难根据异常类型去做相应的处理。解决方案是为每个出现异常情况的整体逻辑定义自定义异常类型，其**名称一定要语义化**：

```php
class UserNotFountException extends RuntimeException
{  
}
```

```php
// ...

throw new UserNotFountException('User with the ID: ' . $userId . ' does not exist');
```

一定要在 **[SPL 异常类](https://www.php.net/manual/en/spl.exceptions.php#spl.exceptions.tree) 基础之上创建自定义异常类型**，不要再使用 `\Exception`。因为这可以为我们的异常类型提供更加丰富的语义，捕获异常时将变得更加灵活。

更好了？这是肯定的，但是离完美还很远。


格式化异常消息
------------

在上面的例子中，我们在抛出异常的地方进行了异常消息的格式化处理，这导致代码有点杂乱且分散注意力，其实这是没有上限的，随着异常消息的丰富，我们的代码将变得更加的丑陋。我们可以通过将异常消息的格式逻辑移动到自定义异常类中来消除代码的复杂程度。最好的方法是使用静态工厂：

```php
class UserNotFoundException extends RuntimeException
{
    public static function forUserId(string $userId) : self
    {
        return new self(sprintf(
            'User with the ID: %s does not exist',
            $userId
        ));
    }
}
```

抛出异常时，我们只需调用添加的静态工厂方法即可：

```php
throw UserNotFoundException::forUserId($userId);
```

如果想更详细地了解格式异常消息，可以阅读 Ross Tuck 写的 [《格式化异常消息》](https://www.rosstuck.com/formatting-exception-messages) 这篇文章。


内聚异常
-------

与其它类型的类一样，在编写异常类时，必须遵守单一职责原则（Single responsibility principle）。下面是一个设计不佳的异常类的例子：

```php
class UserException extends \Exception
{
    public static function forEmptyEmail() : self
    {
        return new self("User's email must not be empty");
    }

    public static function forInvalidEmail(string $email) : self
    {
        return new self(sprintf(
            '%s is not a valid email address',
            $email
        ));
    }

    public static function forNonexistentUser(string $userId) : self
    {
        return new self(sprintf(
            'User with the ID: %s does not exist',
            $userId
        ));
    }
}
```

在这个例子中，我们可以很清楚地注意到这个异常类中有两个不同的职责，一个负责用户验证，另一个则是用于请求不存在的用户，因此我们应该拆分为两个：

```php
class InvalidUserException extends DomainException
{
    public static function forEmptyEmail() : self
    {
        return new self("User's email address must not be empty");
    }

    public static function forInvalidEmail(string $email) : self
    {
        return new self(sprintf(
            '%s is not a valid email address',
            $email
        ));
    }
}
```

```php
class UserNotFoundException extends RuntimeException
{
    public static function forUserId(string $userId) : self
    {
        return new self(sprintf(
            'User with the ID: %s does not exist',
            $userId
        ));
    }
}
```

现在变得更有意义了，比如我们想尝试给这两种异常应用不同的 HTTP 状态码（分别为 400 和 404）。


提供上下文
--------

为我们的异常提供丰富的上下文信息，我们的调用者也许会对这些信息感兴趣，您肯定不希望调用者通过解析异常消息文本来获取感兴趣的信息吧。上面这个例子，我们可以为异常 `UserNotFoundException` 提供用户 ID 信息：

```php
class UserNotFoundException extends RuntimeException          
{  
    private int $userId;
    
    public static function forUserId(string $userId) : self   
    {                                                         
        $ex = new self(sprintf(                              
            'User with the ID: %s does not exist',            
            $userId                                           
        ));    
        
        $ex->userId = $userId;
        
        return $ex;
    } 
    
    public function userId() : int
    {
        return $this->userId;
    }
}                                                             
```

这里我门将 `$userId` 属性设为私有，通过 `userId()`来单独获取，因为我们不希望 `$userId` 值可以被随意的修改。


异常包装
-------

当我们在调用一些第三方组件的时候，我们希望隐藏调用组件抛出的异常，抛出我们包装过的异常，例如，我们正在使用某些组件构建 API SDK，我们不希望调用组件的细节直接暴露给我们的调用者，我们想要隐藏我们使用的组件所抛出的异常，将该异常包装为更有意义的异常类型。包装异常最重要的是保留真正的错误，以便异常发生时可以轻松地追踪到真正的错误信息。

```php
try {
    return $this->toResult(
        $this->httpClient->request('GET', '/users')
    );
} catch (ConnectException $ex) {
    throw ApiNotAvailable::reason($ex);
}  
```

```php
final class ApiNotAvailable extends \Exception implements ExceptionInterface
{
    public static function reason(ConnectException $error) : self
    {
        return new self(
            'API is not available',
            0,
            $error // preserve previous error
        );
    }
}                                                          
```


异常码
-----

用一个码（code）来唯一标识发生的错误是很有用的，这在 API 类型的应用程序中通常是必需的。PHP 异常类已经支持 code（`\Exception` 构造函数第二个参数），所以在我们自定义的异常类中，别忘记使用它：

```php
class UserNotFoundException extends RuntimeException         
{                                                            
    private int $userId;                                     
                                                             
    public static function forUserId(string $userId) : self  
    {                                                        
        $ex = new self(sprintf(                              
                'User with the ID: %s does not exist',           
                $userId                                          
            ),
            ErrorCodes::ERROR_USER_NOT_FOUND
        );                                                  
                                                             
        $ex->userId = $userId;                               
                                                             
        return $ex;                                          
    }                                                        
                                                             
    public function userId() : int                           
    {                                                        
        return $this->userId;                                
    }                                                        
}
```

推荐将异常码放到单个类中集中进行管理，我们可以使用这些异常码来映射 HTTP 状态码。处理 HTTP 状态码，还有一种方式，您可以先思考下？本文后面的内容会介绍到。


组件级异常
---------

当我们创建一个组件时，拥有一个组件级别的异常类型是非常有必要的，这有助于调用我们组件的用户可以轻易的捕获到我们组件的任何一个异常。我们可以通过 [Marker Interface](http://en.wikipedia.org/wiki/Marker_interface_pattern) 来实现：

```php
namespace App\Domain\Exception;

interface ExceptionInterface
{
}
```

```php
class UserNotFoundException extends RuntimeException implements ExceptionInterface       
{                                                            
    private int $userId;                                     
                                                             
    public static function forUserId(string $userId) : self  
    {                                                        
        $ex = new self(sprintf(                              
                'User with the ID: %s does not exist',           
                $userId                                          
            ),
            ErrorCodes::ERROR_USER_NOT_FOUND
        );                                                  
                                                             
        $ex->userId = $userId;                               
                                                             
        return $ex;                                          
    }                                                        
                                                             
    public function userId() : int                           
    {                                                        
        return $this->userId;                                
    }                                                        
}
```

我们仅仅需要在每个异常命名空间下创建一个 `ExceptionInterface` 接口，然后让自定义异常类去继承。如 Controllers、Repositories 等等。这为我们的调用者捕获异常提供了丰富地层级。


错误处理
-------

到目前为止，本文都是在讲关于编写和组织异常类。如何以及在何处捕获异常，这个话题同样也很重要。

MVC 应用程序中通常都是在 `controllers/actions` 中捕获异常：

```php
class UserController extends BaseController
{
    public function viewUserAction(RequestInterface $request)
    {
        try {
            $user = $this->userService->get($request->get('id'));

            return new JsonResponse($user->toArray()); 
        } catch (\Exception $ex) {
            return new JsonResponse([
                'error' => $ex->getCode(),
                'message' => $ex->getMessage(),
            ], 500);
        }
    }
}
```

这种解决方案的问题在于，将异常转换为合适的错误响应的逻辑可能会变得越来越复杂，导致维护困难，并且我们要为每一个 action 编写这样类似的代码，导致代码重复，此外，我们也不想暴露从应用程序较低层引发的异常消息，如数据库连接错误，所以我们需要一些过滤逻辑。

其实我们需要一个异常处理中心来进行集中处理异常。我们可以通过覆盖 PHP 的默认错误处理程序构建您自己的：

```php
set_error_handler(function ($errno, $errstr, $errfile, $errline) {
    if (! (error_reporting() & $errno)) {
        return;
    }

    throw new ErrorException($errstr, 0, $errno, $errfile, $errline);
});

set_exception_handler(function ($exception) {
    //log exception
    //display exception info
});
```

或使用一些现有的解决方案，如 [whoops](https://github.com/filp/whoops) 或 [BooBoo](https://github.com/thephpleague/booboo)。

这里以 whoops 为例，我们看下如何使用，我们首先创建了一个工厂，将错误处理工厂注册到容器中，通过容器获取到 whoops 对象，并执行 `register()` 方法：

```php
final class ErrorHandleFactory
{
	public function __invoke(ContainerInterface $container)
	{
		$whoops = new \Whoops\Run();

		if (\Whoops\Util\Misc::isAjaxRequest()) {
    		$jsonHandler = new JsonResponseHandler();
   			$jsonHandler->setJsonApi(true);
    		$whoops->pushHandler($jsonHandler);
		} elseif (\Whoops\Util\Misc::isCommandLine()) {
			$whoops->pushHandler(new CommandLineHandler());
		} else {
			$whoops->pushHandler(new PrettyPageHandler());
		}

		return whoops;
	}
}
```

```php
// src/bootstrap.php
// ... initialize DI container

$container->get(\Whoops\Run::class)->register();

return $container;
```

```php
// public/index.php
$container = require __DIR__ . '/../src/bootstrap.php';
$container->get('App\Web')->run();
```

```php
// bin/app
$container = require __DIR__ . '/../src/bootstrap.php';
$container->get('App\Console')->run();
```

很简单吧。继续。

出于某种原因，我们不想将所有的异常都记录到错误日志中，例如，我们不会对记录用户错误的客户端错误感兴趣，如找不到用户。我们希望将注意力集中在服务器错误上，这些错误可能导致我们的程序系统出错。所以我们该怎么处理呢？我们可以用一个很大的异常列表来记录这些不希望记录到错误日志的异常。很显然，这肯定不是一个可维护的解决方案。所以我们该怎么做呢？我们可以再次使用 `Marker Interface` 方式来解决问题：

```php
interface DontLog
{
}
```

在所有不希望记录日志的异常中 `implements DontLog`：

```php
class UserNotFoundException extends RuntimeException implements ExceptionInterface, DontLog
{
    // ...
}
```

接下来，如何处理这部分的异常呢？我们创建一个日志处理类，在这里处理日志记录的逻辑：

```php
final class LogHandler extends Handler
{
    // ...
    
	public function handle()
	{
		$error = $this->getException();

		if ($error instanceof DontLog) {
			return self::DONE;
		}

		$this->logger->error($error->getMessage(), [
			'exception' => $error,
		]);

		return self::DONE;
	}
}
```

将日志处理 `LogHandler` 注册到 whoops 上：

```php
final class ErrorHandleFactory
{
	public function __invoke(ContainerInterface $container)
	{
		$whoops = new \Whoops\Run();

		if (\Whoops\Util\Misc::isAjaxRequest()) {
    		$jsonHandler = new JsonResponseHandler();
   			$jsonHandler->setJsonApi(true);
    		$whoops->pushHandler($jsonHandler);
		} elseif (\Whoops\Util\Misc::isCommandLine()) {
			$whoops->pushHandler(new CommandLineHandler());
		} else {
			$whoops->pushHandler(new PrettyPageHandler());
		}
		
		$whoops->pushHandler(new LogHandler($container->get('Logger')));

		return whoops;
	}
}
```

假如我们正在构建一个 Api，希望可以返回正确的 HTTP 状态码，而非全部都用一个 HTTP 状态码，我们怎么知道哪个异常映射到哪个 HTTP 状态码，我们可以用一个大的映射列表存储着异常与 HTTP 状态码的对应关系，但同样是不可维护的，我们只想要一个灵活的解决方案，所以我们再次使用 `Marker Interface` 方式：

```php
interface ProvidesHttpStatusCode
{
	public function getHttpStatusCode() : int;
}
```

```php
class UserNotFoundException extends RuntimeException implements
    ExceptionInterface, 
    DontLog, 
    ProvidesHttpStatusCode
{
    public function getHttpStatusCode() : int
    {
        return 404;
    }
}
```

```php
final class SetHttpStatusCodeHandler extends Handler
{
	public function handle()
	{
		$error = $this->getException();

		$httpStatusCode = ($error instanceof ProvidesHttpStatusCode)
			? $error->getHttpStatusCode()
			: 500;
		}

		$this->getRun()->sendHttpCode($httpStatusCode);

		return self::DONE;
	}
}
```

```php
final class ErrorHandleFactory
{
	public function __invoke(ContainerInterface $container)
	{
		$whoops = new \Whoops\Run();

		if (\Whoops\Util\Misc::isAjaxRequest()) {
    		$jsonHandler = new JsonResponseHandler();
   			$jsonHandler->setJsonApi(true);
    		$whoops->pushHandler($jsonHandler);
		} elseif (\Whoops\Util\Misc::isCommandLine()) {
			$whoops->pushHandler(new CommandLineHandler());
		} else {
			$whoops->pushHandler(new PrettyPageHandler());
		}
		
		$whoops->pushHandler(new LogHandler($container->get('Logger')));
		$whoops->pushHandler(new SetHttpStatusCodeHandler());

		return whoops;
	}
}
```

怎么样？使用错误处理中心去管理维护我们的异常很灵活吧。


测试异常行为
----------

对程序进行负面测试（Negative testing），可以确保应用程序能够处理不当的用户行为。通常我们是怎样测试异常的呢？PHPUNIT 可以通过 `@expectException` 注释或 `expectException()` 方法来对异常进行测试，示例：

```php
class CNYTest extends TestCase
{
    /**
     * @dataProvider emptyCnyProvider
     */
    public function testConvertOrFailWithEmptyCny($value)
    {
        $cny = new CNY();

        $this->expectException(InvalidArgumentException::class);
 
        $cny->convertOrFail($value);
    }
```

PHPUnit 还介绍了 [另一种对异常进行测试的方法](https://phpunit.de/manual/4.5/zh_cn/writing-tests-for-phpunit.html#writing-tests-for-phpunit.exceptions.examples.ExceptionTest4.php) 示例：

```php
class CNYTest extends TestCase
{
    /**
     * @dataProvider emptyCnyProvider
     */
    public function testConvertOrFailWithEmptyCny($value)
    {
        $cny = new CNY();

        try {
            $cny->convertOrFail($value);

            $this->fail('Exception should have been raised!');
        } catch (InvalidArgumentException $e) {
            $this->assertSame(ErrorCode::INVALID_ARGUMENT_EMPTY_CNY, $e->getCode());
            $this->assertSame('Cny must not be empty!', $e->getMessage());
            $this->assertSame((string)$value, $e->originCny());
            $this->assertSame(false, is_null($e->cny()));
        }
    }
```

推荐使用第二种方式来测试异常，因为它符合 ARRANGE-ACT-ASSERT 测试原则并且这种写法对异常进行断言更加灵活，其次代码更直观。


参考文章
-------

1. [Best practices for handling exceptional behavior](https://www.nikolaposa.in.rs/blog/2016/08/17/exceptional-behavior-best-practices/)




