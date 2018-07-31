需求：根据N个点p1(x1,y1),p2(x2,y2),...,pn(xN,yN)绘制一条光滑曲线？

经过了解，发现贝塞尔曲线可以满足要求。<span style="color: #333333;">贝塞尔曲线是由法国工程师皮埃尔·贝塞尔（Pierre Bézier）于1962所广泛发表，他运用贝塞尔曲线来为汽车的主体进行设计。</span>

详细介绍请看<a href="http://zh.wikipedia.org/wiki/%E8%B2%9D%E8%8C%B2%E6%9B%B2%E7%B7%9A">这里</a>。

<!--more-->
<h3>一、贝塞尔曲线的公式</h3>
<h5>线性贝塞尔曲线：</h5>
<img src="http://upload.wikimedia.org/math/0/5/c/05c4210c69ffb1358ceb8eb83a1a06fe.png" alt="线性贝塞尔曲线" width="452" height="21" />

<img class="alignnone" src="http://upload.wikimedia.org/wikipedia/commons/thumb/0/00/B%C3%A9zier_1_big.gif/240px-B%C3%A9zier_1_big.gif" alt="" width="240" height="100" />
<h5><span id=".E4.BA.8C.E6.AC.A1.E6.96.B9.E8.B2.9D.E8.8C.B2.E6.9B.B2.E7.B7.9A">二次贝塞尔曲线：</span></h5>
<img class="alignnone" src="http://upload.wikimedia.org/math/8/a/d/8adc5cc34ea9649d6e546043fce9c407.png" alt="" width="422" height="23" />

<img class="alignnone" src="http://upload.wikimedia.org/wikipedia/commons/thumb/3/3d/B%C3%A9zier_2_big.gif/240px-B%C3%A9zier_2_big.gif" alt="" width="240" height="100" />
<h5><span id=".E4.B8.89.E6.AC.A1.E6.96.B9.E8.B2.9D.E8.8C.B2.E6.9B.B2.E7.B7.9A">三次贝塞尔曲线：</span></h5>
<img class="alignnone" src="http://upload.wikimedia.org/math/5/9/7/597ecc5022fa7ab65509d5edfa9c148c.png" alt="" width="558" height="23" />

<img class="alignnone" src="http://upload.wikimedia.org/wikipedia/commons/thumb/d/db/B%C3%A9zier_3_big.gif/240px-B%C3%A9zier_3_big.gif" alt="" width="240" height="100" />

解决方案：使用贝塞尔三次曲线函数每四个点绘制一条光滑曲线，然后把他们接起来，那么现在的主要问题是如何使两条贝塞尔曲线连接处是光滑的，如果我们想在不增加新的点的情况下来绘制出一整条光滑的曲线肯定是行不通的，因为大部分情况下两条贝塞尔曲线的连接处都是不平滑的，那么如何解决这个问题，请看下面的定理。
<h4>二、一个定理及简单证明</h4>
定理：设三次贝塞尔曲线的四个点为P0,P1,P2,P3（其中P0和P3称为端点，P1和P2称为控制点），则P0和P1的连线是P0的切线，P3和P2的连线是P3的切线。
<p style="text-align: center;"><a href="http://bcs.duapp.com/myblog-wrodpress//blog/201405//bessel2.png"><img class="alignnone size-full wp-image-94" src="http://bcs.duapp.com/myblog-wrodpress//blog/201405//bessel2.png" alt="bessel2" width="1172" height="537" /></a>图一</p>
&nbsp;

<img class="alignnone" src="https://raw.githubusercontent.com/TomkeyZhang/BesselChart/master/2014-05-08_15-08-03.png" alt="" width="824" height="551" />
<h4>三、贝塞尔控制点的选取</h4>
通过上面的定理可知，如果我们想要保证每个数据点处是光滑的，必须数据点和左右控制点共线。下面很关键的问题就是如何选取控制点（P3n+1和P3n+2，0&lt;n&lt;N），我们先把所有的数据点（端点，P3n）分为两类点，一类是极值点（P0，P6），另一类是非极值点（P3），同时我们定义一个光滑变量smoothness，0&lt;smoothness&lt;0.5，smoothness=0表示曲线是一条折线，smoothness越大数据点两侧越平坦，一般取0.33。

1、对于极值点P3n（x3n,y3n），可以计算出它的前后控制点P3n-1(x3n-1,y3n-1)，P3n+1(x3n+1,y3n+1).

x3n-1=x3n-(x3n-x3(n-1))*smoothness;

y3n-1=y3n;

x3n+1=x3n+(x3(n+1)-x3n))*smoothness;

y3n+1=y3n;

2、对于非极值点P3n（x3n,y3n），可以计算出它的前后控制点P3n-1(x3n-1,y3n-1)，P3n+1(x3n+1,y3n+1).令

k=(y3(n+1)-y3(n-1))/(x3(n+1)-x3(n-1))

b=y3n-k*x3n

则有：

x3n-1=x3n - (x3n - (y3(n-1) - b) / k) * smoothness;

y3n-1=k * (x3n-1) + b;

x3n+1=x3n + (x3(n+1) - x3n) * smoothness;

y3n+1=k * (x3n+1) + b;

上面的公式目的就是为了计算下图中的P2点和P4点的坐标

<a href="http://bcs.duapp.com/myblog-wrodpress//blog/201405//bessel3.png"><img class="alignnone size-full wp-image-95" src="http://bcs.duapp.com/myblog-wrodpress//blog/201405//bessel3.png" alt="bessel3" width="1032" height="640" /></a>
<p style="text-align: center;">图二</p>
然后使用三次贝塞尔函数按顺序绘制出这3N+1个点即可，结果如下

<img class="alignnone" src="https://raw.githubusercontent.com/TomkeyZhang/BesselChart/master/device-2014-05-08-171406.png" alt="" width="1080" height="1800" />

这样得出的贝塞尔可以保证每个数据点（端点）的左导数和右导数是相等的，同时可以保证任意两个数据点Pn(xn,yn)和Pn+1(xn+1,yn+1)之间的光滑曲线是单调的。也就是说Pn是平滑的变化到Pn+1的，它们之间不会有极值存在。
<h4>四、开源图库aChartEngine的光滑曲线</h4>
上面的解决方案是改进自开源图表aChartEngine，使用aChartEngine也可以绘制出光滑的曲线，如下图所示：

<img class="alignnone" src="https://raw.githubusercontent.com/TomkeyZhang/BesselChart/master/device-2014-05-08-112427.png" alt="" width="1080" height="1800" />

图中的红色点是实际数据点，蓝点是计算出来得贝塞尔点，按照aChartEngine的算法，中间的数据点都是作为控制点，这样的话，一般情况下整个曲线只会经过数据点的第一个和最后一个，这样做出来的光滑曲线图偏差较大。
<h3>使用贝塞尔函数来绘制光滑曲线图的注意事项：</h3>
1、要保证曲线经过每个数据点

2、两个相邻数据点之间的曲线是平滑的，没有上凸和下凹（即没有极值点）

3、两条贝塞尔曲线的连接处的左右导数必须相等

本文主要讲的是原理，下一遍文章会讲解如何从零绘制出一个经过每个点的带滚动的光滑曲线。
