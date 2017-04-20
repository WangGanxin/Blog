```toml
title = "重温Volley源码(一)：工作流程"
slug = "06-review-volley-1"
desc = "06 review volley 1"
date = "2017-04-20 14:55:04"
update_date = "2017-04-20 14:55:04"
author = "wangganxin"
thumb = ""
tags = ["Volley"]
```

>目录<br>
>
>一、写在前面<br>
>二、工作流程<br>
>参考资料<br>

###一、写在前面
Volley是Google在2013年I/O大会上推出的Android异步网络请求框架和图片加载框架，新技术的日新月异发展，感觉已经慢慢被OkHttp替代了，现在重新去读它的源码，虽然稍显得有些过气，但还是有很大的学习价值的，在此记录下自己的脚印。

先来看看关于Volley的几个关键特点：

- 适合数据量小，通信频繁的网络操作
- 基于接口设计，面向接口编程，可扩展性强
- 一定程度符合Http规范（ResponseCode请求响应码、请求头处理、缓存机制、请求重试、优先级定义）
- Android 2.2以下基于HttpClient，2.3及以上基于HttpUrlConnenction
- 提供了简便的图片加载工具

###二、工作流程

既然是探索Volley的工作流程，我们可以一步步追踪其源码，先来看一个典型的发送Volley网络请求的用法：

```Java

		//1、创建请求队列
		RequestQueue mQueue = Volley.newRequestQueue(this);

		//2、创建一个网络请求
        StringRequest stringRequest = new StringRequest("https://www.baidu.com",
                new Response.Listener<String>() {
                    @Override
                    public void onResponse(String response) {
							Log.i(TAG, response);
                    }
                }, new Response.ErrorListener() {
            @Override
            public void onErrorResponse(VolleyError error) {
      				 Log.i(TAG, error.getMessage(), error);          
            }
        });

		//3、将网络请求添加到请求队列中
        mQueue.add(stringRequest);

```

简简单单的三个步骤，完成了一个网络请求，并在onResponse中返回。那么Volley里面具体帮我们干了哪些事情呢，我们进入`Volley.newRequestQueue(this)` 这个方法内部一探究竟：

```Java

    public static RequestQueue newRequestQueue(Context context, HttpStack stack, int maxDiskCacheBytes) {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }

        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);
        
        RequestQueue queue;
        if (maxDiskCacheBytes <= -1)
        {
        	// No maximum size specified
        	queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        }
        else
        {
        	// Disk cache size specified
        	queue = new RequestQueue(new DiskBasedCache(cacheDir, maxDiskCacheBytes), network);
        }

        queue.start();

        return queue;
    }

```

- 如果HttpStack参数为null，则根据系统在API Level>=9采用HurlStack(内部为HttpUrlConnection)，如果<9，采用基于HttpClientStack 
- 根据HttpStack 创建一个NetWork的具体实现类BasicNetwork对象
- 根据DiskBasedCache磁盘缓存对象、network对象构建一个RequestQueue，调用RequestQueue的start方法启动。

> 通过源码可以看出，我们可以抛开Volley工具类构建自定义的RequestQueue，采用自定义的HttpStack，采用自定义的NetWork实现，采用自定义的Cache实现来构建RequestQueue，Volley的面向接口编程，高可拓展性的魅力就源于此。

<br/>

> 关于HttpURLConnection 和 HttpClient的选择及原因 <br/>
> 1. 在Android2.2之前，HttpURLConnection 有个重大的bug，调用 close() 函数会影响连接池，导致连接复用失效，所以在Android2.2之前使用 HttpURLConnection 需要关闭 keepAlive <br/>
> 2. 在Android2.3，HttpURLConnection 默认开启了 gzip 压缩，提高了 HTTPS 的性能；在Android 4.0,HttpURLConnection 支持了请求结果缓存<br/>
>
>HttpURLConnection 本身 API 相对简单，所以对 Android 来说，在2.3之前建议使用HttpClient，之后建议使用 HttpURLConnection。

