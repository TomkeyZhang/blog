如果不知道Volley框架是什么，请看此帖<a href="http://blog.csdn.net/t12x3456/article/details/9221611">http://blog.csdn.net/t12x3456/article/details/9221611</a>

或者下载官方演讲pdf <a href="http://bcs.duapp.com/myblog-wrodpress//blog/201403//GoogleIO2013：Android简单_快速联网_Volley.pdf">GoogleIO2013：Android简单_快速联网_Volley</a>

一、Volley框架的官方架构图：

<a href="http://bcs.duapp.com/myblog-wrodpress//blog/201403//1.png"><img class="alignnone size-full wp-image-8" alt="Volley框架官方架构图" src="http://bcs.duapp.com/myblog-wrodpress//blog/201403//1.png" width="1276" height="715" /></a>

总的来说，就是一个请求队列和三种线程，main线程（一个），cache线程（一个）和network线程（多个）。main线程负责添加请求任务，执行任务结果；cache线程负责检查缓存，命中后直接将任务结果分发到主线程；network线程由多个任务线程（NetworkDispatcher）组成的，相当于一个大小为size的线程池，这些线程会同时启动，并持续的从任务队列中获取待执行的任务，任务执行完后会将结果分发到主线程。本文先介绍volley的http请求部分。<!--more-->

二、Volley框架http请求处理程序结构图

<a href="http://bcs.duapp.com/myblog-wrodpress//blog/201404//volley1.png"><img class="alignnone size-full wp-image-15" alt="volley" src="http://bcs.duapp.com/myblog-wrodpress//blog/201404//volley1.png" width="1021" height="720" /></a>

三、部分源码解读
创建请求队列包装类RequestQueue
<pre>/**
     * Creates a default instance of the worker pool and calls {@link RequestQueue#start()} on it.
     *
     * @param context A {@link Context} to use for creating the cache dir.
     * @param stack An {@link HttpStack} to use for the network, or null for default.
     * @return A started {@link RequestQueue} instance.
     */
    public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }

        if (stack == null) {
            if (Build.VERSION.SDK_INT &gt;= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);

        RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        queue.start();

        return queue;
    }</pre>
其中的参数HttpStack是一个请求客户端的抽象
<pre>/**
 * An HTTP stack abstraction.
 */
public interface HttpStack {
    /**
     * Performs an HTTP request with the given parameters.
     *
     *</pre>
A GET request is sent if request.getPostBody() == null. A POST request is sent otherwise, * and the Content-Type header is set to request.getPostBodyContentType().
<pre>     *
     * @param request the request to perform
     * @param additionalHeaders additional headers to be sent together with
     *         {@link Request#getHeaders()}
     * @return the HTTP response
     */
    public HttpResponse performRequest(Request&lt;?&gt; request, Map&lt;String, String&gt; additionalHeaders)
        throws IOException, AuthFailureError;

}</pre>
目前Volley里面有2种实现，Java原生的HttpURLConnection实现（HurlStack）和Apache的HttpClient实现（HttpClientStack），我们也可以将其换成另外的实现，比如Square公司的OkHttpClient实现。默认情况下，Volley会在android2.3以前使用HttpClient实现，在android2.3及以后使用HttpURLConnection实现。原因:<a href="http://developer.android.com/reference/org/apache/http/impl/client/DefaultHttpClient.html">官方说明</a>，<a href="http://stackoverflow.com/questions/4799151/apache-http-client-or-urlconnection">stackoverflow</a> ，<a href="http://blog.csdn.net/androidzhaoxiaogang/article/details/8158122">csdn</a>，<a href="http://blog.csdn.net/xyz_lmn/article/details/12200133">android developers翻译</a>。

其中Network是一个执行网络请求的抽象，
<pre>/**
 * An interface for performing requests.
 */
