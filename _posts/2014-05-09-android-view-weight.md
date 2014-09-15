---
title: Android 组件(view)的权重(weight)
layout: post
tags: [Android]
comments: true
---
当使用如下布局时：

{% highlight xml %}
<LinearLayout
    android:orientation="horizontal"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    xmlns:android="http://schemas.android.com/apk/res/android">

    <EditText
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:id="@+id/edit_msg"
        android:hint="@string/edit_msg"/>
    <Button
        android:layout_weight="1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:id="@+id/btn_send"
        android:text="@string/btn_send"/>
</LinearLayout>

{% endhighlight %}


button可以正常工作，但是当在text field输入很长的数据时，text field会挤占button的空间，最终使button“消失”。使用weigh可以很好的解决这个问题。


weigh的值表示应该屏幕应为其保留的空间。如，一个view的weigh为2，第二个view为1，总共为3。所以第一个view占据2/3保留空间，第二个view占据剩下的空间。如果添加第三个weigh为1的view，那么第一个view现在占据1/2保留空间，第三个view得到1/4空间。

所有views的默认weigh为0，所以如果你只给一个view设置任何大于0的weigh值，那么这个view将具有其他所有views的剩余空间。所以，在本例中，设置EditText的weigh为1，而不设置button的weigh。

{% highlight xml %}
 <EditText
        android:layout_weight="1"
        ... />
{% endhighlight %}

为了得到更好的布局效果，应该将EditText的width设置为零(0dp).设置width为0，可以提供布局的性能，因为"wrap_content"会去计算文本的宽度。

{% highlight xml %}
<EditText
        android:layout_weight="1"
        android:layout_width="0dp"
        ... />
{% endhighlight %}

最终的布局文件如下:

{% highlight xml %}
<LinearLayout
    android:orientation="horizontal"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    xmlns:android="http://schemas.android.com/apk/res/android">

    <EditText
        android:layout_weight="1"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:id="@+id/edit_msg"
        android:hint="@string/edit_msg"/>
    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:id="@+id/btn_send"
        android:text="@string/btn_send"/>
    
</LinearLayout>

{% endhighlight %}

