```toml
title = "重温Volley源码(三)：添加Cookie或Https的能力"
slug = "08-review-volley-3"
desc = "08 review volley 3"
date = "2017-04-24 14:55:04"
update_date = "2017-04-24 14:55:04"
author = "wangganxin"
thumb = ""
tags = ["Volley"]
```

>目录<br>
>
>一、Cookie设置与持久化<br>
>二、设置Https<br>
>参考资料<br>

### 一、Cookie设置

##### 方案一：通过Volley自定义Request对象进行设置

Request是Volley的一个抽象请求类，我们可以自定义实现里面的抽象方法来完成对Cookie的获取和保存，如：

```Java
public class CookieStringRequest extends Request<String> {

    private static final String SET_COOKIE_KEY = "Set-Cookie";
    private static final String COOKIE_KEY = "Cookie";

    private Listener<String> mListener;

    public CookieStringRequest(int method, String url, Listener<String> listener,
                               ErrorListener errorListener) {
        super(method, url, errorListener);
        mListener = listener;
    }

    public CookieStringRequest(String url, Listener<String> listener, ErrorListener errorListener) {
        this(Method.GET, url, listener, errorListener);
    }

    @Override
    protected void onFinish() {
        super.onFinish();
        mListener = null;
    }

    @Override
    protected void deliverResponse(String response) {
        if (mListener != null) {
            mListener.onResponse(response);
        }
    }

    @Override
    protected Response<String> parseNetworkResponse(NetworkResponse response) {
        String parsed;
        try {

            Map<String, String> headers=response.headers;

            if(headers!=null&&headers.containsKey(SET_COOKIE_KEY)){
                String cookie=headers.get(SET_COOKIE_KEY);

                if(!TextUtils.isEmpty(cookie)){
                    // TODO: 将cookie存到本地，如Sharepreference
                }
            }

            parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers));

        } catch (UnsupportedEncodingException e) {
            parsed = new String(response.data);
        }
        return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
    }

    @Override
    public Map<String, String> getHeaders() throws AuthFailureError {

        Map<String, String> headers=super.getHeaders();

        if(headers==null||headers.equals(Collections.emptyMap())){
            headers=new HashMap<>();
        }

        // TODO: 从本地获取到cookie，并把cookie添加到header中
        String value=getCookie();
        headers.put(COOKIE_KEY,value);

        return headers;
        //return super.getHeaders();
    }

    /**
     * 获取cookie
     * @return
     */
    private String getCookie() {
        return null;
    }
}

```

<!--more-->

在parseNetworkResponse中获取Cookie并保存到本地，getHeaders方法从本地获取到Cookie后，请求时会添加到头部。

这种方案Volley默认会返回一个Cookie，如果返回多个Cookie的情况可能我们需要修改下源码了，先来看看原来Volley是怎样去拿header的，在HurlStack类的performRequest方法中可以看到：

```Java

public class HurlStack implements HttpStack {
	......
    @Override
    public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
            throws IOException, AuthFailureError {
        ......

        for (Entry<String, List<String>> header : connection.getHeaderFields().entrySet()) {
            if (header.getKey() != null) {
				//get(0)即默认只获取第一条数据
                Header h = new BasicHeader(header.getKey(), header.getValue().get(0));
                response.addHeader(h);
            }
        }
        return response;
    }
}

```

找到了原因所在，只要对症下药，那么在遍历的时候把所有的key-value一并返回即可，修改后如下：

```Java

public class HurlStack implements HttpStack {
	......
    @Override
    public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
            throws IOException, AuthFailureError {
        ......

        for (Entry<String, List<String>> header : connection.getHeaderFields().entrySet()) {
            if (header.getKey() != null) {
                StringBuilder builder = new StringBuilder();
                //获取一个key中的所有value
                for(int i=0;i<header.getValue().size();i++){
                    builder.append(header.getValue().get(i));
                }
                Header h = new BasicHeader(header.getKey(),builder.toString());
                response.addHeader(h);
            }
        }
        return response;
    }
}

```
修改完当我们使用StringRequest或其他Request时，只要重写parseNetworkResponse方法获取Cookie信息即可，同时还需要注意下此时返回的数据格式，是所有Cookie连在一起的字符串，使用的时候按规则解析一下即可。


##### 方案二：使用Java提供的CookieManager和自定义的CookieStore进行设置

首先实现一个自定义的AppCookieStore类，它实现自CookieStore接口，在这个接口里进行我们App的本地持久化和获取操作，供Java自定调用，具体的实现网上有很多示例，下面给出伪代码：

