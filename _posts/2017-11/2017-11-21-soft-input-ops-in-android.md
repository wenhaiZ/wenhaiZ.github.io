---
layout: post
title: "Android 键盘操作汇总"
date: 2017-11-21 19:39:00 +0800
tags: [Code,Android]
subtitle: "To be Continued"
---
这篇博客是对 Android 开发过程中软键盘操作的总结，主要包括键盘的主动弹出、收起以及一些监听操作。   

## 弹出软键盘
Andorid 中的 `EditText` 在获取焦点时会主动弹出软键盘以供用户输入，我们也可以根据需要主动弹出软键盘，软键盘的弹出只需要三行代码就可以搞定：
```kotlin
fun showSoftInput() {
    //1.接收输入的 EditText 获取焦点（如果当前 EditText 已经获取焦点，可以省略）
    mEtSearch.requestFocus()
    //2. 获取 InputMethodManager
    val inputManager: InputMethodManager = activity.getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
    //3. 通过 showSoftInput 弹出软键盘
    inputManager.showSoftInput(mEtSearch, InputMethodManager.SHOW_FORCED)
}
```  
>Android 中用于处理应用和输入法之间交互的框架称为 IMF (*Input Method Framwork*)，而 [InputMethodManager](https://developer.android.google.cn/reference/android/view/inputmethod/InputMethodManager.html)是 IMF 框架的核心系统级 API。 

`showSoftInput(View view, int flags)` 的参数含义如下：
- **view**：接收用户输入**并且已经获取焦点的** View。如果在调用该方法时未获取焦点，则键盘不会弹起
- **flag**：附加的操作信息。当取值为 `SHOW_IMPLICIT`时，表明弹出键盘**不是由用户直接发起**的请求，而是隐式的，此时键盘**可能不会弹出**；当取值为`SHOW_FORCED`时，表明弹出键盘请求**由用户直接发起**，在用户请求收起键盘前，软键盘会一直显示。    

## 收起软键盘
收起键盘的操作同样需要用到 InputMethodManager，只是调用的方法不同：
```kotlin
fun hideSoftInput() {
    val inputManager: InputMethodManager = activity.getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
    inputManager.hideSoftInputFromWindow(mEtSearch.windowToken, 0)
}
```
`hideSoftInputFromWindow(IBinder windowToken, int flags)` 的官方解释为请求从当前正在接收输入的窗口上下文中隐藏软键盘 (*request to hide the soft input window from the context of the window that is currently accepting input*)，它接收的参数含义如下：
- **windowToken**：发起请求的窗口的象征 (*token*)，可通过 `View.getWindowToken()`方法获得
- **flag**：附加的操作信息。当取值为`HIDE_IMPLICIT_ONLY`时，表明软键盘**只会在用户没有明确弹出键盘**的情况下（也就是通过 `SHOW_IMPLICIT` 显示键盘时）隐藏；当取值为 `HIDE_NOT_ALWAYS` 时，表明如果键盘**不是**通过`SHOW_FORCED`显示的，那么就会隐藏。  

我将 `showSoftInput` 和 `hideSoftInputFromWindow` 两个方法的 flag 所有可能的搭配进行了简单的测试，得到了下表:   

|flag for show|flag for hide|Can softInput be hidden?|
|:----:|:----:|:-----:|
|0|0|Yes|
|0|HIDE_IMPLICIT_ONLY|No|
|0|HIDE_NOT_ALWAYS|Yes|
|SHOW_IMPLICIT|0|Yes|
|SHOW_IMPLICIT|HIDE_IMPLICIT_ONLY|Yes|
|SHOW_IMPLICIT|HIDE_NOT_ALWAYS|Yes|
|SHOW_FORCED|0|Yes|
|SHOW_FORCED|HIDE_IMPLICIT_ONLY|No|
|SHOW_FORCED|HIDE_NOT_ALWAYS|No|      

>上表中所有的情况下键盘都能成功弹出。   

结合键盘隐藏情况以及官方文档对 `showSoftInput` 方法 flag 参数的介绍，可以得出：调用 `showSoftInput()` 传入 `SHOW_FORED` （保证键盘一定可以弹起），在调用 `hideSoftInputFromWindow()` 时传入 `0` 是最佳组合。  

当然，如果你不想在 flag 上浪费时间，可以简单粗暴的将两个方法中的 flag 都设为 `0`。


## 重写回车键动作
有时候需要将软键盘上的回车键变为「搜索」或者「发送」等功能来提升用户体验，这可以通过设置控件的属性并添加监听来实现。  

以「发送」为例。   

首先，在 `layout` 文件中，对相应的 `EditText` 进行配置：
```xml
<EditText
        android:id="@+id/input"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:imeOptions="actionSend"
        android:inputType="text"/>
```
需要设置`android:imeOptions` 和 `android:inputType`两个属性，通过前者来指定发送、搜索、下一步等动作，后者指定输入的文本类型。**如果不设置`android:inputType`，就达不到预期效果。**    

>IME 的意思是输入法编辑器(*Input Method Editor*)，`android:imeAction` 可以设置的 action 及其含义可以参考[官方文档](https://developer.android.google.cn/reference/android/view/inputmethod/EditorInfo.html)。

然后，给 EditText 设置动作监听。
```kotlin
mEditText.setOnEditorActionListener { view, actionId, event ->
    //判断 actionId
    if (actionId == EditorInfo.IME_ACTION_SEND) {
        //do something
        
        return@setOnEditorActionListener true
    }
    return@setOnEditorActionListener false
}
```
这样配置后，软键盘上的回车就会变成「发送」，用户点击后就会执行相应的代码。     

之所以要设置 `android:inputType` 是因为在 `TextView` 的 `onEditorAction()` 中有如下代码：
```java
public void onEditorAction(int actionCode) {
        final Editor.InputContentType ict = mEditor == null ? null : mEditor.mInputContentType;
        //判断 InputContentType 是否为空
        if (ict != null) {
            if (ict.onEditorActionListener != null) {
                if (ict.onEditorActionListener.onEditorAction(this,actionCode, null)) {
                    return;
                }
            }

            //...handle default action
        }
}
```
可以看到，如果不设置`android:inputType`，那么代码中的 `ict` 为 `null`，即使设置了监听器也不会起作用。   


## 键盘弹出后的界面调整
有时候软键盘的弹出会遮挡输入区域，此时就需要在 `AndoridManifest.xml`对 `<activity>` 的`android:windowSoftInputMode`属性进行配置：
```xml
<!-- 配置 android:windowSoftInputMode 属性-->
<activity android:name=".MainActivity"
    android:windowSoftInputMode="adjustNothing">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
```   
按照[官方文档](https://developer.android.google.cn/guide/topics/manifest/activity-element.html#wsoft)的说法，`android:windowSoftInputMode`属性主要影响 Activity 的两个方面：
- 当 Activity 成为用户焦点时软键盘的状态（显示或隐藏）
- 是否调整 Activity 窗口或者平移内容来为软键盘腾出空间    

其可设定的值如下表：   


|值|含义|
|:----:|:-------------:|
|stateUnspecified|不指定软键盘的状态（隐藏还是可见）。 将由系统选择合适的状态，或依赖主题中的设置。这是对软键盘行为的默认设置。|
|stateUnchanged|当 Activity 转至前台时保留软键盘最后所处的任何状态|
|stateHidden|当用户选择 Activity（当用户确实是向前导航到 Activity，而不是因离开另一 Activity 而返回时）时隐藏软键盘。|
|stateAlwaysHidden|当 Activity 的主窗口有输入焦点时始终隐藏软键盘。|
|stateVisible|在正常的适宜情况下（当用户向前导航到 Activity 的主窗口时）显示软键盘。|
|stateAlwaysVisible|当用户选择 Activity（当用户确实是向前导航到 Activity，而不是因离开另一 Activity 而返回）时显示软键盘。|
|adjustUnspecified|不指定 Activity 的主窗口是否调整尺寸以为软键盘腾出空间，或者窗口内容是否进行平移以在屏幕上显露当前焦点。 系统会根据窗口的内容是否存在任何可滚动其内容的布局视图来自动选择其中一种模式。 如果存在这样的视图，窗口将进行尺寸调整，前提是可通过滚动在较小区域内看到窗口的所有内容。这是对主窗口行为的默认设置。|
|adjustResize|始终调整 Activity 主窗口的尺寸来为屏幕上的软键盘腾出空间。|
|adjustPan|不调整 Activity 主窗口的尺寸来为软键盘腾出空间， 而是自动平移窗口的内容，使当前焦点永远不被键盘遮盖，让用户始终都能看到其输入的内容。 *这通常不如尺寸调正可取，因为用户可能需要关闭软键盘以到达被遮盖的窗口部分或与这些部分进行交互。*|  

其中 `state_xxx` 决定当 Activity 成为用户焦点时软键盘的状态，`adjust_xxx` 决定软键盘弹出时窗口的配置。`android:windowSoftInputMode`属性值可以设为上表中的一个或者是`state_xxx`和`adjust_xx`的组合，例如：
```xml
android:windowSoftInputMode="stateVisible|adjustResize"
```   
如果发现`adjustResize`没有达到预期效果，可以参考下面文章，但不保证一定会有效果：  
- [取消主题中全屏设置](https://stackoverflow.com/questions/8398102/androidwindowsoftinputmode-adjustresize-doesnt-make-any-difference)
- [使用 ScrollView 包裹](https://stackoverflow.com/questions/22370710/androidwindowsoftinputmode-adjustresize-is-not-working-as-it-should-be)
- [重写 Layout](https://stackoverflow.com/questions/21092888/windowsoftinputmode-adjustresize-not-working-with-translucent-action-navbar)

通过 `ScrollView` 包裹布局的方法比较简单，但是需要用户滚动布局才能看到被软键盘遮挡的部分，其他方式在我的测试下并没有用。  


我想到的一个解决思路是：将`android:windowSoftInputMode`设为`adjustPan`，这样就不会遮挡输入区域，然后想上面那样在`OnEditorActionListener` 中监听`ACTION_DONE`事件，然后用户点击「完成」后隐藏软键盘，恢复界面。

如果将`android:windowSoftInputMode`属性设为`adjustPan`不能满足要求，而设置成`adjustResize`又没有效果，那么可以尝试下面的操作。
## 监听键盘的弹出和收起
我们可以对键盘的弹出和收起进行监听并执行相应动作，比如上面说的，在键盘弹起时 Android 的界面调整无效或者无法满足需求，那么就可以在键盘弹出后通过动画来主动移动内容以露出被键盘遮挡的部分。  

监听键盘弹出或收起的方法有很多种，可以参考[这个问题](https://stackoverflow.com/questions/4312319/how-to-capture-the-virtual-keyboard-show-hide-event-in-android)下的答案，这里只介绍一种：

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var mEditText: EditText
    //1. 声明 OnGlabalLayoutListener 
    private lateinit var mGlobalLayoutListener: ViewTreeObserver.OnGlobalLayoutListener

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        mEditText = findViewById(R.id.input)
        //2.实例化 OnGlobalLayoutListener
        mGlobalLayoutListener = ViewTreeObserver.OnGlobalLayoutListener {
            if (isSoftInputShown(mEditText.rootView)) {//软键盘弹出
                Toast.makeText(this@MainActivity, "SoftInput Shown", Toast.LENGTH_SHORT).show()
            } else {//软键盘收起
                Toast.makeText(this@MainActivity, "SoftInput Hidden", Toast.LENGTH_SHORT).show()
            }
        }
        //3.添加监听
        mEditText.viewTreeObserver.addOnGlobalLayoutListener(mGlobalLayoutListener)
    }


    /**
    监听键盘是否弹出的方法
    */
    private fun isSoftInputShown(rootView: View): Boolean {
        val softKeyboardHeight = 100 //这个值可以略微调整
        val r = Rect()
        rootView.getWindowVisibleDisplayFrame(r)
        val dm = rootView.resources.displayMetrics
        val heightDiff = rootView.bottom - r.bottom
        return heightDiff > softKeyboardHeight * dm.density
    }

    override fun onDestroy() {
        super.onDestroy()
        //4.移除监听器
        mEditText.viewTreeObserver.removeOnGlobalLayoutListener(mGlobalLayoutListener)
    }
}
```  

`isSoftInputShown()` 方法通过判断根布局的显示高度和总高度之差是否大于一个值来判断软键盘是否弹出，并且考虑了屏幕像素密度。  
这种方法在`android:windowSoftInputMode`为`adjustPan`和`adjustResize`时都能奏效。   

*需要注意的是一定要在 `onDestroy()` 回调中移除监听器，否则可能会引发内存泄漏。*

   
## To be Continued
以上就是我目前遇到过的 Android 软键盘操作的总结，以后遇到其他情况，再做更新。：）