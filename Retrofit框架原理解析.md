Retrofit框架是属于Square公司的一个开源框架，其主要作者是Android大神<a href="https://github.com/JakeWharton">JakeWharton</a>，官方简介：A type-safe <strong>REST client</strong> for Android and Java。它的核心特点就是使用接口和注解来描述http请求，尽可能的减少需要编写的http请求的代码，具有很强的可读性和可维护性，同时不失灵活性。
<!--more-->
Retrofit使用示例：
<pre>
public interface RestWeiLiaoV1 {
    /**
     * 登陆
     * 
     * @param phone
     * @param user
     * @param callback
     */
    @POST("/user/login/{phone}")
    String login(@Path("phone") String phone, @Body LoginParam user);

    /**
     * 获取所有用户的消息（返回最新的xx条)
     * 
     * @param phone 手机号码（查日志使用）
     * @param last_max_msg_id 客户端最后接收的消息ID（这里固定传0，以服务器端的状态为主）
     * @param sync 是否接收同步消息，0=不接收，1=接收
     */
    @GET("/message/getAllNewMessages/{phone}/{last_max_msg_id}")
    String getAllNewSyncMessages(@Path("phone") String phone, @Path("last_max_msg_id")
            String last_max_msg_id, @Query("sync") String sync, @Query("_guid") String guid);
}
</pre>
老的api使用方式：
<pre>
/**
     * 获取带有房价信息的附件小区数据
     * 
     * @param cityId
     * @param centerLat
     * @param centerLng
     * @param commNumber
     * @param radius
     * @return
     */
    public CommunitiesWithPrice getNearCommsWithPrice(String cityId, String centerLat, String centerLng,
            int commNumber, int radius) {
        // http://minjiewang.d.corp.anjuke.com/4.0/comm/searchnearby/?city_id=11&lat=31.217642&lng=121.3870168&radius=1&limit=3&maptype=baidu
        String methodUrl = String.format("comm/searchnearby/");

        HashMap<String, String> map = new HashMap<String, String>();
        map.put("city_id", cityId);
        map.put("maptype", MAP_TYPE_BAIDU);
        map.put("lat", centerLat);
        map.put("lng", centerLng);
        map.put("limit", commNumber + "");
        map.put("radius", radius + "");

        return get(CommunitiesWithPrice.class, methodUrl, map);
    }
</pre>
以下内容的基础是需要对Retrofit框架有一个初步的了解，至少要知道怎么使用Retrofit框架，如果不了解，请看<a href="http://square.github.io/retrofit/">官方文档</a>

核心技术点：java动态代理，简单的说就是动态的生成某些接口的实现类。
<pre>/** Create an implementation of the API defined by the specified {@code service} interface. */
	@SuppressWarnings("unchecked")
	public <T> T create(Class<T> service) {
		Utils.validateServiceClass(service);
		return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service }, new RestHandler(getMethodInfoCache(service)));
	}
</pre>
具体来说，动态代理是java.lang.reflect.Proxy类动态的根据指定的所有接口生成一个class byte，该class会继承Proxy类，并实现所有指定的接口（你在参数中传入的接口数组）；然后再利用指定的classloader将 class byte加载进系统，最后生成这样一个类的对象，并初始化该对象的一些值，如invocationHandler,以及所有的接口对应的Method成员。 初始化之后将对象返回给调用的客户端。这样客户端拿到的就是一个实现你所有的接口的Proxy对象。