```Java

public class AppCookieStore implements CookieStore{
    @Override
    public void add(URI uri, HttpCookie httpCookie) {

    }

    @Override
    public List<HttpCookie> get(URI uri) {
        return null;
    }

    @Override
    public List<HttpCookie> getCookies() {
        return null;
    }

    @Override
    public List<URI> getURIs() {
        return null;
    }

    @Override
    public boolean remove(URI uri, HttpCookie httpCookie) {
        return false;
    }

    @Override
    public boolean removeAll() {
        return false;
    }
}


```

接着在我们的网络请求之前调用以下代码，就可以将自定义的CookieStore设置到CookieManager中：

```Java

CookieManager cookieManager=new java.net.CookieManager(new AppCookieStore(), CookiePolicy.ACCEPT_ALL);
        CookieHandler.setDefault(cookieManager);

```

CookieManager会在之后的Http请求中自动帮我们处理response中的Cookie，当然我们也可以把它写在Application的onCreate中。

在请求的时候和方案一类似，在自定义Request对象的getHeaders方法中传入我们需要的Coockies即可

```Java

    @Override
    public Map<String, String> getHeaders() throws AuthFailureError {

        Map<String, String> headers = super.getHeaders();

        if (headers == null || headers.equals(Collections.emptyMap())) {
            headers = new HashMap<>();
        }

        // 获取之前创建的CookieManager对象，调用getCookieStore方法拿到HttpCookie列表
        CookieManager cookieManager = getCookieManager();
        List<HttpCookie> httpCookies = cookieManager.getCookieStore().getCookies();
        StringBuilder cookieBuilder = new StringBuilder();

        String divider = "";
        for (HttpCookie cookie : httpCookies) {
            cookieBuilder.append(divider);
            divider = ";";
            cookieBuilder.append(cookie.getName());
            cookieBuilder.append("=");
            cookieBuilder.append(cookie.getValue());
        }

        String cookieString = cookieBuilder.toString();

        headers.put(COOKIE_KEY, cookieString);

        return headers;
        //return super.getHeaders();
    }

```

### 二、Https设置

Https简单的理解就是http+ssl，ssl即安全套接层，使用Https加密前我们需要准备ssl证书文件（crt、cet、pem格式等），其实Volley是可以支持Https的，但是源码中并未启用，我们可以看一下HurlStack这个类：

```Java

public class HurlStack implements HttpStack {
	......
    public HurlStack() {
        this(null);
    }

    public HurlStack(UrlRewriter urlRewriter) {
        this(urlRewriter, null);
    }

    public HurlStack(UrlRewriter urlRewriter, SSLSocketFactory sslSocketFactory) {
        mUrlRewriter = urlRewriter;
        mSslSocketFactory = sslSocketFactory;
    }
	......
}

```

`HurlStack(UrlRewriter urlRewriter, SSLSocketFactory sslSocketFactory)`这个构造方法仅为`HurlStack(UrlRewriter urlRewriter)`所调用，并传入了一个null的SSLSocketFactory，所以，知道了问题的根源，我们只要创建HurlStack对象的时候调用第三个构造函数，并传入相应的SSLSocketFactory对象即可。

这里给出一个网上写好的示例，替换掉原有源码里的Volley类：

