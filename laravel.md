###PHP的生命周期

比如Laravel 的public\index.php文件时，Php 为了完成这次请求，会发生5个阶段的生命周期切换：

1. 模块初始化(MINI), 即调用php.ini中指明的扩展的初始化函数进行初始化工作，如mysql扩展。
2. 请求初始化(RINT), 即初始化为执行脚本所需要的变量名称和变量值内容的符号表，如$__SESSION变量
3. 执行该PHP脚本
4. 请求处理完成，按顺序调用各个模块的RSHUTDOWN方法，对每个变量调用unset函数，如unset $_SESSION变量。
5. 关闭模块，PHP调用每个模块的MSHUTDOWN方法，这是各个模块最后一次释放内存的机会。


WEB模式和CLI（命令行）模式很相似，区别是：CLI 模式会在每次脚本执行经历完整的5个周期，因为你脚本执行完不会有下一个请求；而WEB模式为了应对并发，可能采用多线程，因此生命周期1和5有可能只执行一次，下次请求到来时重复2-4的生命周期，这样就节省了系统模块初始化所带来的开销。

![](http://oc9orpe44.bkt.clouddn.com/17-2-24/2557771-file_1487903013410_14af5.png)

知道这些有什么用？你可以优化你的Laravel代码，可以更加深入的了解Larave的singleton（单例）。至少你知道了，每一次请求结束，Php的变量都会unset，Laravel的singleton只是在某一次请求过程中的singleton；你在Laravel 中的静态变量也不能在多个请求之间共享，因为每一次请求结束都会unset。理解这些概念，是写高质量代码的第一步，也是最关键的一步。因此记住，Php是一种脚本语言，所有的变量只会在这一次请求中生效，下次请求之时已被重置，而不像Java静态变量拥有全局作用。

###Laravel的生命周期
Laravel 的生命周期从public\index.php开始，从public\index.php结束。

这么说有点草率，但事实确实如此。下面是public\index.php的全部源码（Laravel源码的注释是最好的Laravel文档），更具体来说可以分为四步：

```
1. require __DIR__.'/../bootstrap/autoload.php';

2. $app = require_once __DIR__.'/../bootstrap/app.php';
   $kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

3. $response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
   );
   $response->send();

4. $kernel->terminate($request, $response);
```
这四步详细的解释是：

1.注册加载composer自动生成的class loader，包括所有你composer require的依赖（对应代码1）.
2.生成容器Container，Application实例，并向容器注册核心组件（HttpKernel，ConsoleKernel，ExceptionHandler）（对应代码2，容器很重要，后面详细讲解）。
3.处理请求，生成并发送响应（对应代码3，毫不夸张的说，你99%的代码都运行在这个小小的handle方法里面）。
4.请求结束，进行回调（对应代码4，还记得可终止中间件吗？没错，就是在这里回调的）。

###启动Laravel基础服务#

重点是第三步处理请求，生成并发送响应。
首先Laravel框架捕获到用户发到public\index.php的请求，生成Illuminate\Http\Request实例，传递给这个小小的handle方法。在方法内部，将该$request实例绑定到第二步生成的$app容器上。让后在该请求真正处理之前，调用bootstrap方法，进行必要的加载和注册，如检测环境，加载配置，注册Facades（假象），注册服务提供者，启动服务提供者等等。这是一个启动数组，具体在Illuminate\Foundation\Http\Kernel中，包括：
```
protected $bootstrappers = [
        'Illuminate\Foundation\Bootstrap\DetectEnvironment',
        'Illuminate\Foundation\Bootstrap\LoadConfiguration',
        'Illuminate\Foundation\Bootstrap\ConfigureLogging',
        'Illuminate\Foundation\Bootstrap\HandleExceptions',
        'Illuminate\Foundation\Bootstrap\RegisterFacades',
        'Illuminate\Foundation\Bootstrap\RegisterProviders',
        'Illuminate\Foundation\Bootstrap\BootProviders',
    ];
 ```
 看类名知意，Laravel是按顺序遍历执行注册这些基础服务的，注意顺序：Facades先于ServiceProviders，Facades也是重点，后面说，这里简单提一下，注册Facades就是注册config\app.php中的aliases 数组，你使用的很多类，如Auth，Cache,DB等等都是Facades；而ServiceProviders的register方法永远先于boot方法执行，以免产生boot方法依赖某个实例而该实例还未注册的现象。

所以，你可以在ServiceProviders的register方法中使用任何Facades，在ServiceProviders的boot方法中使用任何register方法中注册的实例或者Facades，这样绝不会产生依赖某个类而未注册的现象。

###将请求传递给路由#

注意到目前为止，Laravel 还没有执行到你所写的主要代码（ServiceProviders中的除外），因为还没有将请求传递给路由。

在Laravel基础的服务启动之后，就要把请求传递给路由了。传递给路由是通过Pipeline来传递的，但是Pipeline有一堵墙，在传递给路由之前所有请求都要经过，这堵墙定义在app\Http\Kernel.php中的$middleware数组中，没错就是中间件，默认只有一个CheckForMaintenanceMode中间件，用来检测你的网站是否暂时关闭。这是一个全局中间件，所有请求都要经过，你也可以添加自己的全局中间件。

然后遍历所有注册的路由，找到最先符合的第一个路由，经过它的路由中间件，进入到控制器或者闭包函数，执行你的具体逻辑代码。
所以，在请求到达你写的代码之前，Laravel已经做了大量工作，请求也经过了千难万险，那些不符合或者恶意的的请求已被Laravel隔离在外。

![](http://oc9orpe44.bkt.clouddn.com/17-2-24/80437449-file_1487903694755_5351.png)























