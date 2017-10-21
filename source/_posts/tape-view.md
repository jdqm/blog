---
title: 带你实现漂亮的滑动卷尺
date: 2017-10-21 12:30:07
updated: 2017-10-21 12:30:07
tags:
    -自定义View
categories: Android
---
HenCoder最近在搞一个仿写活动，[活动地址 http://hencoder.com/activity-mock-1/][1]，之前关注过他写的关于绘制系列的文章，今天就拿这个来练练手，我选择模仿的是荷健康的滑动卷尺效果。
<!-- more -->
![荷健康滑动卷](http://upload-images.jianshu.io/upload_images/3631399-28239e8d2b4841f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

视觉设计师丢下了一张图，然后就潇洒地去喝茶了...

留下孤苦伶仃的你，这个时候旁边飘来了那英的声音：你永远不懂我伤悲，像白天不懂夜的黑...

瞬间有一种爱上那英的感觉。

# 一、分析分析

言归正传，这个View看起来有两部分，上面的有两行字体，下面是一把尺子，从通用性角度来讲，上面的两行字应该由使用者实现（自由度更高），也就是说它不属于这个View，尺子只需要提供一个回调接口，让外部拿到它当前的值即可。

虽然这把尺子看起来比较简单，但是却蕴含这很多细节，到实现的时候你就会发现了。一般地，拿到一个View的设计稿，我们要去分解里面的元素，然后选择合适的技术来实现。下面就把这个View搬到解刨台：

1、背景，可以看到是纯色，所以直接画一个颜色即可，事实上可以支持任意的drawable;
2、刻度，drawLine；
3、刻度下面的数值，drawText
4、三角形指示器，画几何图形（推荐）或者画一个图片；
5、滑动的时候不断的重新绘制；
6、支持快速滑动，涉及到滑动速度的计算。

# 二、实现实现

自定义View选择扩展哪个现成的类有时候是很关键的，可能起到事半功倍的效果。遗憾的是，并没有哪个现成的控件与我们的需求比较相似，所以选择了扩展View来实现。
```
TapeView extends View
```

按照前面分析的步骤一步步来实现吧。

## 1、画背景
这个View的背景只是一个简单的颜色，画颜色的api有下面几个
```
canvas.drawColor(bgColor);
canvas.drawRGB(100, 200, 100);  
canvas.drawARGB(100, 100, 200, 100);  
```

## 2、画刻度线
刻度线是这个View的核心，也是难点所在，比如说你如何保证当前值一定是在View的水平中间位置？这就有点类似于几何数学中的证明题，比如说正明两条直线垂直，通常的思考方式是从结果反推。比如这里我们可以先确定View的水平中就是当前值，然后再去反推出其他刻度的位置，这样就能保证当前值一定是在View的中间（这可是要比证明你爸是你爸要简单很多）。

知道了当前值就在水平的中间位置，那么是不是就可以发推出来最左边的第一条刻度线呢？找到第一条刻度线后再顺序往右画出当前可显示的所有刻度即可。怎么找，请看下面这张很丑的图：

![想想是有多久没拿起笔了](http://upload-images.jianshu.io/upload_images/3631399-a2b82b950249d849.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

offset: 最小值到当前值的物理距离（像素）
per: 每一隔代表的数值
gapWidth: 每一隔的物理距离（像素）
minValue: 刻度尺的最小数值
maxValue: 刻度尺的最大数值
middle: View的水平中间位置

从 x=middle-offset+gapWidth 这条表达式可以看到，middle、gapWidth都是我们初始化时指定的，改变的只有offset，所以在滑动时我们要随时改变offset的值，具体来说就是从左往右滑动offset减小，从右往左滑动offset增大，但是要注意offset的有效范围: 0~(maxValue-minValue)/per*gapWidth。具体实现呢还是看[源码][2]吧。

是不是特别像小学数学计算距离的应用题？如果你看不懂，那证明我不做老师是对的，所以不是你的问题。

## 3、画三角形
三角形怎么画？折腾折腾发现canvas有画矩形、画圆等api，但是没有画三角形的api。这就得借助canvas.drawPath来实现（灵感出自你的知识储备），控制好三个点的坐标就行。根据视觉图三角形位置是：顶部，中间。
```
private void drawTriangle(Canvas canvas) {
    paint.setColor(triangleColor);
    Path path = new Path();
    path.moveTo(getWidth() / 2 - triangleHeight / 2, 0);
    path.lineTo(getWidth() / 2, triangleHeight / 2);
    path.lineTo(getWidth() / 2 + triangleHeight / 2, 0);
    path.close();
    canvas.drawPath(path, paint);
}
```
为什么先画刻度线而不是先画三角形？如果是这样的话，刻度线就会在三角形指示器上面，颜色不一样就不太美观了，举个栗子：

![绘制顺序所导致的问题](http://upload-images.jianshu.io/upload_images/3631399-5b6b806d1392a636.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 4、滑动处理

滑动处理实际上就是对触摸事件的处理，根据Android的事件分发机制，当事件传递到我们的View时会回调onTouchEvent方法，所以我们可以在这个方法中处理。
```
@Override
public boolean onTouchEvent(MotionEvent event) {
    float x = event.getX();
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            lastX = x;
            dx = 0;
            break;
        case MotionEvent.ACTION_MOVE:
            //dx标识滑动距离，这种计算方式 左-->右 dx小于0，所以重新计算offset时要加上dx
            dx = lastX - x;
            validateValue();
            break;
        case MotionEvent.ACTION_UP:
            //当滑动结束时如果三角形指示器不是在刻度上，要继续滑动让他们对齐
            smoothMoveToCalibration();
            return false;
        default:
            return false;

    }
    lastX = x;
    return true;
}
```
可以看到，滑动处理也很简单，就是根据滑动距离重新计算offset的值，然后刷新View让其重新绘制（前面我们说了尺子的状态由offset决定）。

这里说一点题外话，上面提到当事件传递到我们的View时会回调onTouchEvent方法，所以前提是事件要能传递到我们的View上，也就是说你不能在父View拦截了事件，或者你给当前View设置了OnTouchListener并且onTouch方法返回了true事件也不会在传递到onTouchEvent,为啥子呢，请看下面的代码:

View#dispatchTouchEvent
```
public boolean dispatchTouchEvent(MotionEvent event) {
    //省略其他代码
    boolean result = false;
    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) { 
            //onTouch返回ture,resutl就被置为true    
            result = true;
        }
    
        //从这里可以看到onTouch返回true之后，就不会再调用onTouchEvent
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    
    ...
}
```
可以看到，事件是先传递给了View的OnTouchListener，如果它的onTouch返回true，就不会再调用onTouchEvent。这一部分内容属于Android的事件分发机制范畴，这里不再深究。


# 5、弹性滑动
实现弹性滑动的方式有很多，比如通过动画、Scroller、通过Handler来实现延时策略等，这里采用的是Scroller，当然速度的计算还得借助VelocityTracker速度追踪器。这里采用Scroller+VelocityTracker来实现。

如何通过VelocityTracker来计算速度？你是否还记得速度的计算公式？ V = S/T（物理知识终于派上用场了）
```
velocityTracker = VelocityTracker.obtain();
velocityTracker.addMovement(event);

//在获取速度之前一定要先传入时间（ms）计算一下
velocityTracker.computeCurrentVelocity(1000);
float xVelocity = velocityTracker.getXVelocity();
```

但是有一点需要注意，Android里的速度计算方式是：速度 = （终点坐标 - 起点坐标）/ 时间。
也就是说当你从右往左滑动时，速度是负的，而我们通常理解的速度都是正的。如果你还记的高中物理的动量守恒定律，在矢量方程中符号可以理解为方向，并非只有正负之分。

速度是有了Scroller是如何实现滑动的？先看看代码：
```
private void calculateVelocity() {
    velocityTracker.computeCurrentVelocity(1000);
    float xVelocity = velocityTracker.getXVelocity(); //计算水平方向的速度

    //大于这个值才会被认为是fling
    if (Math.abs(xVelocity) > minFlingVelocity) {
        scroller.fling(0, 0, (int) xVelocity, 0, Integer.MIN_VALUE, Integer.MAX_VALUE, 0, 0);
        //注意这个invalidate
        invalidate();
    }
}
```
可以看到，当我们拿到水平方向的速度后，调用了scroller.fling()方法，看着好像是它完成了滑动，其实它内部就是根据参数计算出来一些值并赋值给了它的属性：
```
mVelocity = velocity;
mDuration = getSplineFlingDuration(velocity);
mStartTime = AnimationUtils.currentAnimationTimeMillis();
mStartX = startX;
mStartY = startY;
```
真正的滑动是在调用invalidate()方法之后，我们知道invalidate()方法会导致View重新绘制，而View的draw方法调用了computeScroll()方法，而computeScroll()方法在View中的默认实现是空的，所以为了完成滑动，我们需要重写这个方法。

```
@Override
public void computeScroll() {
    super.computeScroll();
    //返回true表示滑动还没有结束
    if (scroller.computeScrollOffset()) {
        if (scroller.getCurrX() == scroller.getFinalX()) {
            smoothMoveToCalibration();
        } else {
            int x = scroller.getCurrX();
            dx = lastX - x;
            //继续让View刷新
            validateValue();
            lastX = x;
        }
    }
}
```

核心就在 scroller.computeScrollOffset()，它会根据前面调用scroller.fling()计算出来的几个值来得到View的下一个位置，如此反复到目标位置为止。可以简单看看computeScrollOffset的FLING_MODE的实现：

```
/**
 * Call this when you want to know the new location.  If it returns true,
 * the animation is not yet finished.
 */ 
public boolean computeScrollOffset() {

    case FLING_MODE:
    
        //看已过去的时间占总时间的比例
        final float t = (float) timePassed / mDuration;
        final int index = (int) (NB_SAMPLES * t);
        float distanceCoef = 1.f;
        float velocityCoef = 0.f;
        if (index < NB_SAMPLES) {
            final float t_inf = (float) index / NB_SAMPLES;
            final float t_sup = (float) (index + 1) / NB_SAMPLES;
            final float d_inf = SPLINE_POSITION[index];
            final float d_sup = SPLINE_POSITION[index + 1];
            velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
            distanceCoef = d_inf + (t - t_inf) * velocityCoef;
        }

        //重新计算了速度
        mCurrVelocity = velocityCoef * mDistance / mDuration * 1000.0f;
        
        mCurrX = mStartX + Math.round(distanceCoef * (mFinalX - mStartX));
        // Pin to mMinX <= mCurrX <= mMaxX
        mCurrX = Math.min(mCurrX, mMaxX);
        mCurrX = Math.max(mCurrX, mMinX);//得到了新的滑动位置
    }
}

```
可以看到，它重新计算了mCurrX（就是这次刷新的位置）和 mCurrVelocity（速度），可见这个速度是越来越小，所以我们看到的fling是一种减速运动。什么？又是物理！坦率讲，应该感到庆幸，你学的物理终于有用武之地了，哈哈。事实上Android动画中用到了很多物理知识，比如说让你做一个自由落体的动画，那你动画的插值器就应该是符合自由落体的速度规则:V^2 = 2gh。

# 5、还没完

经过前面的步骤，基本上已经实现了目标效果，来看看效果吧。



![实现效果](http://upload-images.jianshu.io/upload_images/3631399-08319cc60962944e.gif?imageMogr2/auto-orient/strip)



但是还有一些细节要做处理，现在抛出这么一个问题，假设用户在使用你的这个控件是给宽高指定宽高为wrap_conten，你觉得会是怎么样？答案是和match_parent是一样的效果。为什么会这样呢？要解释这个问题也不难：
ViewGroup#getChildMeasureSpec
```
// Parent has imposed a maximum size on us
case MeasureSpec.AT_MOST:
    if (childDimension >= 0) {
        // Child wants a specific size... so be it
        resultSize = childDimension;
        resultMode = MeasureSpec.EXACTLY;
    } else if (childDimension == LayoutParams.MATCH_PARENT) {
        // Child wants to be our size, but our size is not fixed.
        // Constrain child to not be bigger than us.
        resultSize = size;
        resultMode = MeasureSpec.AT_MOST;
    } else if (childDimension == LayoutParams.WRAP_CONTENT) {
        // Child wants to determine its own size. It can't be
        // bigger than us.
        resultSize = size;
        resultMode = MeasureSpec.AT_MOST;
    }
    break;
```
看到MATCH_PARENT和WRAP_CONTENT是一样的了吗？那怎么处理？可以重写View的onMeasure()方法，当使用wrap_content时，给它指定一个默认的高度即可。
```
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int mode = MeasureSpec.getMode(heightMeasureSpec);
    //当在布局文件设置高度为wrap_content时，默认为80dp(如果不处理效果和math_parent效果一样)，宽度就不处理了
    if (mode == MeasureSpec.AT_MOST) {
        heightMeasureSpec = MeasureSpec.makeMeasureSpec((int) DisplayUtil.dp2px(80, mContext), MeasureSpec.EXACTLY);
    }
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
}
```
这里的处理原则是如果高度是wrap_content就直接指定高度为80dp，宽度不做处理，所以宽度你用wrap_content和match_parent效果都一样。关于这方面的内容可以研究一下View的测量过程。关于详细的实现还是看[源码][2]吧。

一般来讲，自定义View都需要一些自定义属性来让其更加具有通用性，那我们要支持哪些自定义属性呢？

|name|说明|format|默认值|
|:--|:--|:--|:--:|
|bgColor|背景颜色|color|```#FBE40C```|
|calibrationColor|刻度线的颜色|color|```#FFFFFF```|
|calibrationWidth|刻度线的宽度|dimension|1dp|
|calibrationShort|短的刻度线的长度|dimension|20dp|
|calibrationLong|长的刻度线的长度|dimension|35dp|
|triangleColor|三角形指示器的颜色|color|```#FFFFFF```|
|triangleHeight|三角形的高度|dimension|18dp|
|textColor|刻度尺上数值字体颜色|color|```#FFFFFF```|
|textSize|刻度尺上数值字体大小|dimension|14sp|
|per|两个刻度之间的代表的数值|float|1|
|perCount|两条长的刻度线之间的per数量|integer|10|
|gapWidth|刻度之间的物理距离|dimension|10dp|
|minValue|刻度尺的最小值|float|0|
|maxValue|刻度之间的最大值|float|100|
|value|当前值|float|0|

什么？这个小东西有这么多属性？这个问题这样，如果高度定制，可以写死一些东西，如果想通用性更好，那就不能写死一些东西，随之而来的可能是性能的下降或者复杂度提升。

# 6、总结总结
总结这个事，不是每个人都愿意做的？为什么呢，因为不敢。因为走了那么长的路你累了？还是生怕回头发现你是在耗费青春？亦或是怕回头看看的时候发现很多东西还没有搞懂？谁知道呢。无论如何今天要勇敢一把，首先看看前面用到了哪些知识点：
1. View的绘制（画背景、画刻度线、画三角形，画文字）
2. View的测量（处理wrap_content）
3. 弹性滑动（Scroller）
4. 触摸事件处理（onTouchEvent，提到关于事件传递的概念）

这些知识看起来都比较零碎，那如何才能让自己在自定义View、ViewGroup是没那么吃力呢，换句话说自定义View应该掌握哪些知识？

1. Android系统的事件分发机制，前面也提供了触摸事件，了解它是如果从屏幕传递到目标View，比如处理滑动冲突就要了解事件的分发过程；
2. View的measure；
3. View的layout；
4. View的draw;
5. 动画相关的知识。

怎么学？可以结合一些书籍引导着来看源码，或者直接debug跟踪一下源码，通过方法调用栈一步步分析下去。

如果您有不同的见解，欢迎留下您的足迹，谢谢大家！


等会儿，视觉设计师喝完茶回来了...


[源码传送门][2]

[1]: http://hencoder.com/activity-mock-1/
[2]: https://github.com/jdqm/TapeView