下面我们根据代码的逻辑一步步分析Retrofit的一次http请求的执行过程。由上面可知，我们可以拿到一个service接口的实现类，每次我们调用service接口的方法的时候，系统就会执行RestHandler（实现InvocationHandler接口）的invoke方法
<pre>
                @Override
		public Object invoke(Object proxy, Method method, final Object[] args) throws Throwable {
			// If the method is a method from Object then defer to normal invocation.
			if (method.getDeclaringClass() == Object.class) {
				return method.invoke(this, args);
			}

			// Load or create the details cache for the current method.
			final RestMethodInfo methodInfo = getMethodInfo(methodDetailsCache, method);

			if (methodInfo.isSynchronous) {
				try {
					return invokeRequest(requestInterceptor, methodInfo, args);
				} catch (RetrofitError error) {
					Throwable newError = errorHandler.handleError(error);
					if (newError == null) {
						throw new IllegalStateException("Error handler returned null for wrapped exception.", error);
					}
					throw newError;
				}
			}

			if (httpExecutor == null || callbackExecutor == null) {
				throw new IllegalStateException("Asynchronous invocation requires calling setExecutors.");
			}

			// Apply the interceptor synchronously, recording the interception so we can replay it later.
			// This way we still defer argument serialization to the background thread.
			final RequestInterceptorTape interceptorTape = new RequestInterceptorTape();

			if (methodInfo.isObservable) {
				return rxSupport.createRequestObservable(new Callable<ResponseWrapper>() {
					@Override
					public ResponseWrapper call() throws Exception {
						return (ResponseWrapper) invokeRequest(interceptorTape, methodInfo, args);
					}
				});
			}

			Callback<?> callback = (Callback<?>) args[args.length - 1];
			httpExecutor.execute(new CallbackRunnable(callback, callbackExecutor) {
				@Override
				public ResponseWrapper obtainResponse() {
					return (ResponseWrapper) invokeRequest(interceptorTape, methodInfo, args);
				}
			});
			return null; // Asynchronous methods should have return type of void.
		}
</pre>
第9行是获取方法的基本信息，主要是对方法的返回值类型进行解析。然后根据返回结果做如下处理：
1、如果是同步方法，就直接执行invokeRequest来发送http请求。此时有一个errorHandler来实现自定义的异常抛出
<pre>
Throwable handleError(RetrofitError cause);
</pre>
可以通过这个接口将RetrofitError 转化为自己的异常抛出。
2、如果是基于RXJava的观察者模式，则使用RXJava发起异步请求。
3、如果是普通方法，则使用自定义的Executor来执行http请求。

