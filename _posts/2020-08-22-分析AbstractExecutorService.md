---
layout: article
title: 浅析AbstractExecutorService
mathjax: true
---
[TOC]



# 一、简介

`AbstractExecutorService`类提供了实现`ExecutorService`接口的默认实现方法；

该类通过使用`RunnableFuture`（`newTaskFor()`方法返回）实现了`submit()`、`invokeAny()`和`invokeAll()`方法；具体来说就是：**`submit(Runnable)`方法会创建一个相关联的`RunnableFuture`对象，该对象会被执行并且返回，而子类只需要重写`newTaskFor()`方法来返回`RunnableFuture`实现即可**；

比如实现代码可以是如下：

```java
public class CustomThreadPoolExecutor extends ThreadPoolExecutor {
    static class CustomTask<V> implements RunnableFuture<V> {
        ...
    }
    
    protected <V> RunnableFuture<V> newTaskFor(Callable<V> c) {
        return new CustomTask<V>(c);
    }
    
    protected <V> RunnableFuture<V> newTaskFor(Runnable r, V v) {
        return new CustomTask<V>(r, v);
    }
}
```











# 二、`AbstractExecutorService`声明以继承结构

`AbstractExecutorService`类的声明如下：

```java
public abstract class AbstractExecutorService implements ExecutorService {
    ...
}
```

继承结构如下图所示：

![image](F:\找工作\Java基础\Java并发\AbstractExecutorService.png)

从类声明和类图可以看出，`AbstractExecutorService`是一个抽象类，实现了`ExecutorService`接口；



# 三、`AbstractExecutorService`中方法实现



## 3.1 `newTaskFor()`

该方法会根据给定的`Runnable runnable`和`T value`参数（或者`Callable<T> callable`对象参数）返回一个`RunnableFuture`对象，该对象在执行的时候会作为一个`Runnable`对象（执行任务），而且在返回的时候会作为一个`Future`对象（能得到执行结果，能取消任务执行）；

```java
protected <T> RunnableFuture<T> newTaskFor(Runnable runnbale, T value) {
    return new FutureTask<T>(runnable, value);
}

protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```

代码中的`FutureTask`实现了`RunnableFuture`接口；

## 3.2 `submit()`

该方法会根据提交的参数（`Runnable task`，`Runnable task, T result`，`Callable<T> task`）来执行相应的任务，并且返回一个对应的`Future`对象（表示异步执行的结果，通过`get()`方法能得到执行结果）；

```java
public Future<?> submit(Runnable task) {
    if (task == null)
        throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask); // 这里的execute是Executor接口中的方法
    return ftask;
}

public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) 
        throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Callable<T> task) {
    if (task == null)
        throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

从上面代码可以看出，`submit()`方法依赖`newTaskFor()`方法将输入的任务参数变成一个`RunnableFuture`对象，这样既保证了任务的可执行性又保证了任务执行结果获得是异步的；

变成`RunnableFuture`对象之后，调用`Executor`接口中的`execute()`方法执行这个任务，同时把这个任务返回；

## 3.3 `invokeAny()`

该方法会执行给定的任务集合，并且当集合中某个任务成功完成会返回改任务执行结果（还可以设置超时时间）；一旦返回（不管是完成成功返回还是异常返回），集合中未完成的任务都会被取消；而且如果在任务执行过程中修改任务集合，那么该方法的返回结果是未知的；

```java
// 无超时时间
public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
    throws InterruptedException, ExecutionException {
    try {
        return doInvokeAny(tasks, false, 0); // timeout为0
    } catch (TimeoutException cannotHappen) {
        assert false;
        return null;
    }
}

// 有超时时间的情况，tasks中的任务如果要成功执行并返回的话要在timeout之前执行
public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                       long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    return doInvokeAny(tasks, true, unit.toNanos(timeout));
}

