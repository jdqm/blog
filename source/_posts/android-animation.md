---
title: Android动画总结
date: 2018-05-05 18:34:54
updated: 2018-05-05 18:34:54
tags:
    - Android
    - Animation
    - 动画
categories: Android
---

Android动画的发展历程：

|3.0之前|3.0|5.0|
| -- | -- | -- |
|View动画|增加属性动画，低版本不兼容|增加转场动画，低版本兼容|
动画在UI开发中算比较重要的一块，合理正确地使用动画可以让你的产品体验更出色。这篇文章主要是总结Android中的动画类型和基本使用方式。
<!-- more -->
![Android动画分类](http://upload-images.jianshu.io/upload_images/3631399-db1e5e0c2ae19f70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Demo-属性动画](http://upload-images.jianshu.io/upload_images/3631399-3098e675f5661dec.gif?imageMogr2/auto-orient/strip)

![转场动画](http://upload-images.jianshu.io/upload_images/3631399-aea1e8ed8eceb231.gif?imageMogr2/auto-orient/strip)

[Demo源码: https://github.com/jdqm/AnimationDemo](https://github.com/jdqm/AnimationDemo)

## I.View动画
View动画从大方面来看，可以分为两类，一类是补间（Tween）动画，即我们只需要提供初始状态和终止状态，动画框架自动计算出中间的状态；另一类帧动画，每一帧图片都需要提供，使用简单，但相对来说耗费的资源就比较多。

1.TranslationAnimation
**构造方法一**
```
/**
 * @param fromXDelta 动画开始时的X坐标
 * @param toXDelta 动画停止时的X坐标
 * @param fromYDelta 动画开始时的y坐标
 * @param toYDelta 动画停止时的y坐标
 *
 * Note:这里的坐标指定的左上角的坐标
 */
public TranslateAnimation(float fromXDelta, float toXDelta, float fromYDelta, float toYDelta)
```
1.1举例：将一个图片向右平移100dp，动画时长为300ms，视图停止在动画结束的状态
```
Animation ta = new TranslateAnimation(0, dpToPixel(100), 0, 0);
ta.setDuration(300);//动画时长
ta.setFillAfter(true); //视图停在动画结束的状态
imageView.startAnimation(ta);
```

**构造方法二**
```
/**
 * @param fromXType 指定fromXType的解释方式，可以是 Animation.ABSOLUTE,Animation.RELATIVE_TO_SELF(以自己作为参考点), 或者Animation.RELATIVE_TO_PARENT（以父View作为参考的）.
 * @param fromXValue 动画开始的x值， 如果fromXType是ABSOLUTE那就指定一个具体的数字，否则就是一个百分比
 *
 * Note：其他的参数也是类似的理解方式
 */
public TranslateAnimation(int fromXType, float fromXValue, int toXType, float toXValue,
        int fromYType, float fromYValue, int toYType, float toYValue) 
```
1.2举例：实现1.1例子一样的效果,即向右平移100dp
```
ta = new TranslateAnimation(
Animation.ABSOLUTE, 0,
Animation.ABSOLUTE, dpToPixel(100),
Animation.ABSOLUTE, 0,
Animation.ABSOLUTE, 0);
```
1.3举例：使用Animation.RELATIVE_TO_PARENT实现从父布局左边移动到父布局的水平中间
```
ta = new TranslateAnimation(
Animation.RELATIVE_TO_PARENT, 0.0f,
Animation.RELATIVE_TO_PARENT, 0.5f,
Animation.ABSOLUTE, 0,
Animation.ABSOLUTE, 0);
```


2.缩放动画-ScaleAnimation
```
Animation sa = new ScaleAnimation(1.0f, 1.5f, 1.0f, 1.5f);
sa.setDuration(300);
sa.setFillAfter(true);
imageView.startAnimation(sa);
```

3.透明动画-AlphaAnimation
```
 Animation sa = new AlphaAnimation(0.0f, 1.0f);
sa.setDuration(1000);
sa.setFillAfter(true);
imageView.startAnimation(sa);
```
4.旋转动画-RotationAnimation
```
Animation sa = new RotateAnimation(0, 180);
sa.setDuration(300);
sa.setFillAfter(true);
imageView.startAnimation(sa);
```

5.帧动画-FrameAnimation
此文件放于 drawable下：music_animation.xml
```
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false">

    <item android:drawable="@drawable/music_play01" android:duration="150"/>
    <item android:drawable="@drawable/music_play02" android:duration="150"/>
    <item android:drawable="@drawable/music_play03" android:duration="150"/>
    <item android:drawable="@drawable/music_play04" android:duration="150"/>

</animation-list>
```

```
public class FrameAnimationActivity extends AppCompatActivity {

    private View frameAnimationView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_twenn_animation);
        frameAnimationView = findViewById(R.id.frameAnimationView);
        frameAnimationView.setBackgroundResource(R.drawable.music_animation);
    }

    public void start(View view) {
        AnimationDrawable frameAnimation = (AnimationDrawable) frameAnimationView.getBackground();
        frameAnimation.start();
    }

    public void stop(View view) {
        AnimationDrawable frameAnimation = (AnimationDrawable) frameAnimationView.getBackground();
        frameAnimation.stop();
    }
}
```
View动画除了在Java代码中使用外，还可以在xml中进行定义，放在 res/anim 目录下。
```
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300"
    android:fillAfter="true"
    android:fillBefore="true"
    android:repeatMode="restart"
    android:shareInterpolator="true">

    <translate
        android:fromXDelta="0"
        android:fromYDelta="0"
        android:toXDelta="100"
        android:toYDelta="0" />

    <alpha
        android:fromAlpha="0"
        android:toAlpha="1.0" />

    <scale
        android:fromXScale="0"
        android:fromYScale="0"
        android:pivotX="0.5"
        android:pivotY="0.5"
        android:toXScale="1.0"
        android:toYScale="1.0" />

    <rotate
        android:fromDegrees="0"
        android:pivotX="0.5"
        android:pivotY="0.5"
        android:toDegrees="360" />
</set>
```
在Java代码中使用 AnimationUtils 来加载这个动画资源。这种写法似乎可读性更好一些，当然这个过程涉及到一些XML的解析，性能上可能会有一些影响，但影响不大。
```
Animation animation = AnimationUtils.loadAnimation(context, R.anim.animations);
```

## II.属性动画

1.TranslatorAnimator
```
switch (state) {
    case 0:
        imageView.animate().translationX(dpToPixel(100))
                .setDuration(300)
                .setInterpolator(new AccelerateDecelerateInterpolator())
                .setListener(new AnimatorListenerAdapter() {
                    @Override
                    public void onAnimationEnd(Animator animation) {
                        Toast.makeText(context.getApplicationContext(), "平移动画结束了", Toast.LENGTH_SHORT).show();
                    }
                });
        //如果View默认的Animator没有你想要改变的属性，可以用下面这种写法
        //ObjectAnimator animator = ObjectAnimator.ofFloat(imageView, "translationX", dpToPixel(100));
        //animator.start();
        break;
    case 1:
        imageView.animate().translationX(0);
        break;
    case 2:
        imageView.animate().translationYBy(dpToPixel(100));
        break;
    case 3:
        imageView.animate().translationYBy(-dpToPixel(100));
        break;
    case 4:
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            imageView.animate().translationZ(dpToPixel(15));
        }
        break;
    case 5:
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            imageView.animate().translationZ(0);
        }
        break;
    default:
        break;
}
```

2.ScaleAnimator
```
switch (state) {
    case 0:
        imageView.animate().scaleX(1.5f)
                .setDuration(300)
                .setInterpolator(new AccelerateDecelerateInterpolator());
        break;
    case 1:
        imageView.animate().scaleX(1.0f);
        break;
    case 2:
        imageView.animate().scaleY(.5f);
        break;
    case 3:
        imageView.animate().scaleY(1.0f);
        break;
    default:
        break;
}
```
3.AlphaAnimator
```
if (state == 0) {
    imageView.animate().alpha(0);
    state = 1;
} else {
    imageView.animate().alpha(1);
    state = 0;
}
```
4.RotationAnimator
```
switch (state) {
    case 0:
        imageView.animate().rotation(180);
        break;
    case 1:
        imageView.animate().rotation(0);
        break;
    case 2:
        imageView.animate().rotationX(180);
        break;
    case 3:
        imageView.animate().rotationX(0);
        break;
    case 4:
        imageView.animate().rotationY(180);
        break;
    case 5:
        imageView.animate().rotationY(0);
        break;
    default:
        break;
}
```

5.PropertyValueHolder

```
PropertyValuesHolder holder1 = PropertyValuesHolder.ofFloat("translationX", dpToPixel(100));
PropertyValuesHolder holder2 = PropertyValuesHolder.ofFloat("alpha", 1.0f);
PropertyValuesHolder holder3 = PropertyValuesHolder.ofFloat("rotation", 360);

ObjectAnimator animator = ObjectAnimator.ofPropertyValuesHolder(imageView, holder1, holder2, holder3);
animator.setDuration(500)
        .start();
```
6.AnimatorSet
```
if (state == 0) {
    ObjectAnimator animator1 = ObjectAnimator.ofFloat(imageView, "translationX", dpToPixel(100));
    ObjectAnimator animator2 = ObjectAnimator.ofFloat(imageView, "rotation", 360);
    ObjectAnimator animator3 = ObjectAnimator.ofFloat(imageView, "alpha", 0);

    AnimatorSet animatorSet = new AnimatorSet();
    //animatorSet.playTogether(animator1, animator2, animator3);
    animatorSet.playSequentially(animator1, animator2, animator3);
    animatorSet.start();

    state = 1;
} else {
    ObjectAnimator animator1 = ObjectAnimator.ofFloat(imageView, "translationX", 0);
    ObjectAnimator animator2 = ObjectAnimator.ofFloat(imageView, "rotation", 0);
    ObjectAnimator animator3 = ObjectAnimator.ofFloat(imageView, "alpha", 1);

    AnimatorSet animatorSet = new AnimatorSet();
    animatorSet.playTogether(animator1, animator2, animator3);
    animatorSet.start();

    state = 0;
}
```
## III.转场动画

在Android5.0以前，要个Activity的切换添加一些动画，可以这么干。
```
startActivity(new Intent(this, TransitionActivity.class));
overridePendingTransition(R.anim.activity_enter_anim, R.anim.activity_exit_anim);
```
activity_enter_anim.xml
```
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="500">
    <translate
        android:fromXDelta="100%"
        android:toXDelta="0" />

</set>
```
activity_exit_anim.xml
```
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="500">
    <translate
        android:fromXDelta="0"
        android:toXDelta="-100%" />
</set>
```

但是这种动画只是对整个Activity生效，并不能针对Activity布局里的具体View。Android5.0增加的转场动画，可以有效处理这个问题。

1.source Activity
1.1 Activity java code
```
public void explode(View view) {
    Intent intent = new Intent(this, TransitionsActivity.class);
    intent.putExtra("flag", 0);
    if (SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        startActivity(intent, ActivityOptions.makeSceneTransitionAnimation(this).toBundle());
    }
}

public void slide(View view) {
    Intent intent = new Intent(this, TransitionsActivity.class);
    intent.putExtra("flag", 1);
    if (SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        startActivity(intent, ActivityOptions.makeSceneTransitionAnimation(this).toBundle());
    }
}

public void fade(View view) {
    Intent intent = new Intent(this, TransitionsActivity.class);
    intent.putExtra("flag", 2);
    if (SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        startActivity(intent, ActivityOptions.makeSceneTransitionAnimation(this).toBundle());
    }
}

public void share(View view) {
    View fab = findViewById(R.id.btnFab);
    Intent intent = new Intent(this, TransitionsActivity.class);
    intent.putExtra("flag", 3);
    if (SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        startActivity(intent, ActivityOptions.makeSceneTransitionAnimation(this,
                Pair.create(view, "share"),
                Pair.create(fab, "fab")).toBundle());

    }
}
```
1.2 activity layout
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.jdqm.animationdemo.transition.TransitionActivity">

    <Button
        android:id="@+id/btnExplode"
        android:layout_width="match_parent"
        android:layout_height="100dp"
        android:onClick="explode"
        android:text="explode" />

    <Button
        android:id="@+id/btnSlide"
        android:layout_width="match_parent"
        android:layout_height="100dp"
        android:onClick="slide"
        android:text="slide"
        app:layout_constraintTop_toBottomOf="@id/btnExplode" />

    <Button
        android:id="@+id/btnFade"
        android:layout_width="match_parent"
        android:layout_height="100dp"
        android:onClick="fade"
        android:text="fade"
        app:layout_constraintTop_toBottomOf="@id/btnSlide" />

    <Button
        android:id="@+id/btnShare"
        android:layout_width="match_parent"
        android:layout_height="100dp"
        android:onClick="share"
        android:text="share"
        android:transitionName="share"
        app:layout_constraintTop_toBottomOf="@id/btnFade" />

    <Button
        android:id="@+id/btnFab"
        android:layout_width="56dp"
        android:layout_height="56dp"
        android:background="@drawable/ripple_round"
        android:elevation="5dp"
        android:transitionName="fab"
        app:layout_constraintTop_toBottomOf="@id/btnShare" />

</android.support.constraint.ConstraintLayout>
```

1.3 ripple_round.xml 

```
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">

    <solid android:color="@android:color/holo_green_dark"/>
    <corners android:radius="28dp"/>

</shape>
```


2.target Activity
2.1 Activity Java code
```
public class TransitionsActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        getWindow().requestFeature(Window.FEATURE_CONTENT_TRANSITIONS);
        int flag = getIntent().getExtras().getInt("flag");
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            switch (flag) {
                case 0:
                    getWindow().setEnterTransition(new Explode());
                    break;
                case 1:
                    getWindow().setEnterTransition(new Slide());
                    break;
                case 2:
                    getWindow().setEnterTransition(new Fade());
                    getWindow().setExitTransition(new Fade());
                    break;
                case 3:
                    break;
                default:
                    break;
            }
        }
        setContentView(R.layout.activity_transitions);
    }
}
```

2.2 activity_layout
```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context="com.jdqm.animationdemo.transition.TransitionsActivity">

    <View
        android:id="@+id/holderView"
        android:layout_width="match_parent"
        android:layout_height="300dp"
        android:background="@color/colorAccent"
        android:transitionName="share" />

    <Button
        android:id="@+id/btnFab"
        android:layout_width="56dp"
        android:layout_height="56dp"
        android:layout_alignParentEnd="true"
        android:layout_below="@id/holderView"
        android:layout_marginEnd="10dp"
        android:layout_marginTop="-26dp"
        android:background="@drawable/ripple_round"
        android:elevation="5dp"
        android:transitionName="fab" />

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_below="@id/holderView"
        android:layout_marginTop="10dp">

        <Button
            android:id="@+id/button"
            android:layout_width="match_parent"
            android:layout_height="60dp"
            android:layout_below="@+id/button4" />

        <Button
            android:id="@+id/button4"
            android:layout_width="match_parent"
            android:layout_height="60dp"
            android:layout_alignParentStart="true"
            android:layout_marginTop="10dp" />
    </RelativeLayout>
</RelativeLayout>
```

## IV.总结
动画的实现原理，其实就是通过不断的改变视图的位置并进行重新绘制，难的是如何计算出下一帧的位置、大小等信息，而动画框架则是帮我们完成了这些个杂的计算过程，详情可了解一下动画的插值器Interpolator。

对于以上所提到View动画、属性动画以及转出动画，在给出的Demo中都有完整的示例。需要注意的是，View动画只是对View的影像做了变换，但其实际事件响应坐标等还是在原来的位置，而属性动画则解决了这个问题。
谢谢大家！