本着对Volley请求执行流程的侧重把握，我们接着看RequestQueue的start方法：

```Java

    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }

```

- 调用stop方法，停止所有的线程（CacheDispatcher 和 NetworkDispatcher）
- 创建一个缓存调度线程CacheDispatcher并启动
- 创建n个网络调度线程NetworkDispatcher并启动

>在这段start方法执行完后，Volley就已经默认创建了5个线程（1个CacheDispatcher+4个NetworkDispatcher），这里存在优化的余地，比如我们可以根据CPU核数以及网络类型计算更合适的并发数

start方法执行完，由此得到RequestQueue，我们只需要构建相应的Request，然后调用RequstQueue的add方法，就可以完成网络请求操作。

**关于Request类**

> Request是一个网络请求的抽象类，非抽象子类有StringRequest、JsonRequest、ImageRequest或者自定义子类，我们通过构建这个对象，将其加入RequestQueue来完成一次网络请求操作 <br/>
> Request子类必须实现的方法有两个:parseNetworkResponse 和 deliverResponse
> Volley支持8中请求方式：**GET**、**POST**、**PUT**、**DELETE**、**HEAD**、**OPTIONS**、**TRACE**、**PATCH** <br/>
> Request类包含了请求URL，请求方式，请求Header，请求Body，请求优先级等信息

RequestQueue的add方法内部实现：

```Java

    public <T> Request<T> add(Request<T> request) {
        // Tag the request as belonging to this queue and add it to the set of current requests.
        request.setRequestQueue(this);
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);
        }

        // Process requests in the order they are added.
        request.setSequence(getSequenceNumber());
        request.addMarker("add-to-queue");

        // If the request is uncacheable, skip the cache queue and go straight to the network.
        if (!request.shouldCache()) {
            mNetworkQueue.add(request);
            return request;
        }

        // Insert request into stage if there's already a request with the same cache key in flight.
        synchronized (mWaitingRequests) {
            String cacheKey = request.getCacheKey();
            if (mWaitingRequests.containsKey(cacheKey)) {
                // There is already a request in flight. Queue up.
                Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
                if (stagedRequests == null) {
                    stagedRequests = new LinkedList<Request<?>>();
                }
                stagedRequests.add(request);
                mWaitingRequests.put(cacheKey, stagedRequests);
                if (VolleyLog.DEBUG) {
                    VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
                }
            } else {
                // Insert 'null' queue for this cacheKey, indicating there is now a request in
                // flight.
                mWaitingRequests.put(cacheKey, null);
                mCacheQueue.add(request);
            }
            return request;
        }
    }

```

- 判断是否可以缓存，如果不能缓存则直接加入网络请求队列mNetworkQueue，能缓存则只需执行加入到缓存队列mCacheQueue中
- 默认情况下，Volley的每条请求都是可以缓存的，如果不需要缓存，可以调用Request的setShouldCache(false)方法来改变这一默认行为

既然将请求加入到mNetworkQueue或者mCacheQueue中，接下来就是在对应的NetworkDispatcher或CacheDispatcher线程中执行了。

这里只看NetworkDispatcher的run方法（CacheDispatcher的run方法后半部分和这里类似）

```Java

    public class NetworkDispatcher extends Thread { 
    .....
    @Override
    public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        Request<?> request;
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            // release previous request object to avoid leaking request object when mQueue is drained.
            request = null;
            try {
                // Take a request from the queue.
                request = mQueue.take();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }

            try {
                request.addMarker("network-queue-take");

                // If the request was cancelled already, do not perform the
                // network request.
                if (request.isCanceled()) {
                    request.finish("network-discard-cancelled");
                    continue;
                }

                addTrafficStatsTag(request);

                // Perform the network request.
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
                // we're done -- don't deliver a second identical response.
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                mDelivery.postError(request, volleyError);
            }
        }
    }
	}

```

