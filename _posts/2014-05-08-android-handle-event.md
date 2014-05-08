---
title: Android 事件处理机制
layout: post
tags: [Android]
comments: true
---

# 概述

Android包含两套事件处理机制：

* 基于监听的事件处理
* 基于回调的事件处理

一般来说，基于回调的事件处理用于处理一些具有通用性的事件，基于回调的事件处理代码会
显得比较简洁。但对于某些特定的事件，无法使用基于回调的事件处理，只能采用基于监听的
事件处理。

# 基于监听的事件处理

## 监听的处理模型

在监听的处理模型中，主要涉及三类对象。

* Event Source（事件源）：事件发生的场所，通常为各个组件，如按钮、窗口等。
* Event（事件）：事件封装界面上发生的特定事情。一般通过Event对象取得事件的相关信息。
* Event Listener（事件监听器）：负责监听事件源发生的事件，并做出相应的响应。

基于监听的时间处理模型的编程步骤如下：

1. 获取普通界面组件（事件源），也就是被监听的对象。
2. 实现事件监听器类，必须实现一个XxxListener接口。
3. 调用时间源的setXxxListener方法将事件监听器对象注册给普通组件。


### 内部类作为事件监听器类

使用内部类可以在当前类中复用该监听器类；因为监听器是外部类的内部类，所有可以只用
访问外部类的所有界面组件。

{% highlight java %}

public class MainAct extends Activity
{
    @Override
    public void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        //获取应用的bn按钮
        Button bn = (Button) findViewById(R.id.bn); 
        //绑定事件监听器
        bn.setOnClickListener(new MyClikListener());

    }
    //定义单击事件监听器
    class MyClickListener implements View.OnClickListener
    {
        //实现监听器必须实现的方法，该方法作为事件处理器
        @Override
        public void onClick(View v)
        {
            
        }
    }
}

{% endhighlight %}

### 外部类作为事件监听器类(不推荐)

使用外部类定义事件监听器比较少见，主要原因如下：

* 事件监听器通常属于特定的GUI界面，定义成外部类不利于提高程序的内聚性。
* 外部类形式的事件监听器不能自由访问创建GUI界面的类中组件，编程不够简洁。

{% highlight java %}

public class SendSmsListener implements View.OnLongClickListener
{
    private Activity activity;
    private EditText address;
    private EditText content;
    public SendSmsListener(Activity activity, EditText address,
                           EditText context)
    {
        this.activity = activity;
        this.address = address;
        this.content = context;
    }

    @Override
    public boolean onLongClick(View view) {
        String addressStr = address.getText().toString();
        String contentStr = content.getText().toString();

        SmsManager smsManager = SmsManager.getDefault();
        PendingIntent sendIntent = PendingIntent.getBroadcast(
                activity, 0, new Intent(), 0);
        smsManager.sendTextMessage(addressStr, null, contentStr,
                sendIntent, null);
        Toast.makeText(activity, "短信发送完成", Toast.LENGTH_LONG).show();

        return false;


    }
}

{% endhighlight %}

{% highlight java %}
public class MainActivity extends Activity 
{

    EditText address;
    EditText content;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        address = (EditText) findViewById(R.id.address);
        content = (EditText) findViewById(R.id.content);
        Button bn = (Button) findViewById(R.id.send);

        bn.setOnLongClickListener(new SendSmsListener(
                this, address, content
        ));
    }
}

{% endhighlight %}

不推荐将业务逻辑实现写在事件监听器中，包含业务逻辑的事件监听器将导致程序的显示
逻辑和业务逻辑耦合，增加程序的耦合度。如果有多个事件监听器需要相同的业务逻辑功能，
可以考虑使用业务逻辑组件定义业务逻辑功能，再让事件监听器来调用业务逻辑组件的业务
逻辑方法

### Activity本身作为事件监听器

这种形式非常简洁，但是有两个缺点：

* 可能造成程序结果混乱，Activity的主要职责应该是完成界面初始化工作
* 如果Activity界面类需要实现监听器接口，让人改进比较怪异



{% highlight java %}
public class MainActivity extends Activity implements View.OnClickListener
{

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button bn = (Button) findViewById(R.id.show);
        //直接使用Activity作为事件监听器
        bn.setOnClickListener(this);
    }
    //实现事件处理方法
    @Override
    public void onClick(View view) 
    {

    }
}
{% endhighlight %}

### 匿名内部类作为事件监听器类（最常用）


{% highlight java %}

