上一篇主要介绍了绘制经过每个点的光滑曲线的原理，本文会重点介绍一下在Android中如何从零开始使用贝塞尔方法编写一个光滑曲线图控件。程序的设计图如下：

<a href="http://bcs.duapp.com/myblog-wrodpress//blog/201405//bessel_chart.png"><img class="alignnone size-full wp-image-100" alt="bessel_chart" src="http://bcs.duapp.com/myblog-wrodpress//blog/201405//bessel_chart.png" width="662" height="680" /></a><!--more-->

一、样式控制类ChartStyle
<pre class="lang:java decode:true" title="ChartStyle.java">    /** 网格线颜色 */
    private int gridColor;
    /** 坐标轴分隔线宽度 */
    private int axisLineWidth;

    /** 横坐标文本大小 */
    private float horizontalLabelTextSize;
    /** 横坐标文本颜色 */
    private int horizontalLabelTextColor;
    /** 横坐标标题文本大小 */
    private float horizontalTitleTextSize;
    /** 横坐标标题文本颜色 */
    private int horizontalTitleTextColor;
    /** 横坐标标题文本左间距 */
    private int horizontalTitlePaddingLeft;
    /** 横坐标标题文本右间距 */
    private int horizontalTitlePaddingRight;

    /** 纵坐标文本大小 */
    private float verticalLabelTextSize;
    /** 纵坐标文本上下间距 */
    private int verticalLabelTextPadding;
    /** 纵坐标文本左右间距相对文本的比例 */
    private float verticalLabelTextPaddingRate;
    /** 纵坐标文本颜色 */
    private int verticalLabelTextColor;</pre>
二、基础数据集合ChartData
<pre class="lang:java decode:true">    private Marker marker;
    private List&lt;Series&gt; seriesList;
    private List&lt;Label&gt; xLabels;
    private List&lt;Label&gt; yLabels;
    private List&lt;Title&gt; titles;
    private int maxValueY;
    private int minValueY;
    private int maxPointsCount;
    private LabelTransform labelTransform;
    /** 纵坐标显示文本的数量 */
    private int yLabelCount;

    /** 使用哪一个series的横坐标来显示横坐标文本 */
    private int xLabelUsageSeries;
    public interface LabelTransform {
        /** 纵坐标显示的文本 */
        String verticalTransform(int valueY);

        /** 横坐标显示的文本 */
        String horizontalTransform(int valueX);
        /** 是否显示指定位置的横坐标文本 */
        boolean labelDrawing(int valueX);

    }</pre>
2.1、坐标轴标签Label
<pre class="lang:java decode:true">       /**文本对应的坐标X*/
        public float x;
        /**文本对应的坐标Y*/
        public float y;
        /** 文本对应的绘制坐标Y */
        public float drawingY;
        /**文本对应的实际数值*/
        public int value;
        /**文本*/
        public String text;</pre>
2.2、时间序列Series
<pre class="lang:java decode:true" title="时间序列集合">    /** 序列曲线的标题 */
    private Title title;
    /** 序列曲线的颜色 */
    private int color;
    /** 序列点集合 */
    private List&lt;Point&gt; points;
    /** 贝塞尔曲线点 */
    private List&lt;Point&gt; besselPoints;</pre>
2.3、横向标题Title
<pre class="lang:java decode:true">    /**文本对应的坐标X*/
    public float textX;
    /**文本对应的坐标Y*/
    public float textY;
    /**文本*/
    public String text;
    /**圆点对应的坐标X*/
    public float circleX;
    /**圆点对应的坐标Y*/
    public float circleY;
    /**颜色*/
    public int color;
    /**圆点的半径*/
    public int radius;
    /**图形标注与文本的间距*/
    public int circleTextPadding;
    /**文本区域*/
    public Rect textRect=new Rect();</pre>
2.4、数据结点Point
<pre class="lang:java decode:true">    /**是否在图形中绘制出此结点*/
    public boolean willDrawing;
    /** 在canvas中的X坐标 */
    public float x;
    /** 在canvas中的Y坐标 */
    public float y;
    /** 实际的X数值 */
    public int valueX;
    /** 实际的Y数值 */
    public int valueY;</pre>
