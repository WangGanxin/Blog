```toml
title = "重温Volley源码(二)：重试策略"
slug = "07-review-volley-2"
desc = "07 review volley 2"
date = "2017-04-21 14:55:04"
update_date = "2017-04-21 14:55:04"
author = "wangganxin"
thumb = ""
tags = ["Volley"]
```

>目录<br>
>
>一、核心类<br>
>二、重试策略<br>
>参考资料<br>

本文是建立在对Volley框架的工作流程有一定基础了解的情况下继续深入的，这里顺便贴上我写的上篇文章[《重温Volley源码(一)：工作流程》](https://github.com/mcxiaoke/android-volley) ，欢迎阅读指正。

###一、核心类

**RetryPolicy**：Volley定义的请求重试策略接口

<!--more-->

```Java

public interface RetryPolicy {

    /**当前请求的超时时间
     * Returns the current timeout (used for logging).
     */
    public int getCurrentTimeout();

    /**当前请求重试的次数
     * Returns the current retry count (used for logging).
     */
    public int getCurrentRetryCount();

    /**在请求异常时调用此方法
     * Prepares for the next retry by applying a backoff to the timeout.
     * @param error The error code of the last attempt.
     * @throws VolleyError In the event that the retry could not be performed (for example if we
     * ran out of attempts), the passed in error is thrown.
     */
    public void retry(VolleyError error) throws VolleyError;
}

```

**DefaultRetryPolicy**：RetryPolicy的实现子类

```Java

public class DefaultRetryPolicy implements RetryPolicy {
    /** The current timeout in milliseconds. 当前超时时间*/
    private int mCurrentTimeoutMs; 

    /** The current retry count. 当前重试次数*/
    private int mCurrentRetryCount;

    /** The maximum number of attempts. 最大重试次数*/
    private final int mMaxNumRetries;

    /** The backoff multiplier for the policy. 超时时间的乘积因子*/
    private final float mBackoffMultiplier;

    /** The default socket timeout in milliseconds 默认超时时间*/
    public static final int DEFAULT_TIMEOUT_MS = 2500;

    /** The default number of retries 默认的重试次数*/
    public static final int DEFAULT_MAX_RETRIES = 0;

    /** The default backoff multiplier 默认超时时间的乘积因子*/
	/**
    *   以默认超时时间为2.5s为例
	*   DEFAULT_BACKOFF_MULT = 1f, 则每次HttpUrlConnection设置的超时时间都是2.5s*1f*mCurrentRetryCount
	*   DEFAULT_BACKOFF_MULT = 2f, 则第二次超时时间为:2.5s+2.5s*2=7.5s,第三次超时时间为:7.5s+7.5s*2=22.5s
	*/
    public static final float DEFAULT_BACKOFF_MULT = 1f;


    /**
     * Constructs a new retry policy using the default timeouts.
     */
    public DefaultRetryPolicy() {
        this(DEFAULT_TIMEOUT_MS, DEFAULT_MAX_RETRIES, DEFAULT_BACKOFF_MULT);
    }

    /**
     * Constructs a new retry policy.
     * @param initialTimeoutMs The initial timeout for the policy.
     * @param maxNumRetries The maximum number of retries.
     * @param backoffMultiplier Backoff multiplier for the policy.
     */
    public DefaultRetryPolicy(int initialTimeoutMs, int maxNumRetries, float backoffMultiplier) {
        mCurrentTimeoutMs = initialTimeoutMs;
        mMaxNumRetries = maxNumRetries;
        mBackoffMultiplier = backoffMultiplier;
    }

    /**
     * Returns the current timeout.
     */
    @Override
    public int getCurrentTimeout() {
        return mCurrentTimeoutMs;
    }

    /**
     * Returns the current retry count.
     */
    @Override
    public int getCurrentRetryCount() {
        return mCurrentRetryCount;
    }

    /**
     * Returns the backoff multiplier for the policy.
     */
    public float getBackoffMultiplier() {
        return mBackoffMultiplier;
    }

    /**
     * Prepares for the next retry by applying a backoff to the timeout.
     * @param error The error code of the last attempt.
     */
    @Override
    public void retry(VolleyError error) throws VolleyError {
		//当前重试次数++
        mCurrentRetryCount++;
		//当前超时时间计算
        mCurrentTimeoutMs += (mCurrentTimeoutMs * mBackoffMultiplier);
		//判断是否还有剩余次数，如果没有则抛出VolleyError异常
        if (!hasAttemptRemaining()) {
            throw error;
        }
    }

    /**
	 * 判断当前Request的重试次数是否超过最大重试次数
     * Returns true if this policy has attempts remaining, false otherwise.
     */
    protected boolean hasAttemptRemaining() {
        return mCurrentRetryCount <= mMaxNumRetries;
    }
}

```

###二、重试策略

在深入源码阅读你会发现，retry方法会抛出VolleyError的异常，但该方法内部并不是重新发起了网络请求，而是变更重试策略的属性，如超时时间和重试次数，当超过重试策略设定的限定就会抛异常，这个可以在DefaultRetryPolicy里得到验证。那么它究竟是如何做到重试的呢，我们可以跟踪源码 retry 方法被调用到的地方，来到了BasicNetwork的attemptRetryOnException方法：

```Java

public class BasicNetwork implements Network {

......

	/**
	*  尝试对一个请求进行重试策略，
	*/
    private static void attemptRetryOnException(String logPrefix, Request<?> request,
            VolleyError exception) throws VolleyError {
        RetryPolicy retryPolicy = request.getRetryPolicy(); //获取该请求的重试策略
        int oldTimeout = request.getTimeoutMs(); //获取该请求的超时时间

        try {
            retryPolicy.retry(exception); //内部实现重试次数、超时时间的变更，如果重试次数超过最大限定次数，该方法抛出异常
        } catch (VolleyError e) {
			//当超过最大重试次数，捕获到异常，更改该请求的标记
            request.addMarker(
                    String.format("%s-timeout-giveup [timeout=%s]", logPrefix, oldTimeout));
            //当仍然可以进行重试的时候，不会执行到catch语句，但是当执行到catch语句的时候，表示已经不能进行重试了，就抛出异常  中断while(true)循环
			throw e;
        }
		//给请求添加标记，请求了多少次  
        request.addMarker(String.format("%s-retry [timeout=%s]", logPrefix, oldTimeout));
    }

}

```

继续跟踪attemptRetryOnException方法被调用的地方，来到了BasicNetwork的performRequest方法：

```Java

public class BasicNetwork implements Network {

......

    @Override
    public NetworkResponse performRequest(Request<?> request) throws VolleyError {
        long requestStart = SystemClock.elapsedRealtime();
        while (true) {
            ......

            try {
                ......

				//如果发生了超时、认证失败等错误，进行重试操作，直到成功。若attemptRetryOnException再抛出异常则结束
				//当catch后没有执行上面的return，而当前又是一个while（true）循环，可以保证下面的请求重试的执行，是利用循环进行请求重试，请求重试策略只是记录重试的次数、超时时间等内容
                return new NetworkResponse(statusCode, responseContents, responseHeaders, false,
                        SystemClock.elapsedRealtime() - requestStart);
            } catch (SocketTimeoutException e) {
				//1.尝试进行请求重试
                attemptRetryOnException("socket", request, new TimeoutError());
            } catch (ConnectTimeoutException e) {
				//2.尝试进行请求重试
                attemptRetryOnException("connection", request, new TimeoutError());
            } catch (MalformedURLException e) {
                throw new RuntimeException("Bad URL " + request.getUrl(), e);
            } catch (IOException e) {
                int statusCode = 0;
                NetworkResponse networkResponse = null;
                if (httpResponse != null) {
                    statusCode = httpResponse.getStatusLine().getStatusCode();
                } else {
                    throw new NoConnectionError(e);
                }
                if (statusCode == HttpStatus.SC_MOVED_PERMANENTLY || 
                		statusCode == HttpStatus.SC_MOVED_TEMPORARILY) {
                	VolleyLog.e("Request at %s has been redirected to %s", request.getOriginUrl(), request.getUrl());
                } else {
                	VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
                }
                if (responseContents != null) {
                    networkResponse = new NetworkResponse(statusCode, responseContents,
                            responseHeaders, false, SystemClock.elapsedRealtime() - requestStart);
                    if (statusCode == HttpStatus.SC_UNAUTHORIZED ||
                            statusCode == HttpStatus.SC_FORBIDDEN) {
						//3.尝试进行请求重试
                        attemptRetryOnException("auth",
                                request, new AuthFailureError(networkResponse));
                    } else if (statusCode == HttpStatus.SC_MOVED_PERMANENTLY || 
                    			statusCode == HttpStatus.SC_MOVED_TEMPORARILY) {
						//4.尝试进行请求重试
                        attemptRetryOnException("redirect",
                                request, new RedirectError(networkResponse));
                    } else {
                        // TODO: Only throw ServerError for 5xx status codes.
                        throw new ServerError(networkResponse);
                    }
                } else {
                    throw new NetworkError(e);
                }
            }
        }
    }

}

```

performRequest这个方法名是否很眼熟？是的，它曾被我在上一篇文章中简单提到过，还记得在NetworkDispatcher的run方法中，mNetWork对象（即BasicNetwork）会调用performRequest来执行请求，在该方法内部又是一个while（true）循环

在这个方法里attemptRetryOnException总共被调用了四次：

- 发生SocketTimeoutException时（Socket通信超时，即从服务端读取数据时超时）
- 发生ConnectTimeoutException时（请求超时，即连接HTTP服务端超时或者等待HttpConnectionManager返回可用连接超时）
- 发生IOException，相应的状态码401/403(授权未通过)时
- 发生IOException，相应的状态码301/302（URL发生转移）时


>现在我们归纳一下，首先假设我们设置了请求重试的策略（Volley默认最大请求重试次数为0，即不重试），其次BasicNetwork的performRequest方法的外面其实是个while循环，假如在网络请求过程中发生了异常，如超时、认证未通过等上述四个被调用的异常，catch这个异常的代码会看看是否可以重试，如果可以重试，就会把这个异常吞噬掉，然后进入下一次循环，否则超过重试次数，attemptRetryOnException内部再抛出异常，此时**交由上一层代码去处理**，并退出while循环。

可能还有个疑问，上面说的**交由上一层代码去处理**是到底是怎么处理并退出while循环的？这里因为BasicNetwork的performRequest方法并没有捕获VolleyError异常，因此没有被try&catch住的异常会继续往外抛出，这里我们回过头来看看NetworkDispatcher的run方法里头：

```Java
public class NetworkDispatcher extends Thread {

    ......

    @Override
    public void run() {

		......

        while (true) {

			......

            try {

				......


                // Perform the network request. 真正执行网络请求的地方，BasicNetwork超时抛出的VolleyError最终会抛出到这里
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                
				......

                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
				// 捕获VolleyError异常,通过主线程Handler回调用户设置的ErrorListener中的onErrorResponse回调方法
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

Request无法继续重试后抛出的VolleyError异常，会被NetworkDispatcher捕获，然后利用Delivery去回调用户设置的ErrorListener。


###参考资料

- [android-volley](https://github.com/mcxiaoke/android-volley) 
- [Volley源码解析](http://a.codekk.com/detail/Android/grumoon/Volley%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90) 
- [Android中的volley_12_请求重试策略RetryPolicy和DefaultRetryPolicy](http://blog.csdn.net/vvzhouruifeng/article/details/46385275) 