public interface Network {
    /**
     * Performs the specified request.
     * @param request Request to process
     * @return A {@link NetworkResponse} with data and caching metadata; will never be null
     * @throws VolleyError on errors
     */
    public NetworkResponse performRequest(Request&lt;?&gt; request) throws VolleyError;
}</pre>
实现类BasicNetwork.java，在其中会组装出各种Volley异常，并根据不同的策略执行重连的操作
<pre>@Override
    public NetworkResponse performRequest(Request&lt;?&gt; request) throws VolleyError {
        long requestStart = SystemClock.elapsedRealtime();
        while (true) {
            HttpResponse httpResponse = null;
            byte[] responseContents = null;
            Map&lt;String, String&gt; responseHeaders = new HashMap&lt;String, String&gt;();
            try {
                // Gather headers.
                Map&lt;String, String&gt; headers = new HashMap&lt;String, String&gt;();
                addCacheHeaders(headers, request.getCacheEntry());
                httpResponse = mHttpStack.performRequest(request, headers);
                StatusLine statusLine = httpResponse.getStatusLine();
                int statusCode = statusLine.getStatusCode();

                responseHeaders = convertHeaders(httpResponse.getAllHeaders());
                // Handle cache validation.
                if (statusCode == HttpStatus.SC_NOT_MODIFIED) {
                    return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED,
                            request.getCacheEntry() == null ? null : request.getCacheEntry().data,
                            responseHeaders, true);
                }

                // Some responses such as 204s do not have content.  We must check.
                if (httpResponse.getEntity() != null) {
                  responseContents = entityToBytes(httpResponse.getEntity());
                } else {
                  // Add 0 byte response as a way of honestly representing a
                  // no-content request.
                  responseContents = new byte[0];
                }

                // if the request is slow, log it.
                long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
                logSlowRequests(requestLifetime, request, responseContents, statusLine);

                if (statusCode &lt; 200 || statusCode &gt; 299) {
                    throw new IOException();
                }
                return new NetworkResponse(statusCode, responseContents, responseHeaders, false);
            } catch (SocketTimeoutException e) {
                attemptRetryOnException("socket", request, new TimeoutError());
            } catch (ConnectTimeoutException e) {
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
                VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
                if (responseContents != null) {
                    networkResponse = new NetworkResponse(statusCode, responseContents,
                            responseHeaders, false);
                    if (statusCode == HttpStatus.SC_UNAUTHORIZED ||
                            statusCode == HttpStatus.SC_FORBIDDEN) {
                        attemptRetryOnException("auth",
                                request, new AuthFailureError(networkResponse));
                    } else {
                        // TODO: Only throw ServerError for 5xx status codes.
                        throw new ServerError(networkResponse);
                    }
                } else {
                    throw new NetworkError(networkResponse);
                }
            }
        }
    }</pre>
queue.start()方法启动一个cache线程和多个网络任务线程，等待执行任务。
<pre>/**
     * Starts the dispatchers in this queue.
     */
    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i &lt; mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }</pre>
NetworkDispatcher的核心方法,主要做3个操作，一是执行request将获取一个NetworkResponse，然后将一个NetworkResponse解析为一个业务层的Response，最后将Response传递给UI线程
<pre>@Override
    public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        Request&lt;?&gt; request;
        while (true) {
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
                if (networkResponse.notModified &amp;&amp; request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
                Response&lt;?&gt; response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                if (request.shouldCache() &amp;&amp; response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                mDelivery.postError(request, new VolleyError(e));
            }
        }
    }</pre>
Request的一个实现StringRequest.java,将byte数据转化为String
<pre>@Override
    protected Response parseNetworkResponse(NetworkResponse response) {
        String parsed;
        try {
            parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers));
        } catch (UnsupportedEncodingException e) {
            parsed = new String(response.data);
        }
        return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
    }</pre>
结果分发的抽象ResponseDelivery
<pre>public interface ResponseDelivery {
    /**
     * Parses a response from the network or cache and delivers it.
     */
    public void postResponse(Request&lt;?&gt; request, Response&lt;?&gt; response);

    /**
     * Parses a response from the network or cache and delivers it. The provided
     * Runnable will be executed after delivery.
     */
    public void postResponse(Request&lt;?&gt; request, Response&lt;?&gt; response, Runnable runnable);

    /**
     * Posts an error for the given request.
     */
    public void postError(Request&lt;?&gt; request, VolleyError error);
}</pre>
实现类的核心代码ExecutorDelivery.ResponseDeliveryRunnable.run()
这个实现是通过handler.post方法来达到让代码运行在UI线程中
<pre>
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
            }</pre>
Volley的Request都是可以被取消的，取消后主线程的回调将不会执行。上面的代码检查了request是否被取消，取消后就不在分发请求，然后根据请求的结果类型进行分发，最后有一个mResponse.intermediate判断，想知道这个是用来做什么的嘛，请看下回分解~~~
