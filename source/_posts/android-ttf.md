---
title: Android自定义字体
date: 2017-09-10 12:03:15
updated: 2017-10-21 12:03:15
tags:
    -Android
    -自定义
categories: Android
---

准备字体文件,创建目录 src/main/assets/fonts/，然后把你的字体文件拷贝到这个目录下，Android支持 .ttf (TrueTypeFont) or .otf (OpenTypeFont) 这两种格式。

<!-- more -->

假设我们有这样一个TextView引用
```
TextView tvText = (TextView)findViewById(R.id.tvText);
```

方法一：

```
// 创建一个 TypeFace 
Typeface mtypeFace = Typeface.createFromAsset(getAssets(),"fonts/MyFont.ttf");

// 为目标控件设置TypeFace
tvText.setTypeface(mtypeFace);
```
这种方法需要为所有需要使用自定义字体的TextView /Button /EditText 设置字体，如果你有大量的需要使用自定义字体的控件，这显然不是明智的选择，那么也许你需要方法二。

方法二：

创建一个自定义TextView，然后在XML文件中直接使用（当然你也可以自定义Button /EditText，道理是一样的）。

```
public class CustomTextView extends TextView {
   
    private static final String TAG = "CustomTextView";
 
    public CustomTextView(Context context) {
        super(context);
    }
 
    public CustomTextView(Context context, AttributeSet attrs) {
        super(context, attrs);
        setCustomFont(context, attrs);
    }
 
    public CustomTextView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        setCustomFont(context, attrs);
    }
 
    private void setCustomFont(Context ctx, AttributeSet attrs) {
        TypedArray a = ctx.obtainStyledAttributes(attrs, R.styleable.CustomTextView);
        String customFont = a.getString(R.styleable.CustomTextView_customFont);
        setCustomFont(ctx, customFont);
        a.recycle();
    }
 
    public boolean setCustomFont(Context ctx, String asset) {
        Typeface typeface = null;
        try {
            typeface = Typeface.createFromAsset(ctx.getAssets(),"fonts/"+asset);
        } catch (Exception e) {
            Log.e(TAG, "Unable to load typeface: "+e.getMessage());
            return false;
        }
 
        setTypeface(typeface);
        return true;
    }
}
```

这个自定义TextView有一个自定义属性，所以你需要在你的values/attrs.xml（若没有则创建这样一个文件）文件中添加如下内容:

```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="CustomTextView">
        <attr name="customFont" format="string"/>
    </declare-styleable>
</resources>
```

自定义TextView创建完毕，现在开始在XML文件中使用。

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout 
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:orientation="vertical" 
    android:layout_width="match_parent"
    android:layout_height="match_parent">
 
    <com.jdqm.CustomTextView
        android:id="@+id/tv_custom"
        android:layout_height="match_parent"
        android:layout_width="match_parent"
        android:text="@string/sample_text"
        app:customFont="my_font.otf">
    </com.jdqm.CustomTextView>
</LinearLayout>

```