Retrofit中执行一个http请求的核心方法invokeRequest
<pre>
/**
     * Execute an HTTP request.
     *
     * @return HTTP response object of specified {@code type} or {@code null}.
     * @throws RetrofitError if any error occurs during the HTTP request.
     */
    private Object invokeRequest(RequestInterceptor requestInterceptor, RestMethodInfo methodInfo,
        Object[] args) {
      String url = null;
      try {
        methodInfo.init(); // Ensure all relevant method information has been loaded.

        String serverUrl = server.getUrl();
        RequestBuilder requestBuilder = new RequestBuilder(serverUrl, methodInfo, converter);
        requestBuilder.setArguments(args);

        requestInterceptor.intercept(requestBuilder);

        Request request = requestBuilder.build();
        url = request.getUrl();

        if (!methodInfo.isSynchronous) {
          // If we are executing asynchronously then update the current thread with a useful name.
          Thread.currentThread().setName(THREAD_PREFIX + url.substring(serverUrl.length()));
        }

        if (logLevel.log()) {
          // Log the request data.
          request = logAndReplaceRequest("HTTP", request);
        }

        Object profilerObject = null;
        if (profiler != null) {
          profilerObject = profiler.beforeCall();
        }

        long start = System.nanoTime();
        Response response = clientProvider.get().execute(request);
        long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

        int statusCode = response.getStatus();
        if (profiler != null) {
          RequestInformation requestInfo = getRequestInfo(serverUrl, methodInfo, request);
          //noinspection unchecked
          profiler.afterCall(requestInfo, elapsedTime, statusCode, profilerObject);
        }

        if (logLevel.log()) {
          // Log the response data.
          response = logAndReplaceResponse(url, response, elapsedTime);
        }

        Type type = methodInfo.responseObjectType;

        if (statusCode >= 200 && statusCode < 300) { // 2XX == successful request
          // Caller requested the raw Response object directly.
          if (type.equals(Response.class)) {
            // Read the entire stream and replace with one backed by a byte[]
            response = Utils.readBodyToBytesIfNecessary(response);

            if (methodInfo.isSynchronous) {
              return response;
            }
            return new ResponseWrapper(response, response);
          }

          TypedInput body = response.getBody();
          if (body == null) {
            return new ResponseWrapper(response, null);
          }

          ExceptionCatchingTypedInput wrapped = new ExceptionCatchingTypedInput(body);
          try {
            Object convert = converter.fromBody(wrapped, type);
            if (methodInfo.isSynchronous) {
              return convert;
            }
            return new ResponseWrapper(response, convert);
          } catch (ConversionException e) {
            // If the underlying input stream threw an exception, propagate that rather than
            // indicating that it was a conversion exception.
            if (wrapped.threwException()) {
              throw wrapped.getThrownException();
            }

            // The response body was partially read by the converter. Replace it with null.
            response = Utils.replaceResponseBody(response, null);

            throw RetrofitError.conversionError(url, response, converter, type, e);
          }
        }

        response = Utils.readBodyToBytesIfNecessary(response);
        throw RetrofitError.httpError(url, response, converter, type);
      } catch (RetrofitError e) {
        throw e; // Pass through our own errors.
      } catch (IOException e) {
        if (logLevel.log()) {
          logException(e, url);
        }
        throw RetrofitError.networkError(url, e);
      } catch (Throwable t) {
        if (logLevel.log()) {
          logException(t, url);
        }
        throw RetrofitError.unexpectedError(url, t);
      } finally {
        if (!methodInfo.isSynchronous) {
          Thread.currentThread().setName(IDLE_THREAD_NAME);
        }
      }
    }
  }
</pre>
下面我们分几点来介绍这段源码：
1、methodInfo.init(),解析注解并缓存，包括方法注解（GET,POST,DELETE,PUT,PATCH,HEAD,FormUrlEncoded,Multipart,Headers）和参数注解（Path,Query,QueryMap,EncodedPath,
EncodedQuery,EncodedQueryMap,Header,Body,Field,FieldMap,Part）
<pre>
        synchronized void init() {
		if (loaded)
			return;

		parseMethodAnnotations();
		parseParameters();

		loaded = true;
	}
</pre>
2、执行请求之前的拦截器,可以添加请求头、路径参数和url参数。
<pre>
    requestInterceptor.intercept(requestBuilder);
</pre>
3、使用RequestBuilder组装请求，包括url的拼接，请求参数和请求头的添加，请求数据的转化等，生成一个request对象
4、打印请求log，Retrofit分为四个级别，每个级别打印的log都比上一个级别多
<pre>
/** Controls the level of logging. */
	public enum LogLevel {
		/** No logging. */
		NONE,//不打印log
		/** Log only the request method and URL and the response status code and execution time. */
		BASIC,//只打印请求方法，url，响应码和执行时间
		/** Log the basic information along with request and response headers. */
		HEADERS,//打印请求和响应头相关的基本信息
		/**
		 * Log the headers, body, and metadata for both requests and responses.
		 * <p>
		 * Note: This requires that the entire request and response body be buffered in memory!
		 */
		FULL;//打印整个请求和响应过程中的所有信息

		public boolean log() {
			return this != NONE;
		}
	}
</pre>
5、Profiler请求分析器,在请求执行之前会调用beforeCall()，返回一个自定义对象，请求结束后会调用afterCall()，同时会把请求信息，时间，响应码和自定义对象返回，可以用来实现记录api请求时间的功能。
<pre>
public interface Profiler<T> {
     T beforeCall();
     void afterCall(RequestInformation requestInfo, long elapsedTime, int statusCode,
        T beforeCallData);
}
</pre>
6、执行http请求，具体使用哪种方式来发起http请求将由使用者来决定，目前内置有ApacheClient、UrlConnectionClient和OKClient,也可以自己实现Client接口来实现自定义的Client
<pre>
   Response response = clientProvider.get().execute(request);
