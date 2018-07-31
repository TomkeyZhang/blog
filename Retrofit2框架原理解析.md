之前我对<a href="http://tomkeyzhang.duapp.com/?p=122#more-122">Retrofit1框架原理解析</a>做过分析，为了更好的理解本文，如果没有看过建议先去阅读一下。如果对Retrofit2的使用还不了解的话，建议阅读<a href="http://square.github.io/retrofit/">官网说明</a>或者<a href="http://www.jcodecraeer.com/plus/view.php?aid=3460">这篇文章</a>。

首先，我们来看一下源码的结构图：

<img class="alignleft" alt="" src="http://7xozsw.dl1.z0.glb.clouddn.com/retrofit2_str.png" width="1300" height="710" /><!--more-->

根据这张图，咱们以问答的形式来完成对源码的分析：
<ol>
	<li>为什么执行一个接口方法就可以发起一个http请求？<strong>跟retrofit1一样依旧使用java的动态代理生成http请求接口的实现类</strong>，核心代码：
<pre class="lang:java decode:true">public &lt;T&gt; T create(final Class&lt;T&gt; service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class&lt;?&gt;[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall&lt;&gt;(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }</pre>
执行http请求接口方法时，会执行InvocationHandler的invoke方法，并传入相应的metod对象。</li>
	<li>在哪里解析接口方法及其参数的注解的？在ServiceMethod.Build类中通过了如下的方法分别来解析方法注解和参数注解：
<pre class="lang:java decode:true">private void parseMethodAnnotation(Annotation annotation)
private ParameterHandler&lt;?&gt; parseParameterAnnotation(
        int p, Type type, Annotation[] annotations, Annotation annotation)</pre>
解析过程会有各种判断条件，代码比较多，大家可以自行去查看源码。</li>
	<li>如何将请求的参数转换成需要的格式（json，xml等），将响应的数据转换成自定义java对象？Retrofit2使用Convert.Factory转化器工厂生成的转换器来进行数据转换，对于@Part,@PartMap和@Body注解的参数会使用requestBodyConverter把参数转换成okhttp3.RequestBody类型的对象；对于@Path,@Query,@QueryMap,@Header注解的参数会使用stringConverter把参数转换成okhttp3的RequestBody对象，核心代码：
<pre class="lang:java decode:true">public interface Converter&lt;F, T&gt; {
  T convert(F value) throws IOException;

  /** Creates {@link Converter} instances based on a type and target usage. */
  abstract class Factory {
    /**
     * Returns a {@link Converter} for converting an HTTP response body to {@code type}, or null if
     * {@code type} cannot be handled by this factory. This is used to create converters for
     * response types such as {@code SimpleResponse} from a {@code Call&lt;SimpleResponse&gt;}
     * declaration.
     */
    public Converter&lt;ResponseBody, ?&gt; responseBodyConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
      return null;
    }

    /**
     * Returns a {@link Converter} for converting {@code type} to an HTTP request body, or null if
     * {@code type} cannot be handled by this factory. This is used to create converters for types
     * specified by {@link Body @Body}, {@link Part @Part}, and {@link PartMap @PartMap}
     * values.
     */
    public Converter&lt;?, RequestBody&gt; requestBodyConverter(Type type,
        Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
      return null;
    }

    /**
     * Returns a {@link Converter} for converting {@code type} to a {@link String}, or null if
     * {@code type} cannot be handled by this factory. This is used to create converters for types
     * specified by {@link Field @Field}, {@link FieldMap @FieldMap} values,
     * {@link Header @Header}, {@link Path @Path}, {@link Query @Query}, and
     * {@link QueryMap @QueryMap} values.
     */
    public Converter&lt;?, String&gt; stringConverter(Type type, Annotation[] annotations,
        Retrofit retrofit) {
      return null;
    }
  }
}</pre>
</li>
	<li>拿到数据以后是怎么发起一个http请求的？首先，将解析得到的数据组装成okhttp3.Request
<pre class="lang:java decode:true">Request toRequest(Object... args) throws IOException {
    RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
        contentType, hasBody, isFormEncoded, isMultipart);

    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler&lt;Object&gt;[] handlers = (ParameterHandler&lt;Object&gt;[]) parameterHandlers;

    int argumentCount = args != null ? args.length : 0;
    if (argumentCount != handlers.length) {
      throw new IllegalArgumentException("Argument count (" + argumentCount
          + ") doesn't match expected count (" + handlers.length + ")");
    }

    for (int p = 0; p &lt; argumentCount; p++) {
      handlers[p].apply(requestBuilder, args[p]);
    }

    return requestBuilder.build();
  }</pre>
然后，使用调用OkHttpClient.newCall方法组装生成一个okhttp3.Call对象，这个对象可以用来真正的发起http请求，并同时支持同步（execute）和异步方法（enqueue）
<pre class="lang:java decode:true">private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }</pre>
&nbsp;
<pre title="okhttp3.Call的同步和异步请求方法">  Response execute() throws IOException;

  void enqueue(Callback responseCallback);</pre>
