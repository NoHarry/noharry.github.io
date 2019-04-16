---
title: OkHttp 流程浅析
# toc: true
tags:
- OkHttp
- Android
date: 2019/4/15 20:30:00
update: 2019/4/15 21:01:00
---

## 简介
本文通过结合[OkHttp源码](https://github.com/square/okhttp/tree/okhttp_3.12.x),分析发送请求的大致流程。

* 本文源码基于[3.12.0](https://github.com/square/okhttp/tree/okhttp_3.12.x)版本

### 示例

首先我们创建一个最简单的请求，以此为例开始进行分析

```java
    //创建OkHttpClient
    OkHttpClient client=new OkHttpClient.Builder().build();
    //创建Request
    Request request=new Request
        .Builder()
        .url(url)
        .build();
    //发送一个异步请求
    client.newCall(request).enqueue(new Callback() {
      @Override
      public void onFailure(Call call, IOException e) {

      }

      @Override
      public void onResponse(Call call, Response response) throws IOException {

      }
    });

    //发送一个同步请求
    try {
      Response response = client.newCall(request).execute();
    } catch (IOException e) {
      e.printStackTrace();
    }
```

## 流程
### 1.1 创建请求
```java
public final class Request {
  final HttpUrl url;
  final String method;
  final Headers headers;
  final @Nullable RequestBody body;
  final Map<Class<?>, Object> tags;

  private volatile @Nullable CacheControl cacheControl;

  Request(Builder builder) {
    this.url = builder.url;
    this.method = builder.method;
    this.headers = builder.headers.build();
    this.body = builder.body;
    this.tags = Util.immutableMap(builder.tags);
  }

  ...

  public static class Builder {
    @Nullable HttpUrl url;
    String method;
    Headers.Builder headers;
    @Nullable RequestBody body;

    Map<Class<?>, Object> tags = Collections.emptyMap();

    ....

  }
}
```
首先使用建造者模式构建一个Requst,来插入请求的数据。
### 1.2 封装请求
请求封装在了接口Call的实现类RealCall中：
```java
final class RealCall implements Call {

  ...

  private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    //构建的OkHttpClient
    this.client = client;
    //用户构建的Request
    this.originalRequest = originalRequest;
    //是不是WebSocket请求
    this.forWebSocket = forWebSocket;
    //构建RetryAndFollowUpInterceptor拦截器
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
    //Okio中提供的用于超时机制的方法
    this.timeout = new AsyncTimeout() {
      @Override protected void timedOut() {
        cancel();
      }
    };
    this.timeout.timeout(client.callTimeoutMillis(), MILLISECONDS);
  }

  ...

}
```
### 1.3 执行请求
请求分为**同步请求**和**异步请求**:

* 同步请求:
```java
  @Override
  public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    timeout.enter();
    eventListener.callStart(this);
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      e = timeoutExit(e);
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

* 异步请求:
```java
  @Override
  public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

从上面可以看到不论是同步请求还是异步请求都是在Dispatcher中进行处理，

区别在于：
* 同步请求：直接执行executed，并返回结果
* 异步请求：构造一个AsyncCall，并将其加入到readyAsyncCalls这个准备队列中

```java
final class AsyncCall extends NamedRunnable {
   private final Callback responseCallback;

   AsyncCall(Callback responseCallback) {
     super("OkHttp %s", redactedUrl());
     this.responseCallback = responseCallback;
   }

   String host() {
     return originalRequest.url().host();
   }

   Request request() {
     return originalRequest;
   }

   RealCall get() {
     return RealCall.this;
   }

   /**
    * Attempt to enqueue this async call on {@code executorService}. This will attempt to clean up
    * if the executor has been shut down by reporting the call as failed.
    */
   void executeOn(ExecutorService executorService) {
     assert (!Thread.holdsLock(client.dispatcher()));
     boolean success = false;
     try {
       executorService.execute(this);
       success = true;
     } catch (RejectedExecutionException e) {
       InterruptedIOException ioException = new InterruptedIOException("executor rejected");
       ioException.initCause(e);
       eventListener.callFailed(RealCall.this, ioException);
       responseCallback.onFailure(RealCall.this, ioException);
     } finally {
       if (!success) {
         client.dispatcher().finished(this); // This call is no longer running!
       }
     }
   }

   @Override protected void execute() {
     boolean signalledCallback = false;
     timeout.enter();
     try {
       Response response = getResponseWithInterceptorChain();
       if (retryAndFollowUpInterceptor.isCanceled()) {
         signalledCallback = true;
         responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
       } else {
         signalledCallback = true;
         responseCallback.onResponse(RealCall.this, response);
       }
     } catch (IOException e) {
       e = timeoutExit(e);
       if (signalledCallback) {
         // Do not signal the callback twice!
         Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
       } else {
         eventListener.callFailed(RealCall.this, e);
         responseCallback.onFailure(RealCall.this, e);
       }
     } finally {
       client.dispatcher().finished(this);
     }
   }
 }
```
AsyncCall继承自NamedRunnable，而NamedRunnable可以看成一个会给其所运行的线程设定名字的Runnable，Dispatcher会通过ExecutorService来执行这些Runnable。

### 1.4 请求的调度

```java
public final class Dispatcher {
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;
  private @Nullable Runnable idleCallback;

  /** Executes calls. Created lazily. */
  private @Nullable ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  void enqueue(AsyncCall call) {
      synchronized (this) {
        readyAsyncCalls.add(call);
      }
      promoteAndExecute();
    }

  synchronized void executed(RealCall call) {
      runningSyncCalls.add(call);
    }

  private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (runningCallsForHost(asyncCall) >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }

    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }


}
```
请求的调度主要在Dispatcher类中进行，其中维护了3个双端队列：

* readyAsyncCalls:准备队列用于添加准备执行的异步请求。
* runningAsyncCalls：正在执行的异步请求队列。
* runningSyncCalls：正在执行的同步请求队列。

对于同步请求，Dispatcher会直接将请求加入到同步请求队列执行；对于异步请求首先会将请求加入readyAsyncCalls中，接下来会遍历readyAsyncCalls判断如果当前执行的异步请求数量小于65并且同一host下的异步请求数小于5，则将readyAsyncCalls中的请求加入到runningAsyncCalls开始执行并从readyAsyncCalls中移除。

### 1.5 请求的执行

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    //用户自定义拦截器
    interceptors.addAll(client.interceptors());
    //用于处理请求失败时重试和重定向
    interceptors.add(retryAndFollowUpInterceptor);
    //给发送的添加请求头等信息，同时处理返回的响应使之转换成对用户友好的响应
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    //处理缓存相关逻辑
    interceptors.add(new CacheInterceptor(client.internalCache()));
    //处理建立连接相关
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    //从服务器获取响应数据
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

![okhttp](okhttp.png)

可以说okhttp最核心的部分就是拦截器的这部分，这里采用责任链的设计模式，使各个功能充分解耦，各司其职，请求从用户自定义的拦截器开始层层传递到CallServerInterceptor，每层做出相应的处理，直到请求发出，与此同时，返回的响应从CallServerInterceptor开始逐层上传直到用户的自定义拦截器，每层都会对返回的响应做出相应处理，最终将处理好的响应结果返回给用户。
