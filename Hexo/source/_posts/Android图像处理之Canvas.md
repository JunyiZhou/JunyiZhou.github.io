---
title: Android图像处理之Canvas
date: 2016-03-09 16:32:07
tags: [Android,Canvas]
---
> 上一篇我们了解了View生命周期的相关方法和调用流程，其中最重要的onDraw方法是自定义View的重要基石。而其中的Canvas参数则至关重要。

Android官方源码中对Canvas的描述是：“Canvas类容纳所有和Draw（绘制）相关方法。为了去Draw些东西，你需要具备4个基础要素：1个Bitmap用来承载像素信息，1个Canvas用来管理Draw相关方法（写入Bitmap中），1个绘图基元（例如，Rect，Path，text，Bitmap），1个画笔（用于描绘图像的颜色和风格）。

这和我们日常理解的绘画异曲同工，Bitmap作为画布，Canvas管理着绘画的手法，绘图基元代表着要绘制的目标，Paint就是你手里的画笔和颜料。

### 如何得到1个Canvas对象
1. 之前提到的onDraw方法的入口参数就是Canvas，我们用变量承载它，就可以使用，而我们操作这个Canvas最终的效果会直接反应在这个View上。
2. SurfaceView是View的继承了，其内部有专门的线程来完成画图的工作，而不用像View一样需要等待刷新，主要用在游戏和高品质动画方面。既然是View的继承类，SurfaceView自然可以获取到Canvas的对象，方式是通过调用SurfaceView的好基友SurfaceHolder的lockCanvas()方法。
3. 当然，除了获取View或SurfaceView自带的现成的Canvas对象，我们还可以自己创建。从4大基本要素我们就可以知道，1个Canvas对象一定要结合1个Bitmap对象。所以一定要为新建的Canvas对象设置1个Bitmap对象。
```javascript
/**
 * 得到一个Bitmap对象，当然也可以使用别的方式得到。但是要注意，改bitmap一定要是mutable(异变的)
 * mutable : 易变的，不定的
 * mutable 作用 ：  控制bitmap的setPixel方法能否使用，也就是外界能否修改bitmap的像素。
 * Bitmap.createBitmap(mWidth, mHeight, Config.ARGB_8888) 为 mutable 为true
 * BimapFactory.decodeResource() 得到的mutable 为false, 要想其为true
 * 一般会BimtapFactory.decodeResource().copy(configu_argb_8888, true);
 * 先new一个Canvas对象，在调用setBitmap方法，一样的效果
 * Canvas c = new Canvas();
 * c.setBitmap(b);
 */
Bitmap b = Bitmap.createBitmap(100, 100, Bitmap.Config.ARGB_8888);
Canvas canvas2 = new Canvas(b);
```
### Canvas能画些什么
Canvas类提供了若干draw……的方法，从字面上我们就可以知道用这些方法我们可以绘制哪些东西。
#### 填充
Canvas内部维持了一个mutable Bitmap，所以我们可以用颜色来填充整个Bitmap。而填充的范围受限于clip的范围。

* drawRGB(int r, int g, int b)：r-红色要素（0~255），g-绿色要素（0~255），b-蓝色要素（0~255）。
* drawARGB(int a, int r, int g, int b)：a-透明度（0~255），r-红色要素（0~255），g-绿色要素（0~255），b-蓝色要素（0~255）。
* drawColor(int color)：这里的color要为16进制的颜色号，例如：0xFF000000，共8位，两位一组从左到右分别为ARGB，具体含义同上。若要使用“#FF000000”这样的色号，可以调用Color.parseColor(String colorString)方法来进行转换。
* drawColor(int color, PorterDuff.Mode mode)：color同上，PorterDuff.Mode比较有意思，有很多种模式，每一种模式都会有特定的效果，感兴趣的朋友可以自行了解一下。
* drawPaint(Paint paint)：Canvas同样可以用画笔来填充Bitmap，当然也受限于clip的范围。
#### 绘制图形
* canvas.drawArc （扇形）
* canvas.drawCircle（圆）
* canvas.drawOval（椭圆）
* canvas.drawLine（线）
* canvas.drawPoint（点）
* canvas.drawRect（矩形）
* canvas.drawRoundRect（圆角矩形）
* canvas.drawVertices（顶点）
* cnavas.drawPath（路径）
#### 绘制图片
* canvas.drawBitmap（位图）
* canvas.drawPicture（图片）
#### 文本
* canvas.drawText
### Canvas的变换
Canvas不仅仅可以draw一些图形、图片，其本身也提供了可操作的方法：rorate(旋转)、scale(压缩)、translate(平移)、skew(扭曲)等。
```javascrpit
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    canvas.drawRect(new Rect(0, 0, 200, 200), new Paint());
    canvas.scale(0.5f, 0.5f);//缩放了
    canvas.drawRect(new Rect(400, 400, 600, 600), new Paint());

    canvas.translate(600, 600);//平移了
    canvas.rotate(45);//旋转了
    canvas.drawRect(new Rect(0, 0, 200, 200), new Paint());

    canvas.translate(200, 200);
    canvas.skew(.5f, .5f);//扭曲了
    canvas.drawRect(new Rect(0, 0, 200, 200), new Paint());
}
```
![](/images/blog-20160309-1.png)
### Canvas的保存和回滚
为了配合Canvas本身的变换操作，Canvas提供了保持当前状态和回滚的方法。来个小例子。
我们准备画一个表盘，那么我们就有两种实现的思路：
1. 表盘上有60个刻度，每个刻度之间间隔6°，这是有规律的，那么我们就可以利用三角函数的知识来把刻度的两个坐标求出来，再利用drawLine画到Canvas上。考验你逻辑思维和数学功底的时候到了，不用仔细想就能知道这个方法有点麻烦。
2. 如果不喜欢第一种方法，那么我们可以尝试一下这一种。Canvas提供了旋转操作的方法，也提供了保存和回复状态的方法，那么我们就用这些搭配起来。首先我在（100，0）和（100，10）两个坐标之间画一条竖线，很简单，整点的刻度就出来了。之后先将Canvas的状态保存起来，在（100，0）和（100，10）两个坐标之间画一条竖线，回复Canvas的状态，一分钟的刻度就出来了。以此类推，思路是不是很简单。
```javascript
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    for (int i = 0; i < 360; i = i + 6) {
        canvas.save();
        canvas.rotate(i, 100, 100);
        canvas.drawLine(100, 0, 100, 10, new Paint());
        canvas.restore();
    }
}
```
![](/images/blog-20160309-2.png)
### 总结
Canvas的大体情况就是这样，如果只是需要用Canvas来实现自定义View的话，那么这些内容应该可以帮到你。当然，还有相当一部分的方法和属性我没有提到，感兴趣的同学可以深入研究一下Canvas。
下一篇我们一起研究一下自定义控件中比较重要的Matrix（矩阵）类。