</pre>
<pre>
/**
 * Abstraction of an HTTP client which can execute {@link Request Requests}. This class must be
 * thread-safe as invocation may happen from multiple threads simultaneously.
 */
public interface Client {
  /**
   * Synchronously execute an HTTP represented by {@code request} and encapsulate all response data
   * into a {@link Response} instance.
   */
  Response execute(Request request) throws IOException;

  /**
   * Deferred means of obtaining a {@link Client}. For asynchronous requests this will always be
   * called on a background thread.
   */
  interface Provider {
    /** Obtain an HTTP client. Called once for each request. */
    Client get();
  }
}
</pre>
7、在http返回2xx的时候根据responseObjectType进行数据组装，例如当用户声明的方法返回是一个自定义对象时，会使用用户设置的Converter来将返回的数据转化为用户的自定义对象
<pre>
/**
 * Arbiter for converting objects to and from their representation in HTTP.
 *
 * @author Jake Wharton (jw@squareup.com)
 */
public interface Converter {
  /**
   * Convert an HTTP response body to a concrete object of the specified type.
   *
   * @param body HTTP response body.
   * @param type Target object type.
   * @return Instance of {@code type} which will be cast by the caller.
   * @throws ConversionException if conversion was unable to complete. This will trigger a call to
   * {@link retrofit.Callback#failure(retrofit.RetrofitError)} or throw a
   * {@link retrofit.RetrofitError}. The exception message should report all necessary information
   * about its cause as the response body will be set to {@code null}.
   */
  Object fromBody(TypedInput body, Type type) throws ConversionException;

  /**
   * Convert and object to an appropriate representation for HTTP transport.
   *
   * @param object Object instance to convert.
   * @return Representation of the specified object as bytes.
   */
  TypedOutput toBody(Object object);
}
</pre>
Retrofit框架会使用toBody将一个用户自定义对象写入http请求的body中，同时会使用fromBody将http请求返回的数据转化为用户的自定义对象。上面的TypedInput和TypedOutput都是接口，分别表示指定类型的输入和输出。它们的共同实现类有TypedByteArray、TypedFile和TypedString，TypedOutput还有两个特有的实现类FormUrlEncodedTypedOutput和MultipartTypedOutput，表示表单和多重部分数据输出。
8、组装并抛出异常给调用者处理，比如数据转化异常ConversionException，http请求异常（非2xx返回）等。

<strong>上面的分析基本上已经涵盖了Retrofit源码所有核心点，下面对Retrofit框架的设计特点做一个总结：</strong>
  1、面向接口编程，这个是面向对象程序设计的核心思想之一，尤其是对于这种框架级的程序设计，它直接决定了框架的灵活性和可扩展性
  2、拦截器模式，可以使用RequestInterceptor、Profile来对每个请求进行拦截，对请求的控制更为灵活
  3、声明式的api调用，可以使用更少、更易理解的代码来完成http请求，完全实现了请求代码和业务代码的分离，降低了出错的概率。
主要是以上3点，其它的还有可以自定义http请求的底层框架，自定义请求数据和返回数据的解析，同步异步的请求方式，自定义执行请求的Executor，分发结果的Executor，完善的请求log打印机制以及灵活统一的错误处理机制。

<strong>使用下来感觉可以改进的地方：</strong>
1、拦截器不可以对请求进行控制，未附带请求信息，这样直接导致无法实现对请求的签名
2、不支持显示上传和下载请求的进度，不支持断点续传。
3、不可以针对单个请求配置连接和超时时间
4、不可以在接口中直接配置server的base url
5、不支持客户端缓存
