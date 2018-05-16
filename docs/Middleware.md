# Authorization Middleware

Authorization is applied to your application as a middleware. The
`AuthorizationMiddleware` handles the following responsibilities:

* Decorating the request 'identity' with a decorator that adds the `can` and
  `applyScope` if necessary.
* Ensuring that authorization has been checked/bypassed in the request.

To use the middleware, update the `middleware` hook of your `Application` with
the following:

```php
// Import the class.
use Authorization\Middleware\AuthorizationMiddleware;

// inside your application's middleware hook.
$middlewareQueue->add(new AuthorizationMiddleware($this));
```

By passing your application instance into the middlware, it can invoke the
``authorization`` hook method on your application which should return
a configured `AuthorizationService`. A very simple example would be:

```php
namespace App;

use Authorization\AuthorizationService;
use Authorization\Policy\OrmResolver;
use Cake\Http\BaseApplication;

class Application extends BaseApplication
{
    public function authorization($request)
    {
        $resolver = new OrmResolver();

        return new AuthorizationService($resolver);
    }
}
```

The authorization service requires a policy resolver. See the 
[Policies](./Policies.md) documentation on what resolvers are available and how
to use them.

## Identity Decorator

By default the `identity` in the request will be decorated (wrapped) with
`Authorization\IdentityDecorator`. The decorator class proxies most read
operations and method calls to the wrapped identity. If you have an existing
`User` or identity class you can skip the decorator by implementing the
`Authorization\IdentityInterface` and using the `identityDecorator` middleware
option. First lets update our `User` class:

```php
namespace App\Model\Entity;

use Authorization\AuthorizationServiceInterface;
use Authorization\IdentityInterface;
use Cake\ORM\Entity;


class User extends Entity implements IdentityInterface
{

    /**
     * Authorization\IdentityInterface method
     */
    public function can($action, $resource)
    {
        return $this->authorization->can($this, $action, $resource);
    }

    /**
     * Authorization\IdentityInterface method
     */
    public function applyScope($action, $resource)
    {
        return $this->authorization->applyScope($this, $action, $resource);
    }

    /**
     * Authorization\IdentityInterface method
     */
    public function getOriginalData()
    {
        return $this;
    }

    /**
     * Setter to be used by the middleware.
     */
    public function setAuthorization($service)
    {
        $this->authorization = $service;

        return $this;
    }

    // Other methods
}
```

Now that our user implements the necessary interface, lets update our middleware
setup:

```php
// In your Application::middleware() method;

// Authorization
$middleware->add(new AuthorizationMiddleware($this, [
    'identityDecorator' => function ($auth, $user) {
        return $user->setAuthorization($auth);
    }
]));
```

You no longer have to change any existing typehints, and can start using
authorization policies anywhere you have access to your user.

### Ensuring Authorization is Applied

By default the `AuthorizationMiddleware` will ensure that each request
containing an `identity` also has authorization checked/bypassed. If
authorization is not checked an `AuthorizationRequiredException` will be raised.
This exception is raised *after* your other middleware/controller actions are
complete, so you cannot rely on it to prevent unauthorized access, however it is
a helpful aid during development/testing. You can disable this behavior via an
option:

```php
$middlewareStack->add(new AuthorizationMiddleware($this, [
    'requireAuthorizationCheck' => false
]));
```

### Handling unauthorized requests

By default authorization exceptions thrown by the application are rethrown by the middleware.
You can configure handlers for unauthorized requests and perform custom action, e.g.
redirect the user to the login page.

The built-in handlers are:

* `Exception` - this handler will rethrow the exception, this is a default behavior of the middleware.
* `Redirect` - this handler will redirect the request to the provided URL.
* `CakeRedirect` - redirect handler with support for CakePHP Router.

Both redirect handlers share the same configuration options:

* `url` - URL to redirect to (`CakeRedirect` supports CakePHP Router syntax).
* `exceptions` - a list of exception classes that should be redirected. By default only `MissingIdentityException` is redirected.
* `queryParam` - the accessed request URL will be attached to the redirect URL query parameter (`redirect` by default).
* `statusCode` - HTTP status code of a redirect, `302` by default.

For example:

```php
$middlewareStack->add(new AuthorizationMiddleware($this, [
    'unauthorizedHandler' => [
        'className' => 'Authorization.Redirect',
        'url' => '/users/login',
        'queryParam' => 'redirectUrl',
        'exceptions' => [
            MissingIdentityException::class,
            OtherException::class,
        ],
    ],
]));
```

You can also add your own handler. Handlers should implement `Authorization\Middleware\UnauthorizedHandler\HandlerInterface`,
be suffixed with `Handler` suffix and reside under your app's or plugin's 
`Middleware\UnauthorizedHandler` namespace.

Configuration options are passed to the handler's `handle()` method as the last parameter.

Handlers catch only those exceptions which extend the `Authorization\Exception\Exception` class.