// 执行invokeAny的核心逻辑
private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                              boolean timed, long nanos)
    throws InterruptedException, ExecutionException, TimeoutException {
    // 首先先检查tasks参数的合法性（是否为null，个数是否为0）
    if (tasks == null)
        throw new NullPointerException();
    int ntasks = tasks.size();
    if (ntasks == 0)
        throw new IllegalArgumentException();
    ArrayList<Future<T>> futures = new ArrayList<Future<T>>(ntasks); // 存储任务集合的返回Future对象
    // 通过提供的executor参数来创建一个ExecutorCompletionService对象，该对象会作为基本任务执行的单元并且提供一个LinkedBlockingQueue作为一个完成队列存放已完成的任务
    ExecutorCompletionService<T> ecs =
        new ExecutorCompletionService<T>(this);

    // For efficiency, especially in executors with limited
    // parallelism, check to see if previously submitted tasks are
    // done before submitting more of them. This interleaving
    // plus the exception mechanics account for messiness of main
    // loop.
	
    // 为了在executor并行有限的情况下提高效率，在提交其他任务之前先查看上一个任务是否已完成
    try {
        // Record exceptions so that if we fail to obtain any
        // result, we can throw the last exception we got.
        ExecutionException ee = null; // 记录下异常
        final long deadline = timed ? System.nanoTime() + nanos : 0L; // 设置超时时间
        Iterator<? extends Callable<T>> it = tasks.iterator();

        // Start one task for sure; the rest incrementally
        futures.add(ecs.submit(it.next())); // 先提交一个任务并将Future加入ArrayList中
        --ntasks;
        int active = 1; // 表示在执行中的任务数
        // 循环
        for (;;) {
            Future<T> f = ecs.poll(); // 获得完成队列中队首的Future对象，如果完成队列为空，返回null
            if (f == null) { // 完成队列为空，表示上一个任务还没执行完成
                if (ntasks > 0) { // 还有任务可以添加
                    --ntasks;
                    futures.add(ecs.submit(it.next()));
                    ++active;
                }
                else if (active == 0) // 完成队列为空且正在执行任务数为0，跳出当前循环
                    break;
                else if (timed) { // 如果任务都提交，且设置了超时时间
                    f = ecs.poll(nanos, TimeUnit.NANOSECONDS); // 在完成队列中等待nanos时间获取
                    if (f == null) // 超时
                        throw new TimeoutException();
                    nanos = deadline - System.nanoTime(); // 更新可等待时间
                }
                else
                    f = ecs.take(); // 所有任务都提交，等待完成队列出现任务执行结果
            }
            if (f != null) { // 完成队列不为空，已经得到任务返回结果，表示上个任务已经完成
                --active; // 在执行中的任务数--
                try {
                    return f.get(); // 尝试得到任务执行结果
                } catch (ExecutionException eex) {
                    ee = eex;
                } catch (RuntimeException rex) {
                    ee = new ExecutionException(rex);
                }
            }
        }

        if (ee == null)
            ee = new ExecutionException();
        throw ee;

    } finally {
        // 出现异常，成功执行后，将任务集合都标记为取消状态
        for (int i = 0, size = futures.size(); i < size; i++)
            futures.get(i).cancel(true);
    }
}
```

从上面代码中可以看出，无论是否带超时时间的`invokeAny()`方法具体的执行逻辑都是在`doInvokeAny()`方法中实现的，该方法的实现步骤是：

- 先对任务集合进行合法性判断，比如：是否为null，集合大小是否为0；
- 再创建一个元素类型为`Future<T>`的`ArrayList`用来存放提交任务后的返回结果`Future`；
- 创建一个`ExecutorCompletionService`对象来执行任务，该对象会维护一个完成队列来存放任务执行的完成结果；
- 向`ExecutorCompletionService`对象中提交一个任务，并且将任务返回结果`Future`加入`ArrayList`中；
- 接下来进入一个无限循环：
  - 首先尝试从完成队列的队首中得到一个表示任务返回结果`Future`对象；
  - 如果这个对象为null：**说明完成队列为空，表示上一个提交的任务还没完成**：
    - 判断当前任务集合中是否还有没有提交的任务，如果有的话就再提交一个任务到`ExecutorCompletionService`对象中；
    - 如果任务集合中所有任务都提交，且设置了超时时间，会尝试在完成队列中等待某段时间获取执行结果`Future`对象；
    - 如果任务集合中所有任务都提交，但没有设置超时时间，那么就会等待完成队列中有`Future`对象被添加；
  - 如果这个对象不为null：**说明完成队列不为空，表示上一个提交的任务执行完成**：
    - 从`Future`对象中得到任务执行的返回结果；
- 最后，无论是有一个任务成功执行还是有任务执行出现异常，都会将任务集合中的返回结果`Future`对象设置为取消状态；







## 3.4 `invokeAll()`

### 3.4.1 `invokeAll(Collection<? extends Callable<T>> tasks)`

该方法会执行给定任务集合，并且当所有任务执行完成时，返回一个元素为`Future`的`List`用来存放所有任务的状态和结果，这是调用`Future.isDone()`会返回`true`；

**需要注意的是，执行完成不一定代表执行成功，抛出异常也算执行完成**；

而且如果执行的过程中修改任务集合，那么该方法的返回结果为未定义；

```java
// 没有设置超时参数
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
    throws InterruptedException {
    // 校验tasks合法性
    if (tasks == null)
        throw new NullPointerException();
    // 创建一个ArrayList来存放任务集合返回结果
    ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
    boolean done = false; // 表示任务集合是否执行完成
    try {
        for (Callable<T> t : tasks) { // 遍历任务集合
            RunnableFuture<T> f = newTaskFor(t); // 将任务转化为一个RunnableFuture对象
            futures.add(f); // 将任务执行结果加入ArrayList
            execute(f); // 执行任务
        }
        // 遍历任务执行结果集合
        for (int i = 0, size = futures.size(); i < size; i++) {
            Future<T> f = futures.get(i); // 获取第i个任务对应的Future对象
            if (!f.isDone()) { // 如果第i个任务还没执行完成
                try {
                    f.get(); // 阻塞等待第i个任务完成
                } catch (CancellationException ignore) {
                } catch (ExecutionException ignore) {
                }
            }
        }
        done = true; // 所有任务执行完成
        return futures; // 返回所有任务集合对应的future集合
    } finally {
        if (!done) // 如果出现异常且任务集合没有执行完，会将每个任务对应的future对象设置为取消状态
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
    }
}