public class MainActivity extends Activity
{

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button bn = (Button) findViewById(R.id.show);
        bn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {

            }
        });
    }
}
{% endhighlight %}

### 直接绑定到标签

{% highlight xml %}

<LinearLayout
    android:orientation="vertical"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    xmlns:android="http://schemas.android.com/apk/res/android">

    <!--在标签页中为按钮绑定事件处理方法-->
    <Button
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="click me"
        android:onClick="clickHandler"/>

</LinearLayout>

{% endhighlight %}

{% highlight java %}

public class MainActivity extends Activity
{
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    //定义一个事件处理方法
    //其中source代表事件源
    public void clickHandler(View source)
    {
        
    }
}

{% endhighlight %}

## 基于回调的事件处理

### 回调机制与监听机制

对于回调的事件处理模型来说，事件源和事件监听器是统一的，或者说事件监听器完全消失了。
当用户在GUI组件上激活某个事件时，组件自己特定的方法会负责处理该事件。

为了使用回调机制类处理GUI组件上所发生的事情，我们需要为该组件提供对应的事件处理
方法---继承GUI组件，并重写该类的时间处理方法来实现。

{% highlight java %}

public class MyButton extends Button
{
    public MyButton(Context context, AttributeSet set)
    {
        super(context, set);
    }

    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        super.onKeyDown(keyCode, event);
        Log.v("app", "the onKeyDown in MyButton");
        //该事件不会向外扩散
        return true;
    }
}

{% endhighlight %}

上面自定义的MyButton类中，我们重写了Button类的onKeyDown(int keyCode, KeyEvent event)方法，
该方法负责处理按钮上的键盘事件。

{% highlight xml %}

<LinearLayout
    android:orientation="vertical"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- 使用自定义View时应使用全限定类名-->
    <com.example.test.app.MyButton
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="click me"
    />

</LinearLayout>

{% endhighlight %}

### 基于回调的事件传播

几乎所有基于回调的事件处理方法都有一个`boolean`类型的返回值，返回值用于标识该处理
方法是否完全处理该事件。

* 如果返回true，表明该处理方法已完全处理该事件，事件不会传播出去。
* 如果返回false，表明该处理方法未完全处理该事件，事件会传播出去。

对于基于回调事件传播而言，某组件上所发生的事情不仅激发该组件上的回调方法，也会触发
该组件所在Activity的回调方法---只要事件能传播到该Activity。

下面的程序示范Android系统事件传播，程序重写Button类的onKeyDown方法，而且重写了该
Button所在Activity的onKeyDown方法---而且程序没有阻止事件传播。

{% highlight java %}

public class MyButton extends Button
{
    public MyButton(Context context, AttributeSet set)
    {
        super(context, set);
    }

    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        super.onKeyDown(keyCode, event);
        Log.v("app", "the onKeyDown in MyButton");
        //该事件依然会向外扩散
        return false;
    }
}

{% endhighlight%}

{% highlight java %}

public class MainActivity extends Activity
{
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Button bn = (Button) findViewById(R.id.show);
        bn.setOnKeyListener(new View.OnKeyListener() {
            @Override
            public boolean onKey(View view, int i, KeyEvent keyEvent) {
                //只处理按下键的事件
                if(keyEvent.getAction() == KeyEvent.ACTION_DOWN)
                {
                    Log.v("-Lisenter-", "the onKeyDown in Listener");
                }
                //该事件未处理完
                return false;
            }
        });
    }

    //重写onKeyDown方法，该方法可监听它所包含的所有组件的按键按下事件
    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        super.onKeyDown(keyCode, event);
        Log.v("-Activity-", "the onKeyDown in Activity");
        //该事件未处理完，依然向外传播
        return false;
    }
}
{% endhighlight %}

按下按键时，log窗口输出：


    05-08 04:48:48.301: V/-Lisenter-(2955): the onKeyDown in Listener
    05-08 04:48:48.301: V/app(2955):        the onKeyDown in MyButton
    05-08 04:48:48.301: V/-Activity-(2955): the onKeyDown in Activity
    
可以看出Android系统最先触发的应该是按键上绑定的事件监听器，接着触发该组件提供的
事件回调方法，然后传播到该组件所在的Activity。

对比Android提供的两种事件处理模型，不难发现基于监听的事件处理模型具有更大的优势：

* 分工更明确，事件源、事件监听由两个类分开实现，具有更好的可维护性。
* Android的事件处理机制保证基于监听的事件监听器会被优先触发。
