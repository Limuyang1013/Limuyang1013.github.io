---
title: Android Design Support Library--TextInputLayout的使用
date: 2016-04-30 00:20:19
categories: Android
tags: [Material Design,Google,TextInputLayout]
toc: true
---

#### 引言
Google在2015的IO大会上，给我们带来了更加详细的Material Design设计规范，同时，也给我们带来了全新的Android Design Support Library，Android Design Support Library的兼容性更广，直接可以向下兼容到Android 2.2，我准备从最简单的控件开始，逐渐延伸，把新控件都给熟悉一遍。
<!--more-->
先从看起来最简单的控件开始，也就是`TextInputLayout`，说实话`TextInputLayout` 我所见到的平常用的并不多，它的大体作用是在我们正常的EditText左上角显示出一个浮动标签，这个标签的内容就是我们设置的`android:hint` 属性的值。
先来看一下它的继承结构：

![继承结构](http://oasusatoz.bkt.clouddn.com/TextInput_1.jpg)

可以很清晰的看到我们的`TextInputLayout` 继承于`LinearLayout` ，那么很明显这是一个布局，需要配合它的子控件来显示出想要的效果，这里谷歌把它专门设计用来包裹`EditText`(或者`EditText`的子类)，然后当用户进行输入动作的时候我们设置的`android:hint` 提示就会以动画的形式运动到左上角，谷歌官方提供的最简单的使用示例如下：

```xml
 <android.support.design.widget.TextInputLayout
         android:layout_width="match_parent"
         android:layout_height="wrap_content">

     <android.support.design.widget.TextInputEditText
             android:layout_width="match_parent"
             android:layout_height="wrap_content"
             android:hint="@string/form_username"/>

 </android.support.design.widget.TextInputLayout>
```
有些人可能会奇怪，之前说好的`TextInputLayout` 是用来包裹`EditText` 的，为什么这里出现了`TextInputEditText` ，先别急，我们看一下谷歌官方对这个控件的描述：

```java
A special sub-class of EditText designed for use as a child of TextInputLayout.

Using this class allows us to display a hint in the IME when in 'extract' mode.
```
大意是说，这只是一种特殊的`EditText` 的子类，用来在`'extract' mode` 下在输入法编辑器中显示我们的`hint`提示信息，这里的`'extract' mode` 其实就是全屏模式，谷歌官方对它的解释是有时候你的输入框的UI界面很大，大的不能与你自己的应用程序的UI结合起来，这时候就可以切换到全屏模式来输入，这么说可能不太明白，上图：
比如说，下面这种情况使用的是`EditText`：

![使用EditText](http://oasusatoz.bkt.clouddn.com/TextInput_2.jpg)

我们看到下面那里输入框已经很大了，然后你点击输入框进行输入，会发现这个现象：

![点击输入](http://oasusatoz.bkt.clouddn.com/TextInput_3.jpg)

你进入到了全屏模式输入，但是界面上空空如也，对比一下使用`TextInputEditText` 的情况：

![使用TextInputEditText](http://oasusatoz.bkt.clouddn.com/TextInput_4.jpg)

看到左上角的文字了嘛，这是我们在之前设置的`android:hint` 属性的值，这么一看这两者的区别的就一目了然了，但是说实话`TextInputEditText` 用到的地方还是很有限的，所以日常开发我们还是使用`TextInputLayout` 去包裹`EditText` 来实现浮动标签的功能。

> [**以上图片出处**](http://stackoverflow.com/questions/35775919/edittext-added-is-not-a-textinputedittext-please-switch-to-using-that-class-ins) 感谢万能的**stackoverflow**

#### 常用方法
因为它是继承自`LinearLayout`的所以理论上`LinearLayout` 有的属性它全都有，这里我们只看有关它本身的属性：

| 属性名                   	| 相关方法                         	| 描述                                                   	|
|--------------------------	|----------------------------------	|--------------------------------------------------------	|
| app:counterEnabled       	| setCounterEnabled(boolean)       	| 设置是否显示一个计数器，布尔值                         	|
| app:counterMaxLength     	| setCounterMaxLength(int)         	| 设置计数器的最大计数数值，整型                         	|
| app:errorEnabled         	| setErrorEnabled(boolean)         	| 设置是否显示一个错误信息，布尔值                       	|
| app:hintAnimationEnabled 	| setHintAnimationEnabled(boolean) 	| 设置是否要显示输入状态时候的动画效果，布尔值           	|
| app:hintEnabled          	| setHintEnabled(boolean)          	| 设置是否要用这个浮动标签的功能，布尔值                 	|
| app:hintTextAppearance   	| setHintTextAppearance(int)       	| 设置提示文字的样式(注意这里是运行了动画效果之后的样式) 	|


----------


这里我们通过一个简单的Demo来了解以上这些属性，简单起见我们就做一个登录界面，这个界面长这样：


![这里写图片描述登录界面](http://oasusatoz.bkt.clouddn.com/TextInput_5.jpg)

先上布局文件：

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    tools:context="com.test.textinputlayoutdemo.MainActivity">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_marginTop="65dp"
        android:orientation="vertical">

        <android.support.design.widget.TextInputLayout
            android:id="@+id/layout_name"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:counterEnabled="true"
            app:counterMaxLength="5"
            app:counterOverflowTextAppearance="@style/MyOverflowText"
            app:errorTextAppearance="@style/MyErrorStyle">

            <EditText
                android:id="@+id/input_name"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="@string/EnterName"
                android:singleLine="true" />
        </android.support.design.widget.TextInputLayout>

        <android.support.design.widget.TextInputLayout
            android:id="@+id/layout_password"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:counterEnabled="true"
            app:counterMaxLength="11"
            app:counterOverflowTextAppearance="@style/MyOverflowText"
            app:errorTextAppearance="@style/MyErrorStyle">

            <EditText
                android:id="@+id/input_password"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="@string/EnterPassWord"
                android:inputType="textPassword"
                android:singleLine="true" />
        </android.support.design.widget.TextInputLayout>

        <android.support.design.widget.TextInputLayout
            android:id="@+id/layout_email"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:counterOverflowTextAppearance="@style/MyOverflowText"
            app:errorTextAppearance="@style/MyErrorStyle">

            <EditText
                android:id="@+id/input_email"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:hint="@string/EnterEmail"
                android:inputType="textEmailAddress"
                 />
        </android.support.design.widget.TextInputLayout>

        <Button
            android:id="@+id/login"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="50dp"
            android:background="@color/colorPrimary"
            android:text="@string/login"
            android:textColor="#ffffff"
            android:textSize="20sp"
            android:textStyle="bold" />
    </LinearLayout>


</RelativeLayout>


```
代码如下：

```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {

    private EditText input_name, input_password, input_email;
    private TextInputLayout layout_name, layout_password, layout_email;
    private Button btn_login;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initWidget();

    }

    private void initWidget() {
        input_name = (EditText) findViewById(R.id.input_name);
        input_password = (EditText) findViewById(R.id.input_password);
        input_email = (EditText) findViewById(R.id.input_email);

        layout_name = (TextInputLayout) findViewById(R.id.layout_name);
        layout_password = (TextInputLayout) findViewById(R.id.layout_password);
        layout_email = (TextInputLayout) findViewById(R.id.layout_email);

        btn_login = (Button) findViewById(R.id.login);
        btn_login.setOnClickListener(this);

        //添加监听
        input_name.addTextChangedListener(new MyTextWatcher(input_name));
        input_password.addTextChangedListener(new MyTextWatcher(input_password));
        input_email.addTextChangedListener(new MyTextWatcher(input_email));
    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.login:
                canLogin();
                break;
            default:
                break;

        }
    }

    /**
     * 判断是否可以登录的方法
     */
    private void canLogin() {
        if (!isNameValid()) {
            Toast.makeText(this, getString(R.string.check), Toast.LENGTH_SHORT).show();
            return;
        }
        if (!isPasswordValid()) {
            Toast.makeText(this, getString(R.string.check), Toast.LENGTH_SHORT).show();
            return;
        }
        if (!isEmailValid()) {
            Toast.makeText(this, getString(R.string.check), Toast.LENGTH_SHORT).show();
            return;
        }
        Toast.makeText(this, getString(R.string.login_success), Toast.LENGTH_SHORT).show();
    }

    public boolean isNameValid() {

        if (input_name.getText().toString().trim().equals("") || input_name.getText().toString().trim().isEmpty()) {
            layout_name.setError(getString(R.string.error_name));
            input_name.requestFocus();
            return false;
        }
        layout_name.setErrorEnabled(false);
        return true;
    }

    public boolean isPasswordValid() {
        if (input_password.getText().toString().trim().equals("") || input_password.getText().toString().trim().isEmpty()) {
            layout_password.setErrorEnabled(true);
            layout_password.setError(getResources().getString(R.string.error_password));
            input_password.requestFocus();
            return false;
        }
        layout_password.setErrorEnabled(false);
        return true;
    }

    public boolean isEmailValid() {
        String email = input_email.getText().toString().trim();
        if (TextUtils.isEmpty(email) || !android.util.Patterns.EMAIL_ADDRESS.matcher(email).matches()) {
            layout_email.setErrorEnabled(true);
            layout_email.setError(getString(R.string.error_email));
            layout_email.requestFocus();
            return false;
        }
        layout_email.setErrorEnabled(false);
        return true;
    }

    //动态监听输入过程
    private class MyTextWatcher implements TextWatcher {

        private View view;

        private MyTextWatcher(View view) {
            this.view = view;
        }

        @Override
        public void beforeTextChanged(CharSequence s, int start, int count, int after) {

        }

        @Override
        public void onTextChanged(CharSequence s, int start, int before, int count) {

        }

        @Override
        public void afterTextChanged(Editable s) {
            switch (view.getId()) {
                case R.id.input_name:
                    isNameValid();
                    break;
                case R.id.input_password:
                    isPasswordValid();
                    break;
                case R.id.input_email:
                    isEmailValid();
                    break;

            }

        }
    }
}
```


----------


先来看一下最终的实现效果：

![实现效果](http://oasusatoz.bkt.clouddn.com/TextInput_6.gif)


可以很明显的看到，当我们同时设置了`app:counterEnabled` 和`app:counterMaxLength` 属性时，我们输入的`EditText` 右下角会出现一个计数器还有一个最大输入字符数的数字显示，我们在输入名字这一栏设置最大输入为5个字符，所以当超过了5个字符的时候，`EditText` 的整个样式的颜色都会改变以示警告，如果我们只设置了`app:counterEnabled` 属性的话`EditText` 右下角一开始会出现一个0，随着输入字符的增多而逐步进行计数，注意如果设置了整个属性我们`EditText` 布局的高度会有一定的增大，具体的可以自己实践一下。

另外，我们在代码中设置了不同的饿输入类型，如果输入类型错误，我们就可以通过设置`app:errorEnabled` 来开启错误显示，此时需要通过在代码中调用 `setError(string)` 方法来设置显示的错误提示文字，当不需要的时候记得设置`app:errorEnabled(false)` 来取消错误提示，不然错误提示会一直存在。

**注意：** 当我们使用`app:counterMaxLength` 这个属性的时候，一定要设置 `app:counterOverflowTextAppearance` 属性，不然的话程序运行会报错，这个属性是设置当我们输入字符超过限定的个数时候`EditText`控件整体显示的样式，需要在style.xml文件里面定义一个自己的style，注意我们自定义的style的parent是`TextAppearance.AppCompat.Small` ，拿我上面的程序举例：

```xml
    <style name="MyOverflowText" parent="TextAppearance.AppCompat.Small">
        <item name="android:textColor">#f3672b</item>
    </style>
```
这样定义好后再在`app:counterOverflowTextAppearance` 里面设置这个style就行

#### 关于自定义样式

有些人可能不喜欢官方提供的默认样式想要自己定义，下面说一下自定义几种样式的方法：

 - 如果你想更改下划线的颜色，只要在style.xml文件里面找到AppTheme：

```xml
 <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
    </style>
```
更改里面的colorAccent属性就行了

 - 如果你想更改错误提示的样式的话，也是在style.xml文件里面，自定义一个style，同样拿上面的程序举例：

```xml
<style name="MyErrorStyle">
        <item name="android:textColor">#ec4722</item>
    </style>
```
然后在xml文件`TextInputLayout`控件里面这么设置一下就行了：

```
app:errorTextAppearance="@style/MyErrorStyle"
```

包括前面提到的设置当输入字符大于我们限定个数字符时的样式，基本上我们可以很好地自定义出自己想要的style了，以上两种不提供演示，都很简单，可以自己去尝试。

#### 最后

下一次准备分析SnackBar控件，很多东西说简单也简单，说不简单也不简单，就像做这个Demo我之前光看官方文档根本没有告诉有`app:counterOverflowTextAppearance` 这个属性的存在，也是一直查资料，还是要亲自去尝试一下才好，下面上源码(注意是AS文件)

参考：[**Android Material Design Floating Labels for EditText**](http://www.androidhive.info/2015/09/android-material-design-floating-labels-for-edittext/)

[**项目GitHub地址**](https://github.com/GiitSmile/TextInputLayoutDemo/tree/master)

最后来个小提示，当我们在Android Studio中导入support design开发包的时候，版本号最好和v7包的版本号一致，不然有些时候会出现莫名其妙的错误：

```gradle
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:23.0.1'
    compile 'com.android.support:design:23.0.1'
```

有任何问题欢迎留言探讨~