&nbsp;</li>
	<li>那么我们声明的http请求的返回参数是不是就是这个Call呢？当然不是，因为这个Call是okhttp3.Call，而我们接口的返回值是retrofit2.Call，完全不是一个东西啊。其实是这样，okhttp3.Call方法的执行委托给了retrofit2.OkHttpCall，进行了包装，执行OkHttpCall的execute()和enqueue(Callback responseCallback)方法
<pre class="lang:java decode:true crayon-selected"> @Override public Response&lt;T&gt; execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      if (creationFailure != null) {
        if (creationFailure instanceof IOException) {
          throw (IOException) creationFailure;
        } else {
          throw (RuntimeException) creationFailure;
        }
      }

      call = rawCall;
      if (call == null) {
        try {
          call = rawCall = createRawCall();
        } catch (IOException | RuntimeException e) {
          creationFailure = e;
          throw e;
        }
      }
    }

    if (canceled) {
      call.cancel();
    }

    return parseResponse(call.execute());
  }

@Override public void enqueue(final Callback&lt;T&gt; callback) {
    if (callback == null) throw new NullPointerException("callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null &amp;&amp; failure == null) {
        try {
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }

    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response&lt;T&gt; response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        callSuccess(response);
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callSuccess(Response&lt;T&gt; response) {
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }</pre>
问题又来了，为什么要这样设计呢？我觉得原因主要是解析响应数据，将okhttp3.Response转换成retrofit2.Response&lt;T&gt;，T为自定义解析后的数据类型；还有就是对异步请求回调异常和错误的捕获，防止程序crash。</li>
	<li>那么这个OkHttpCall对象是不是就是我们接口返回的Call对象？其实也不是，我们知道当调用okhttp3.Call.enqueue方法时，okhttp会在一个子线程中执行此网络请求，执行完后会继续在子线程调用callback相应的方法，但在android中我们一般需要在ui线程执行回调。为了适配类似的需求，retrofit2设计了CallAdapter和CallAdapter.Factory接口，将retrofit2.Call对象转换成我们自己更方便使用的对象，比如使用ExecutorCallAdapterFactory生成的ExecutorCallbackCall，可以在指定的执行器（比如Android平台会使用ui线程执行器）上执行回调。代码：
<pre class="lang:java decode:true">public interface CallAdapter&lt;T&gt; {

  Type responseType();

  &lt;R&gt; T adapt(Call&lt;R&gt; call);

  abstract class Factory {

    public abstract CallAdapter&lt;?&gt; get(Type returnType, Annotation[] annotations,
        Retrofit retrofit);
  }
}

final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
  final Executor callbackExecutor;

  ExecutorCallAdapterFactory(Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

  @Override
  public CallAdapter&lt;Call&lt;?&gt;&gt; get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter&lt;Call&lt;?&gt;&gt;() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public &lt;R&gt; Call&lt;R&gt; adapt(Call&lt;R&gt; call) {
        return new ExecutorCallbackCall&lt;&gt;(callbackExecutor, call);
      }
    };
  }

  static final class ExecutorCallbackCall&lt;T&gt; implements Call&lt;T&gt; {
    final Executor callbackExecutor;
    final Call&lt;T&gt; delegate;

    ExecutorCallbackCall(Executor callbackExecutor, Call&lt;T&gt; delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }

    @Override public void enqueue(final Callback&lt;T&gt; callback) {
      if (callback == null) throw new NullPointerException("callback == null");

      delegate.enqueue(new Callback&lt;T&gt;() {
        @Override public void onResponse(Call&lt;T&gt; call, final Response&lt;T&gt; response) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override public void onFailure(Call&lt;T&gt; call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }

    @Override public boolean isExecuted() {
      return delegate.isExecuted();
    }

    @Override public Response&lt;T&gt; execute() throws IOException {
      return delegate.execute();
    }

    @Override public void cancel() {
      delegate.cancel();
    }

    @Override public boolean isCanceled() {
      return delegate.isCanceled();
    }

    @Override public Call&lt;T&gt; clone() {
      return new ExecutorCallbackCall&lt;&gt;(callbackExecutor, delegate.clone());
    }

    @Override public Request request() {
      return delegate.request();
    }
  }
}</pre>
终于我们知道了我们在请求的接口方法上面声明的retrofit2.Call类型对应的实例其实是ExecutorCallbackCall的对象。根据这个设计，我们完全可以自定义一个包含自己app api公共逻辑的CustomCall，并实现一个CustomCallAdapterFactory和CustomCallAdapter。</li>
</ol>
到此，核心的代码分析就结束了，再回过头来看看这张程序结构图：

<img class="alignleft" alt="" src="http://7xozsw.dl1.z0.glb.clouddn.com/retrofit2_str.png" width="1300" height="635" />

<strong>我梳理一下retrofit2用到的设计模式：</strong>
<ol>
	<li>工厂模式</li>
	<li>Builder模式</li>
	<li>代理模式（动态代理&amp;静态代理）</li>
	<li>“组装器（转换器）”模式</li>
	<li>适配器模式</li>
</ol>
&nbsp;
