> laravel 源码分析具体注释见 [https://github.com/FX-Max/source-analysis-laravel](https://github.com/FX-Max/source-analysis-laravel)



# 前言

队列 (Queue) 是 laravel 中比较常用的一个功能，队列的目的是将耗时的任务延时处理，比如发送邮件，从而大幅度缩短 Web 请求和响应的时间。本文我们就来分析下队列创建和执行的源码。



# 队列任务的创建



先通过命令创建一个 Job 类，成功之后会创建如下文件 laravel-src/laravel/app/Jobs/DemoJob.php。

```bash
> php artisan make:job DemoJob

> Job created successfully.
```



下面我们来分析一下 Job 类的具体生成过程。

执行 `php artisan make:job DemoJob` 后，会触发调用如下方法。

laravel-src/laravel/vendor/laravel/framework/src/Illuminate/Foundation/Providers/ArtisanServiceProvider.php

```php
/**
 * Register the command.
 * [A] make:job 时触发的方法
 * @return void
 */
protected function registerJobMakeCommand()
{
    $this->app->singleton('command.job.make', function ($app) {
        return new JobMakeCommand($app['files']);
    });
}
```



接着我们来看下 JobMakeCommand 这个类，这个类里面没有过多的处理逻辑，处理方法在其父类中。

```php
class JobMakeCommand extends GeneratorCommand
```



我们直接看父类中的处理方法，GeneratorCommand->handle()，以下是该方法中的主要方法。

```php
public function handle()
{
    // 获取类名
    $name = $this->qualifyClass($this->getNameInput());
    // 获取文件路径
    $path = $this->getPath($name);
    // 创建目录和文件
    $this->makeDirectory($path);
    // buildClass() 通过模板获取新类文件的内容
    $this->files->put($path, $this->buildClass($name));
    // $this->type 在子类中定义好了，例如 JobMakeCommand 中 type = 'Job'
    $this->info($this->type.' created successfully.');
}
```



方法就是通过目录和文件，创建对应的类文件，至于新文件的内容，都是基于已经设置好的模板来创建的，具体的内容在 buildClass($name) 方法中。

```php
protected function buildClass($name)
{
    // 得到类文件模板，getStub() 在子类中有实现，具体看 JobMakeCommand 
    $stub = $this->files->get($this->getStub());
    // 用实际的name来替换模板中的内容，都是关键词替换
    return $this->replaceNamespace($stub, $name)->replaceClass($stub, $name);
}
```



获取模板文件

```php
protected function getStub()
{
    return $this->option('sync')
                    ? __DIR__.'/stubs/job.stub'
                    : __DIR__.'/stubs/job-queued.stub';
}
```



job.stub

```php
<?php
/**
* job 类的生成模板
*/
namespace DummyNamespace;

use Illuminate\Bus\Queueable;
use Illuminate\Foundation\Bus\Dispatchable;

class DummyClass
{
    use Dispatchable, Queueable;

    /**
     * Create a new job instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        //
    }
}
```



job-queued.stub

```php
<?php
/**
* job 类的生成模板
*/
namespace DummyNamespace;

use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;

class DummyClass implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * Create a new job instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        //
    }
}
```



下面看一下前面我们创建的一个Job类，DemoJob.php，就是来源于模板 job-queued.stub。

```php
<?php
/**
* job 类的生成模板
*/
namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;

class DemoJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * Create a new job instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        //
    }
}
```



至此，我们已经大致明白了队列任务类是如何创建的了。下面我们来分析下其是如何生效运行的。