```Java

public class Volley {

    /**
     * Default on-disk cache directory.
     */
    private static final String DEFAULT_CACHE_DIR = "volley";
    private static BasicNetwork network;
    private static RequestQueue queue;

    private Context mContext;

    /**
     * Creates a default instance of the worker pool and calls
     * {@link RequestQueue#start()} on it.
     *
     * @param context A {@link Context} to use for creating the cache dir.
     * @param stack   An {@link HttpStack} to use for the network, or null for
     *                default.
     * @return A started {@link RequestQueue} instance.
     */
    public static RequestQueue newRequestQueue(Context context,
                                               HttpStack stack, boolean selfSignedCertificate, int rawId) {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(
                    packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }

        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                if (selfSignedCertificate) {
                    stack = new HurlStack(null, buildSSLSocketFactory(context,
                            rawId));
                } else {
                    stack = new HurlStack();
                }
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See:
                // http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                if (selfSignedCertificate)
                    stack = new HttpClientStack(getHttpClient(context, rawId));
                else {
                    stack = new HttpClientStack(
                            AndroidHttpClient.newInstance(userAgent));
                }
            }
        }

        if (network == null) {
            network = new BasicNetwork(stack);
        }
        if (queue == null) {
            queue = new RequestQueue(new DiskBasedCache(cacheDir),network);
        }
        queue.start();

        return queue;
    }

    /**
     * Creates a default instance of the worker pool and calls
     * {@link RequestQueue#start()} on it.
     *
     * @param context A {@link Context} to use for creating the cache dir.
     * @return A started {@link RequestQueue} instance.
     */
    public static RequestQueue newRequestQueue(Context context) {
        // 如果你目前还没有证书,那么先用下面的这行代码,http可以照常使用.
        //       return newRequestQueue(context, null, false, 0);
        // 此处R.raw.certificateName 表示你的证书文件,替换为自己证书文件名字就好
        return newRequestQueue(context, null, true, R.raw.certificateName);
    }

    private static SSLSocketFactory buildSSLSocketFactory(Context context,
                                                          int certRawResId) {
        KeyStore keyStore = null;
        try {
            keyStore = buildKeyStore(context, certRawResId);
        } catch (KeyStoreException e) {
            e.printStackTrace();
        } catch (CertificateException e) {
            e.printStackTrace();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }

        String tmfAlgorithm = TrustManagerFactory.getDefaultAlgorithm();
        TrustManagerFactory tmf = null;
        try {
            tmf = TrustManagerFactory.getInstance(tmfAlgorithm);
            tmf.init(keyStore);

        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (KeyStoreException e) {
            e.printStackTrace();
        }

        SSLContext sslContext = null;
        try {
            sslContext = SSLContext.getInstance("TLS");
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        try {
            sslContext.init(null, tmf.getTrustManagers(), null);
        } catch (KeyManagementException e) {
            e.printStackTrace();
        }

        return sslContext.getSocketFactory();

    }

    private static HttpClient getHttpClient(Context context, int certRawResId) {
        KeyStore keyStore = null;
        try {
            keyStore = buildKeyStore(context, certRawResId);
        } catch (KeyStoreException e) {
            e.printStackTrace();
        } catch (CertificateException e) {
            e.printStackTrace();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
        if (keyStore != null) {
        }
        org.apache.http.conn.ssl.SSLSocketFactory sslSocketFactory = null;
        try {
            sslSocketFactory = new org.apache.http.conn.ssl.SSLSocketFactory(
                    keyStore);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (KeyManagementException e) {
            e.printStackTrace();
        } catch (KeyStoreException e) {
            e.printStackTrace();
        } catch (UnrecoverableKeyException e) {
            e.printStackTrace();
        }

        HttpParams params = new BasicHttpParams();

        SchemeRegistry schemeRegistry = new SchemeRegistry();
        schemeRegistry.register(new Scheme("http", PlainSocketFactory
                .getSocketFactory(), 80));
        schemeRegistry.register(new Scheme("https", sslSocketFactory, 443));

        ThreadSafeClientConnManager cm = new ThreadSafeClientConnManager(
                params, schemeRegistry);

        return new DefaultHttpClient(cm, params);
    }

    private static KeyStore buildKeyStore(Context context, int certRawResId)
            throws KeyStoreException, CertificateException,
            NoSuchAlgorithmException, IOException {
        String keyStoreType = KeyStore.getDefaultType();
        KeyStore keyStore = KeyStore.getInstance(keyStoreType);
        keyStore.load(null, null);

        Certificate cert = readCert(context, certRawResId);
        keyStore.setCertificateEntry("ca", cert);

        return keyStore;
    }

    private static Certificate readCert(Context context, int certResourceID) {
        InputStream inputStream = context.getResources().openRawResource(
                certResourceID);
        Certificate ca = null;

        CertificateFactory cf = null;
        try {
            cf = CertificateFactory.getInstance("X.509");
            ca = cf.generateCertificate(inputStream);

        } catch (CertificateException e) {
            e.printStackTrace();
        }
        return ca;
    }
}

```

在项目新建raw文件夹，然后将SSL证书拷贝放到该目录，修改Volley的newRequestQueue方法即可。另外，由于在Android API 23中已经废弃了HttpClient，如果你的项目compileSdkVersion>=23，使用上述Volley源码时需降级编译。

###参考资料

- [android-volley](https://github.com/mcxiaoke/android-volley) 
- [ Android中Cookie的持久化（包含Volley的Cookie持久化）](http://blog.csdn.net/u010940300/article/details/51444909) 
- [  Android进阶——Volley+Https给你的安卓应用加上SSL证书](http://blog.csdn.net/haovip123/article/details/49509045) 


