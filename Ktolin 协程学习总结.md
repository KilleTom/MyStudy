[TOC]



# Ktolin 协程学习总结

## 协程概念与目的

1. 概念

   协程是一种特殊函数，协程对比于线程更加轻量化，存活于线程，线程与协程关系为：1：N，程序自主决定协程的状态

2. 目的

   解决线程间切换的三大常见问题：同步锁、线程阻塞与线程运行状态间切换、线程上下文切换状态。

   感知用户当前操作状态，结合业务需求，运用逻辑自主操作相应操作譬如协程的挂起、切换、取消、运行等
   
## 协程基本概念

在学习协程基本使用前，通过一些简单的概念学习，掌握一些基本后，便于更快的上手协程的使用。

### CoroutineContext

协程上下文，它包含了一个默认的协程调度器。所有协程都必须在` CoroutineContext` 中执行。 

### CoroutineScope

协程的作用域，它的实现类似于的一个生命周期的实体，该作用域内的协程们，在这个实体内，不同的时机做不同的事情，互相协作把事情做完。

`CoroutineScope`是一个抽象接口，有且只有一个`CoroutineContext`属性，目的利用这个属性将这个作用域内的协程进行上下文关联。

在`kotlin`内有一个全局`GlobalScope`，目的在于操作顶级协程函数，将这些顶级的协程函数始终伴随这程序的生命周期内执行相应的运行。

### CoroutineDispatcher

类似于线程调度器，目的在于决定哪些协程存活于哪些线程中运行。在`Kotlin`的协程包含多个协程调度器。

常见实现的`Dispatcher`

1. **Dispatchers.Main**

   在当前的`Android`主线程上运行，也就是可用于更新`UI`。

2. **Dispatchers.IO**

   运行去主线程之外的`IO`线程，例如：常见的网络、数据库等IO操作

3. **Dispatchers.Default**

   提供的默认线程，基于主线程之外执行，目的在于运行`运算CPU密集计算的操做`。

4. **Dispatchers.Unconfined**

   直接在当前线程运行。

### suspend

试想一下，当程序运行到某一时刻时候，某个需求需要暂且且不能阻塞线程的操作，当到达合适的时候再次运行。也就是挂起操作。

`suspend`作用就是将这些特殊需求的函数标记起来。用来表达这些函数是可以被挂起。注意被`suspend`标记的函数只能运行于协程内，或者其他被`suspend`标记的函数内。

其标记范围为:普通函数、拓展函数、lambda表达式。

### suspension point

协程中每个被挂起的函数，都可以作为一个`suspension point`，也就是挂点。

### Continuation

在协程的作用域内，多个协程或者单个协程，在某个时刻，挂起然后执行 再次挂起、执行，交替将挂起执行 反复进行这样的操作。

相邻的两个挂点可以看作是`Continuation`，一个完整的协程运行存在多个`Continuation`。

### Job

将某个业务需求执行过程看做一个对象。这个对象交给`CoroutineDispatcher`调度处理。并且具有生命周期，允许存在`上下流关联关系`，当上游被中断时下游也会随之中断，这里中断代指取消。

具有三种生命周期：

- `isActive`：是否处于执行中

- `isCompleted`:是否完成

- `isCancelled`:是否取消也就是中断运行


### Deferred
基于`Job`的实现，目的将协程运行的结果返回。

### Coroutine builders

`Coroutine builder`代表一种这样的函数：“通过可接受`suppend`类型的`lambda`表达式，并创建一个协程序来运行运行协程”

`Kotlin` 提供了多种 `Coroutine builders`，通过这些`builders` 满足常见的业务场景。


## 协程的上手开撸之路

### 基本使用

结合上述一些概念，这节内容将会分享如何简单有效的使用协程，并掌握一些基本使用。

#### 简单创建协程的方式以及运用

- `launch`方式

  `launch`目的在于创建一种不需要产生结果回调的协程。使用方式如下

  ```kotlin
   fun lunch(){
  
          GlobalScope.launch {
  
              delay(1000)
  
              logI("launch")
          }
  
      }
  ```

  相应的执行结果：

  ```kotlin
  launch
  ```

  

- `async`方式

  `async`可以理解为创建一种具有返回值的协程，注意运用它创建的对象为`Deferred<T>`，示例方式如下:

  ```kotlin
      fun async(){
  
       val deferred =   GlobalScope.async {
  
             return@async "just async"
         }
  
          GlobalScope.launch {
              logI(deferred.await())
          }
  
      }
  ```

  相应的结果如下:

  ```kotlin
  just async
  ```

  

- `runBlocking`方式

  `runBlocking`目的在于基于当前运行的线程，创建出一个阻塞线程的协程。

  需要注意的是：`runBlocking`其内部可以运用`launch`函数创建一个`Job`，但是在`launch`内部则不能利用`runBlocking`否则会报错。示例方式如下：

  ```kotlin
      fun runBlck(){
          
         runBlocking {
             
             launch {
                 delay(1000)
                 println("Hello World!")
             }
  
             delay(2000)
         }
      }
  ```

  相应结果如下：

  ```kotlin
  Hello World!
  ```

  

- `invokeOnCompletion`关注协程的运行状态

  有没有这样一种方式：能够感知到协程是否运行完成，或者协程运行过程种发生的异常呢？有。`invokeOnCompletion`专门处理在协程运行过程中是否能够正常走完。示例如下

  ```kotlin
    fun completion(){
        
          GlobalScope.launch {
  
              val  result = async{
  
                  delay(2000)
                  1
              }
  
              result.invokeOnCompletion {
  
                  if (it!=null){
  
                      println("exception: ${it.message}")
                  } else {
  
                      println("result is complete")
                  }
              }
  
              result.cancelAndJoin()
  
              println(result.await())
          }
  
          Thread.sleep(5000)
      }
  ```

  对应结果如下:

  ```kotlin
  exception: DeferredCoroutine was cancelled
  ```

  其实`invokeOnComplection`基于`CompletionHandler`回调实现，你可以这样理解`invokeOnComplection`负责将`CompletionHandler`当作一个配置项设置给当前协程，当当前协程运行完成后，不管是否异常都会调用到`CompletionHandler`这个`Handler`发送`job was done`这样的一个信息，至于这件事情是否正常做完还是出现了啥问题，就在对于`Throwable`作为一个结果返回来做判断

  

- 关于协程`start`的探讨

#### 多种挂起协程的方式

#### 协程调度攻略

#### 协程的上下文攻略

#### 合理运用协程作用域

#### 协程间的通讯机制

### 协程在Android上的运用

利用协程结合一些相关`Android`常见需求，进行讲解协程在`Android`如何运用

#### 运用协程简单实现防重复单击


#### 协程与Retrofit的配合

#### 协程结合JetPack中MVVM的使用

## 协程总结

