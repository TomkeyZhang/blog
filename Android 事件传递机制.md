最近在开发房价图的时候正好碰到事件的传递问题，以前也没有系统的研究过，只知道个大概，这次正好研究下，做一个总结。
Android的事件机制用一句话来说就是：<strong>事件的分发是自上而下的，而事件的处理是自下而上的，整个事件的传递是U型的</strong>。
ViewGroup的touch事件的<a href="http://developer.android.com/intl/zh-cn/training/gestures/viewgroup.html" target="_blank">官方说明</a>。
关于touch事件详细处理过程和源码分析请看<a href="http://www.infoq.com/cn/articles/android-event-delivery-mechanism" target="_blank">这里</a>。
在这里来分享一下我对Android的事件处理机制的理解，<!--more-->先上图：<a href="http://bcs.duapp.com/myblog-wrodpress//blog/201404//android_event2.png"><img class="alignnone size-full wp-image-85" alt="android_event2" src="http://bcs.duapp.com/myblog-wrodpress//blog/201404//android_event2.png" width="1310" height="530" /></a>

Android会在三个级别处理事件：
<strong>1、Activity（上图中白色表示）</strong>
public boolean dispatchTouchEvent(MotionEvent ev)
public boolean onTouchEvent(MotionEvent event)
<strong>2、ViewGroup（上图中淡蓝色表示）</strong>
public boolean dispatchTouchEvent(MotionEvent ev)
public boolean onInterceptTouchEvent(MotionEvent ev)
public boolean onTouchEvent(MotionEvent event)
<strong>3、View（上图中淡红色表示）</strong>
public boolean dispatchTouchEvent(MotionEvent ev)
public boolean onTouchEvent(MotionEvent event)

其中dispatchTouchEvent是负责分发事件，我们一般无需实现；onInterceptTouchEvent是对事件进行拦截，它会改变事件的传递方向；onTouchEvent是具体对事件的处理。
从上面的逻辑处理图可以看到，我们触摸屏幕的时候，首先由Activity的dispatchTouchEvent来对事件进行分发，它会把事件分发给相应的ViewGroup，由ViewGroup的dispatchTouchEvent来对事件进行再分发，分发的时候会调用onInterceptTouchEvent方法来决定是否要对事件进行拦截，如果此方法返回true的话就不再向下传递事件，转而执行自身（ViewGroup）的onTouchEvent方法；否则执行子View或者子ViewGroup的dispatchTouchEvent方法，如果是ViewGroup的话会重复上述ViewGroup的处理过程，如果是View的话，会执行View的onTouchEvent方法，然后检查View的dispatchTouchEvent的返回，true的话表示事件处理结束，fasle会继续向上传递到父ViewGroup的onTouchEvent，ViewGroup的dispatchTouchEvent返回true，事件就会直接结束；返回false继续向上传递到Activity的onTouchEvent，执行完后事件就结束啦。

总结及注意点：
<ul>
	<li>dispatchTouchEvent方法的返回会影响上一级组件的onTouchEvent方法的执行（上一级的响应）。</li>
	<li>onInterceptTouchEvent方法的返回会直接决定下一级组的dispatchTouchEvent方法的执行（下一级的分发）。</li>
	<li>onTouchEvent负责响应touch事件，我们的业务代码常常写在此处。</li>
	<li>touch事件的分发是从外到内的，而touch事件的响应是从内到外的</li>
	<li>如果View想要接收后续事件，收到ACTION_DOWN事件时，dispatchTouchEvent一定都要返回true，默认实现中dispatchTouchEvent是跟onTouchEvent相关的，所以我们需要在onTouchEvent方法中返回true，告诉系统我处理了当前事件，同时希望接收后续事件。</li>
	<li>收到ACTION_DOWN事件后，如果不希望后续事件被父ViewGroup拦截（即执行onInterceptTouchEvent），可以调getParent().requestDisallowInterceptTouchEvent(true)禁止父ViewGroup执行拦截方法，具体说明可以参考<a href="http://developer.android.com/intl/zh-cn/training/gestures/viewgroup.html" target="_blank">官方文档</a></li>
</ul>
上面的分析是基于一个demo的，demo的github<span style="font-size: 1rem;"><a href="https://github.com/TomkeyZhang/TouchDemo.git" target="_blank">下载</a></span><span style="line-height: 1.714285714; font-size: 1rem;"><a href="https://github.com/TomkeyZhang/TouchDemo.git" target="_blank">地址</a>，大家可以在demo里面自行修改TouchEvent方法的返回，观察结果。</span>

<strong>参考文献：</strong>
<a href="http://ming-fanglin.iteye.com/blog/1396723" target="_blank">http://ming-fanglin.iteye.com/blog/1396723</a>
<a href="http://orgcent.com/android-touch-event-mechanism/" target="_blank">http://orgcent.com/android-touch-event-mechanism/</a>
<a href="http://developer.android.com/intl/zh-cn/training/gestures/viewgroup.html" target="_blank">http://developer.android.com/intl/zh-cn/training/gestures/viewgroup.html</a>
<a href="http://www.infoq.com/cn/articles/android-event-delivery-mechanism" target="_blank">http://www.infoq.com/cn/articles/android-event-delivery-mechanism</a>
<a href="http://www.oschina.net/question/565065_72840" target="_blank">http://www.oschina.net/question/565065_72840</a>