```

从上面的代码中，我们可以整理出在无超时参数下`invokeAll()`方法的执行逻辑：

- 首先先检查tasks参数的合法性；
- 接下来创建一个存放任务执行结果的`Future`对象的集合；
- 遍历任务集合（tasks参数），将每个`Callable<T>`类型任务转换为`RunnableFuture`类型，并提交任务进行执行，将任务执行结果加入集合中；
- 遍历任务执行结果集合，对于第i个任务来说，通过对应的`Future.get()`方法阻塞等待第i个任务执行完成；
- 等到所有任务执行完成（包括成功执行，抛出异常），返回`Future`对象集合；
- 如果执行任务过程中出现异常且任务集合没有全部执行完成，会将每个任务对应的`Future`对象设置为取消状态；

### 3.4.2 `invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit)`

该方法会执行给定任务集合，并且当所有任务执行完成或超时时间到达时，返回一个元素为`Future`的`List`用来存放所有任务的状态和结果，这是调用`Future.isDone()`会返回`true`；

**在超时的情况下，未完成的任务会被设置为取消状态，同样执行完成不一定代表执行成功，抛出异常也算执行完成**；

而且如果执行的过程中修改任务集合，那么该方法的返回结果为未定义；

```java
// 设置超时参数
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                     long timeout, TimeUnit unit)
    throws InterruptedException {
    // tasks参数合法性检验
    if (tasks == null)
        throw new NullPointerException();
    long nanos = unit.toNanos(timeout); // 表示可等待时间
    ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size()); // 创建一个Future对象集合
    boolean done = false; // 任务集合是否执行完成
    try {
        for (Callable<T> t : tasks) // 遍历任务集合并且将每个任务对应的Future对象加入集合中
            futures.add(newTaskFor(t));

        final long deadline = System.nanoTime() + nanos;
        final int size = futures.size();

        // Interleave time checks and calls to execute in case
        // executor doesn't have any/much parallelism.
        for (int i = 0; i < size; i++) {
            execute((Runnable)futures.get(i)); // 执行第i个任务
            nanos = deadline - System.nanoTime(); 
            if (nanos <= 0L)
                return futures; // 如果执行超时，直接返回
        }

        for (int i = 0; i < size; i++) {
            Future<T> f = futures.get(i); // 得到第i个任务的执行结果
            if (!f.isDone()) {
                if (nanos <= 0L)
                    return futures; // 超时，直接返回
                try {
                    f.get(nanos, TimeUnit.NANOSECONDS); // 不超时，等待一段时间
                } catch (CancellationException ignore) {
                } catch (ExecutionException ignore) {
                } catch (TimeoutException toe) {
                    return futures;
                }
                nanos = deadline - System.nanoTime(); // 更新可等待时间
            }
        }
        done = true;
        return futures;
    } finally {
        if (!done)
            // 如果出现异常（或超时）且任务集合没有执行完，会将每个任务对应的future对象设置为取消状态
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
    }
}
```

从上面的代码中，我们可以整理出在有超时参数下`invokeAll()`方法的执行逻辑：

- 首先先检查tasks参数的合法性；
- 接下来创建一个存放任务执行结果的`Future`对象的集合，以及一个可等待时间；
- 遍历任务集合（tasks参数），将每个`Callable<T>`类型任务转换为`RunnableFuture`类型，将任务执行结果加入集合中；
- 再次遍历任务集合，对于第i任务来说，将第i任务提交执行并更新可等待时间，如果可等待时间小于0直接返回任务执行结果集合；
- 遍历任务执行结果集合，对于第i个任务执行结果来说，先判断任务是否执行完成，如果还没执行完成，则阻塞等待（不超过可等待时间）任务执行结果；
- 等到所有任务执行完成（包括成功执行，抛出异常），返回`Future`对象集合；
- 如果执行任务过程中出现异常，超时且任务集合没有全部执行完成，会将每个任务对应的`Future`对象设置为取消状态；