三、光滑曲线图BesselChartView
<pre class="lang:java decode:true">    /** 通用画笔 */
    private Paint paint;
    /** 曲线的路径，用于绘制曲线 */
    private Path curvePath;
    /** 曲线图绘制的计算信息 */
    private BesselCalculator calculator;
    /** 曲线图的样式 */
    private ChartStyle style;
    /** 曲线图的数据 */
    private ChartData data;
    /** 手势解析 */
    private GestureDetector detector;
    /** 是否绘制全部贝塞尔结点 */
    private boolean drawBesselPoint;
    /** 滚动计算器 */
    private Scroller scroller;
    @Override
    protected void onDraw(Canvas canvas) {
        if (data.getSeriesList().size() == 0)
            return;
        calculator.ensureTranslation();
        canvas.translate(calculator.getTranslateX(), 0);
        drawGrid(canvas);
        drawCurveAndPoints(canvas);
        drawMarker(canvas);
        drawHorLabels(canvas);
    }</pre>
四、核心类BesselCalculator
<pre class="lang:java decode:true">    /** 纵坐标文本矩形 */
    public Rect verticalTextRect;
    /** 横坐标文本矩形 */
    public Rect horizontalTextRect;
    /** 横坐标标题文本矩形 */
    public Rect horizontalTitleRect;
    /** 图形的高度 */
    public int height;
    /** 图形的宽度 */
    public int width;
    /** 纵轴的宽度 */
    public int yAxisWidth;
    /** 纵轴的高度 */
    public int yAxisHeight;
    /** 横轴的高度 */
    public int xAxisHeight;
    /** 横轴的标题的高度 */
    public int xTitleHeight;
    /** 横轴的长度 */
    public int xAxisWidth;
    /** 灰色竖线顶点 */
    public Point[] gridPoints;

    /** 画布X轴的平移，用于实现曲线图的滚动效果 */
    private float translateX;

    /** 用于测量文本区域长宽的画笔 */
    private Paint paint;

    private ChartStyle style;
    private ChartData data;
    /** 光滑因子 */
    private float smoothness;
    /**
     * 计算图形绘制的参数信息
     * 
     * @param width 曲线图区域的宽度
     */
    public void compute(int width) {
        this.width = width;
        this.translateX = 0;
        computeVertcalAxisInfo();// 计算纵轴参数
        computeHorizontalAxisInfo();// 计算横轴参数
        computeTitlesInfo();// 计算标题参数
        computeSeriesCoordinate();// 计算纵轴参数
        computeBesselPoints();// 计算贝塞尔结点
        computeGridPoints();// 计算网格顶点
    }</pre>
五、核心代码：

5.1 计算光滑曲线的贝塞尔控制点
<pre class="lang:java decode:true crayon-selected">    /** 计算贝塞尔结点 */
    private void computeBesselPoints() {
        for (Series series : data.getSeriesList()) {
            List&lt;Point&gt; besselPoints = series.getBesselPoints();
            List&lt;Point&gt; points = new ArrayList&lt;Point&gt;();
            for (Point point : series.getPoints()) {
                if (point.valueY &gt; 0)
                    points.add(point);
            }
            int count = points.size();
            if (count &lt; 2)
                continue;
            besselPoints.clear();
            for (int i = 0; i &lt; count; i++) {
                if (i == 0 || i == count - 1) {
                    computeUnMonotonePoints(i, points, besselPoints);
                } else {
                    Point p0 = points.get(i - 1);
                    Point p1 = points.get(i);
                    Point p2 = points.get(i + 1);
                    if ((p1.y - p0.y) * (p1.y - p2.y) &gt;= 0) {// 极值点
                        computeUnMonotonePoints(i, points, besselPoints);
                    } else {
                        computeMonotonePoints(i, points, besselPoints);
                    }
                }
            }
        }
    }

    /** 计算非单调情况的贝塞尔结点 */
    private void computeUnMonotonePoints(int i, List&lt;Point&gt; points, List&lt;Point&gt; besselPoints) {
        if (i == 0) {
            Point p1 = points.get(0);
            Point p2 = points.get(1);
            besselPoints.add(p1);
            besselPoints.add(new Point(p1.x + (p2.x - p1.x) * smoothness, p1.y));
        } else if (i == points.size() - 1) {
            Point p0 = points.get(i - 1);
            Point p1 = points.get(i);
            besselPoints.add(new Point(p1.x - (p1.x - p0.x) * smoothness, p1.y));
            besselPoints.add(p1);
        } else {
            Point p0 = points.get(i - 1);
            Point p1 = points.get(i);
            Point p2 = points.get(i + 1);
            besselPoints.add(new Point(p1.x - (p1.x - p0.x) * smoothness, p1.y));
            besselPoints.add(p1);
            besselPoints.add(new Point(p1.x + (p2.x - p1.x) * smoothness, p1.y));
        }
    }

    /**
     * 计算单调情况的贝塞尔结点
     * 
     * @param i
     * @param points
     * @param besselPoints
     */
    private void computeMonotonePoints(int i, List&lt;Point&gt; points, List&lt;Point&gt; besselPoints) {
        Point p0 = points.get(i - 1);
        Point p1 = points.get(i);
        Point p2 = points.get(i + 1);
        float k = (p2.y - p0.y) / (p2.x - p0.x);
        float b = p1.y - k * p1.x;
        Point p01 = new Point();
        p01.x = p1.x - (p1.x - (p0.y - b) / k) * smoothness;
        p01.y = k * p01.x + b;
        besselPoints.add(p01);
        besselPoints.add(p1);
        Point p11 = new Point();
        p11.x = p1.x + (p2.x - p1.x) * smoothness;
        p11.y = k * p11.x + b;
        besselPoints.add(p11);
    }</pre>
