# AOP (Aspect Oriented Programming)

## Concept

AOP is an abbreviation for `Aspect Oriented Programming`, a technique for achieving unified maintenance of program functions through techniques such as dynamic proxy. AOP is a continuation of OOP and an important part of Hyperf. It is a derivative paradigm of functional programming. AOP can be used to isolate the various parts of the business logic, which reduces the degree of coupling between the various parts of the business logic, improves the reusability of the program, and improves the efficiency of development. 

Popular speaking, it is in Hyperf that you can intervene in the execution of any method of any class managed by [hyperf/di](https://github.com/hyperf/di) through `Aspect`. Going in the process to change or enhance the functionality of the original method, this is AOP.

> Use AOP have to use [hyperf/di](https://github.com/hyperf/di) as the dependency injection container

## Introduction

Compared to the AOP feature implemented by other frameworks, we have further simplified the usage of this function without a more division, there is only a universal form of "Around":

- `Aspect` is a definition class that weaves into the code flow, including the definition of target to be involved, and the modification of the original method of the target.
- `ProxyClass` ，Each of the involved target classes will eventually generate a proxy class to achieve the purpose of executing the `Aspect` method, rather than passing the original class.

## Define Aspect

Each `Aspect` have to implemented `Hyperf\Di\Aop\AroundInterface`, and provided `$classes` and `$annotations` properties at `public` level. For ease of use, we can simplify the usage by inheriting `Hyperf\Di\Aop\AbstractAspect` in our aspect class.

```php
<?php
namespace App\Aspect;

use App\Service\SomeClass;
use App\Annotation\SomeAnnotation;
use Hyperf\Di\Annotation\Aspect;
use Hyperf\Di\Aop\AbstractAspect;
use Hyperf\Di\Aop\ProceedingJoinPoint;

#[Aspect]
class FooAspect extends AbstractAspect
{
    // The class to be cut in can be multiple, or can be identified by `::` to the specific method, or use * for fuzzy matching
    public $classes = [
        SomeClass::class,
        'App\Service\SomeClass::someMethod',
        'App\Service\SomeClass::*Method',
    ];
    
    // The annotations to be cut into, means the classes that use these annotations to be cut into, can only cut into class annotations and class method annotations.
    public $annotations = [
        SomeAnnotation::class,
    ];

    public function process(ProceedingJoinPoint $proceedingJoinPoint)
    {
        // After the Aspect is cut into, the corresponding method will be responsible by this method.
        // $proceedingJoinPoint is the joining point, the original method is called by the process() method of the class and obtain the result.
        // Do something before original method
        $result = $proceedingJoinPoint->process();
        // Do something after original method
        return $result;
    }
}
```

Each `Aspect` have to define `#[Aspect]` annotation or configure in `config/autoload/aspects.php` to enable.

> Use `#[Aspect]` annotatin have to `use Hyperf\Di\Annotation\Aspect;` namespace;  

## Cache of Proxy Class

All classes affected by AOP will generate the corresponding `proxy class cache` in the `./runtime/container/proxy/` folder. When the server starts, if the proxy class cache corresponding to the class exists, it will not be regenerated, using the cache directly, even the `Aspect` or `Business Class` has changed. When the cache not present, the new proxy class cache will regenerated automatically.

When deploying the production environment, we may want Hyperf to generate all proxy classes in advance, rather than dynamically generating them at runtime. All proxy classes can be generated by the `php bin/hyperf.php di:init-proxy` command. The command ignores the existing proxy class cache and regenerates it all.

Based on the above, we can combine the commands to generate the proxy class with the command and start the server, `php bin/hyperf.php di:init-proxy && php bin/hyperf.php start`, this command will automatically regenerate all the proxy class cache and then start the server.