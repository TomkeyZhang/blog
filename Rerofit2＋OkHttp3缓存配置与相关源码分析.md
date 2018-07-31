相信大家在开发中经常会遇到一些情况（比如：接口因为逻辑复杂或者数据量特别大导致响应很慢，同时返回的数据又在一段时间内不变）需要缓存http请求的数据。http协议很好的支持了缓存的定义，其中最关键的配置就是header中的Cache-Control字段。很多http的客户端都会支持缓存，下面咱们先一起来看看Retrofit2+OkHttp3是如何配置缓存。

一、Retrofit2+OkHttp3缓存配置示例

1、设置request的Cache-Control，配置缓存时间为1小时
<code>@Headers("Cache-Control: public, max-age=3600")
@GET("supplier/chargeArea")
Call getCharge(@Query("supplierStatus") int supplierStatus,
@Query("supplierCategory") int supplierCategory);
</code>
2、定义拦截器，设置response的Cache-Control
<code>public class ResponseCacheInterceptor implements Interceptor {</code>

&nbsp;

<code> @Override
public Response intercept(Chain chain) throws IOException {
Request request = chain.request();
String cacheControl=request.cacheControl().toString();
Response originalResponse = chain.proceed(request);
if(cacheControl.contains("public")){//需要进行/使用缓存
return originalResponse.newBuilder()
.removeHeader("Pragma")// 清除头信息，一些服务器会返回一些干扰信息，不清除下面可能无法生效
.removeHeader("Cache-Control")
.header("Cache-Control", cacheControl)
.build();
}else{//不进行缓存
return originalResponse.newBuilder().header("Cache-Control", "no-store").build();
}
}
}</code>
3、设置拦截器
<code>new OkHttpClient.Builder().addInterceptor(ResponseCacheInterceptor())</code>

这样咱们就实现了接口数据缓存1小时的功能。

二、原理分析

1、OkHttp3是如何发送http请求的？