5.2、坐标变换。由于手机屏幕的坐标是朝右下方的，而我们实际显示的时候是朝左上方的，所以需要进行坐标变换，代码：
<pre class="lang:java decode:true">          float ratio = (point.valueY - data.getMinValueY()) / (float) (data.getMaxValueY() - data.getMinValueY());
          point.y = maxCoordinateY - (maxCoordinateY - minCoordinateY) * ratio;</pre>
5.3、实现拖动
<pre class="lang:java decode:true">    @Override
    public boolean onTouchEvent(MotionEvent event) {
        return detector.onTouchEvent(event);
    }</pre>
实现OnGestureListener的OnScroll方法
<pre class="lang:java decode:true">            @Override
            public boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {
                if (Math.abs(distanceX / distanceY) &gt; 1) {
                    getParent().requestDisallowInterceptTouchEvent(true);
                    BesselChartView.this.calculator.move(distanceX);
                    invalidate();
                    return true;
                }
                return false;
            }</pre>
在BesselChartView的onDraw方法中调用如下代码来平移画布实现拖动
<pre class="lang:java decode:true">canvas.translate(calculator.getTranslateX(), 0);</pre>
&nbsp;

5.4、实现滑动

只实现拖动会让人有一种不流畅的感觉，所以还需要实现滑动，考虑到应用要支持api level 8，可以使用Scroller来实现（api level 9以后google推荐使用OverScroller来实现，OverScroller允许滚动超出边界，可以实现回弹效果）， OnGestureListener的onFling和onDown方法：
<pre class="lang:java decode:true">            @Override
            public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY) {
                scroller.fling((int) BesselChartView.this.calculator.getTranslateX(), 0, (int) velocityX, 0, -getWidth(), 0, 0, 0);
                ViewCompat.postInvalidateOnAnimation(BesselChartView.this);
                return true;
            }

            @Override
            public boolean onDown(MotionEvent e) {
                scroller.forceFinished(true);
                ViewCompat.postInvalidateOnAnimation(BesselChartView.this);
                return true;
            }</pre>
获取scroller计算的偏移，同时刷新UI，computeScroll()会在View的onDraw方法之前执行
<pre class="lang:java decode:true">    @Override
    public void computeScroll() {
        if (scroller.computeScrollOffset()) {
            calculator.moveTo(scroller.getCurrX());
            ViewCompat.postInvalidateOnAnimation(this);
        }
    }</pre>
5.5、实现滚动动画
<pre class="lang:java decode:true">scroller.startScroll(0, 0, -calculator.xAxisWidth / 2, 0, 7000);</pre>
六、使用到的绘图相关的api

6.1 Canvas 画布
<pre class="lang:java decode:true">translate(float dx, float dy)
drawLine(float startX, float startY, float stopX, float stopY, Paint paint)
drawBitmap(Bitmap bitmap, Rect src, Rect dst, Paint paint)
drawCircle(float cx, float cy, float radius, Paint paint)
drawPath(Path path, Paint paint)
drawText(String text, float x, float y, Paint paint)</pre>
6.2 Paint 画笔
<pre class="lang:java decode:true">setStyle(Style style)
setStrokeWidth(float width)
setColor(int color)
setTextSize(float textSize)
setTextAlign(Align align)
setAlpha(int a)</pre>
6.3 Path 路径
<pre class="lang:java decode:true">moveTo(float x, float y)
cubicTo(float x1, float y1, float x2, float y2, float x3, float y3)</pre>
七、源代码地址：<a href="https://github.com/TomkeyZhang/BesselChart">https://github.com/TomkeyZhang/BesselChart</a>

八、在安居客android app中的效果图

<a href="http://bcs.duapp.com/myblog-wrodpress//blog/201405//graph.png"><img class="alignnone size-full wp-image-120" alt="graph" src="http://bcs.duapp.com/myblog-wrodpress//blog/201405//graph.png" width="1080" height="1800" /></a>