- 外部while（true）的死循环，说明网络线程始终运行
- 调用mNetwork的performRequest方法，将request对象传进入，执行具体的网络请求
- 根据请求的返回值，调用Request的parseNetworkResponse方法来解析NetworkResponse的数据，以及将数据写入到缓存，这个方法的实现是交给Request的子类来完成的，因为不同种类的Request解析的方式也不同，就像在自定义Request中，必须重写parseNetworkResponse方法一样。

在解析完NetworkResponse的数据后，紧接着会调用ResponseDelivery的实现子类ExecutorDelivery的postResponse方法来回调解析出的数据：

```Java

    @Override
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }

```

mResponsePoster的execute方法传入了一个ResponseDeliveryRunnable对象：

```Java

    private class ResponseDeliveryRunnable implements Runnable {
        private final Request mRequest;
        private final Response mResponse;
        private final Runnable mRunnable;

        public ResponseDeliveryRunnable(Request request, Response response, Runnable runnable) {
            mRequest = request;
            mResponse = response;
            mRunnable = runnable;
        }

        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            // If this request has canceled, finish it and don't deliver.
            if (mRequest.isCanceled()) {
                mRequest.finish("canceled-at-delivery");
                return;
            }

            // Deliver a normal response or error, depending.
            if (mResponse.isSuccess()) {
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }

            // If this is an intermediate response, add a marker, otherwise we're done
            // and the request can be finished.
            if (mResponse.intermediate) {
                mRequest.addMarker("intermediate-response");
            } else {
                mRequest.finish("done");
            }

            // If we have been provided a post-delivery runnable, run it.
            if (mRunnable != null) {
                mRunnable.run();
            }
       }
    }

```

可以看出实现了一个Runnable接口，在run方法内部判断是否响应成功，如果成功则调用Request的deliverResponse方法，否则调用deliverError方法。这里的deliverResponse方法内部最终会回调我们在构建Request时设置的`Response.Listener`对象，例如StringRequest的deliverResponse内部代码如下。

```Java

	public class StringRequest extends Request<String> {
    ......
    @Override
    protected void deliverResponse(String response) {
        if (mListener != null) {
            mListener.onResponse(response);
        }
    }
	}
```

其实performRequest内部转换成Response的处理过程，这里借用[Volley源码解析](http://a.codekk.com/detail/Android/grumoon/Volley%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90) 里的一张图，更加从宏观上清晰的说明问题了：

![volley-response-process](/media/2017/volley-response-process-flow-chart.png)

从上到下表示从得到数据后一步步的处理，箭头旁的注释表示该步处理后的实体类。

**关于Cache类**
> 1. 缓存接口，代表一个可以获取请求结果、存储请求结果的缓存<br/>
> 2. 默认的两个实现子类：NoCache和DiskBasedCache
> 3. DiskBasedCache类会把从服务器返回的信息写入磁盘，然后从磁盘取出缓存，这其中涉及了一些静态的方法如writeInt、writeLong等等，何解？原因之一是Java的IO本身是对byte进行操作，一个int占4个byte，需要按位写入，另一方面也是因为网络字节序是大端字节序，在80x86的平台中，是以小端法存放的，比如我们经过网络发送0x12345678这个整型，但实际上在流中是0x87654321


好了，到这里Volley的整体流程大概梳理了一遍，可能稍微讲得有点乱，当然也仅仅是个人的记录为主，最后，放上Volley官方的请求流程图镇楼（原本想着自己画一张流程图，但翻了翻发觉画不出比这个更好的了）：

![volley-request](/media/2017/volley-request.png)

###参考资料

- [android-volley](https://github.com/mcxiaoke/android-volley) 
- [Volley源码解析](http://a.codekk.com/detail/Android/grumoon/Volley%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90) 
- [Android Volley完全解析(四)，带你从源码的角度理解Volley](http://blog.csdn.net/guolin_blog/article/details/17656437) 

