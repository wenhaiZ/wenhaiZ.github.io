---
layout: post
title: "[译]在 Android 中使用 Kotlin 委托属性"
date: 2019-11-18 19:30:00 +0800
tags: [Code,Kotlin,Android]
subtitle: "在 Android 开发中发挥委托属性的优势"
published: true
---
>**原文作者：[Dmitry Akishin](https://proandroiddev.com/@akishindev),Android developer at FINCH, Moscow**

> 原文链接：[Kotlin Delegates in Android: Utilizing the power of Delegated Properties in Android development](https://proandroiddev.com/kotlin-delegates-in-android-1ab0a715762d)


Kotlin 真的是一门美丽的开发语言，她拥有的一些很棒的特性使 Android 开发变成的有趣和令人兴奋。[委托属性](https://kotlinlang.org/docs/reference/delegated-properties.html)就是其中之一,在这篇文章里我们将会看到委托是如何把 Android 开发变得更加轻松的。 


## 基础

首先，什么是委托？它又是如何工作的？虽然委托看起来有点魔幻，但它其实并没有想象中的那么复杂。   

委托就是一个类，这个类为属性提供值并且处理值的变化。这让我们可以把属性的 getter-setter 逻辑从属性声明的地方移动到（或者说委托给）另一个类，以达到逻辑复用的目的。   

比如我们有一个`String`类型的属性`param`，这个属性的值需要去掉首尾的空格(trim)。我们可以在属性的 setter 里这样做：
```kotlin
class Example {
    var param: String = ""
        set(value) {
            field = value.trim()
        }
}
```   


*如果对语法不熟悉，可以参考 Kotlin 文档的[属性部分](https://kotlinlang.org/docs/reference/properties.html)。*    

如果我们想要在其他类里复用这个逻辑呢？这就轮到**委托**登场了。  

```kotlin
class TrimDelegate : ReadWriteProperty<Any?, String> {

    private var trimmedValue: String = ""

    override fun getValue(
        thisRef: Any?,
        property: KProperty<*>
    ): String {
        return trimmedValue
    }

    override fun setValue(
        thisRef: Any?,
        property: KProperty<*>, value: String
    ) {
        trimmedValue = value.trim()
    }
}
```  

委托就是一个拥有两个方法（读取和设置属性的值）的类。更具体来说，`KProperty`类的示例代表被委托的属性，而`thisRef`就是拥有这个属性的对象。仅此而已。我们可以这样使用刚才创建的委托：   
```kotlin
class Example {
    //使用 by 关键字
    var param: String by TrimDelegate()
}
```   

上面的代码和下面的代码效果相同：
```kotlin
class Example {

    private val delegate = TrimDelegate()

    var param: String
        get() = delegate.getValue(this, ::param)
        set(value) {
            delegate.setValue(this, ::param, value)
        }
}
```

*`::param`是一个操作符，他可以为属性返回一个`KProperty`实例。*  

如你所见，委托属性并没有什么神奇的。但是，它虽然简单，却非常有用，让我们来看一些在 Android 开发中的例子。

*你可以在[官方文档](https://kotlinlang.org/docs/reference/delegated-properties.html)中了解更多关于委托属性的内容。*

## 给 Fragment 传参

我们经常需要给 Fragment 传递一些参数，这通常看起来是这样：
```kotlin
class DemoFragment : Fragment() {
    private var param1: Int? = null
    private var param2: String? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        arguments?.let { args ->
            param1 = args.getInt(Args.PARAM1)
            param2 = args.getString(Args.PARAM2)
        }
    }
    companion object {
        private object Args {
            const val PARAM1 = "param1"
            const val PARAM2 = "param2"
        }

        fun newInstance(param1: Int, param2: String): DemoFragment =
            DemoFragment().apply {
                arguments = Bundle().apply {
                    putInt(Args.PARAM1, param1)
                    putString(Args.PARAM2, param2)
                }
            }
    }
}
```

我们把参数传递给用于创建 Fragment 实例的 `newInstance`方法，在方法里面把参数传递给 Fragment 的 `arguments`，以便可以在`onCreate`中获取。  

我们可以把 `arguments`相关的逻辑移到属性的 getter 和 setter 中来代码变得更好看。

```kotlin
class DemoFragment : Fragment() {
    private var param1: Int?
        get() = arguments?.getInt(Args.PARAM1)
        set(value) {
            value?.let {
                arguments?.putInt(Args.PARAM1, it)
            } ?: arguments?.remove(Args.PARAM1)
        }
    private var param2: String?
        get() = arguments?.getString(Args.PARAM2)
        set(value) {
            arguments?.putString(Args.PARAM2, value)
        }
    companion object {
        private object Args {
            const val PARAM1 = "param1"
            const val PARAM2 = "param2"
        }
        fun newInstance(param1: Int, param2: String): DemoFragment =
            DemoFragment().apply {
                this.param1 = param1
                this.param2 = param2
            }
    }
}
```

但是我们还是要为每个属性写重复的代码，如果属性太多的话就太繁琐了，而且太多与`arguments`相关的代码看起来太乱了。    


所以还有别的方法进一步美化代码吗？答案是有的。正如你猜的那样，我们将会用委托属性。  


首先，我们需要做些准备。   

Fragment的`arguments`用`Bundle`对象存储的,`Bundle`提供了很多方法用于存储不同类型的值。所以让我们来写一个扩展函数用于往Bundle 中存储某种类型的值，在类型不支持的时候抛出异常。  

```kotlin
fun <T> Bundle.put(key: String, value: T) {
    when (value) {
        is Boolean -> putBoolean(key, value)
        is String -> putString(key, value)
        is Int -> putInt(key, value)
        is Short -> putShort(key, value)
        is Long -> putLong(key, value)
        is Byte -> putByte(key, value)
        is ByteArray -> putByteArray(key, value)
        is Char -> putChar(key, value)
        is CharArray -> putCharArray(key, value)
        is CharSequence -> putCharSequence(key, value)
        is Float -> putFloat(key, value)
        is Bundle -> putBundle(key, value)
        is Parcelable -> putParcelable(key, value)
        is Serializable -> putSerializable(key, value)
        else -> throw IllegalStateException("Type of property $key is not supported")
    }
}
```

接下来我们可以创建委托了。  

```kotlin
class FragmentArgumentDelegate<T : Any> 
    :ReadWriteProperty<Fragment, T> {

    @Suppress("UNCHECKED_CAST")
    override fun getValue(
        thisRef: Fragment,
        property: KProperty<*>
    ): T {
        //key 为属性名
        val key = property.name
        return thisRef.arguments
            ?.get(key) as? T
            ?: throw IllegalStateException("Property ${property.name} could not be read")
    }

    override fun setValue(
        thisRef: Fragment,
        property: KProperty<*>, value: T
    ) {
        val args = thisRef.arguments
            ?: Bundle().also(thisRef::setArguments)
        val key = property.name
        args.put(key, value)
    }
}
```

委托从 `Fragment` 的 `arguments` 中读取值，当属性值改变时，它会获取`Fragment`的`arguments`（如果没有则会创建新的并设置给`Fragment`），然后通过刚才创建的扩展函数`Bundle.put`把新的值存储起来。   


`ReadWriteProperty` 是一个接收两个类型参数的泛型接口。我们把第一个设置成`Fragment`，即保证这个委托只能用于`Fragment`的属性。这可以让我们通过`thisRef`来获取`Fragment`实例并管理它的 `arguments`。

**由于我们使用属性的名称作为`arguments`存储时的键，所以我们不用再把键写成常量了**。  

`ReadWriteProperty` 第二个类型参数决定了这个属性可以拥有那些类型的值。我们把这个类型设为非空的，并且在不能读取时抛出了异常，这让我们可以在 Fragment 中获取非空的值，避免了空值检查。   

但有时我们确实需要一些属性是可以为`null`的，所以让我们再创建一个委托，当在`arguments`中没有找到值时不抛出异常而是返回`null`。    

```kotlin
class FragmentNullableArgumentDelegate<T : Any?> :
    ReadWriteProperty<Fragment, T?> {

    @Suppress("UNCHECKED_CAST")
    override fun getValue(
        thisRef: Fragment,
        property: KProperty<*>
    ): T? {
        val key = property.name
        return thisRef.arguments?.get(key) as? T
    }

    override fun setValue(
        thisRef: Fragment,
        property: KProperty<*>, value: T?
    ) {
        val args = thisRef.arguments
            ?: Bundle().also(thisRef::setArguments)
        val key = property.name
        value?.let { args.put(key, it) } ?: args.remove(key)
    }
}
```  


接下来，为了方便使用我们创建一些函数（不是必须的，单纯为了美化代码）：

```kotlin
fun <T : Any> argument(): ReadWriteProperty<Fragment, T> =
    FragmentArgumentDelegate()

fun <T : Any> argumentNullable(): ReadWriteProperty<Fragment, T?> =
    FragmentNullableArgumentDelegate()
```

最后，我们来使用委托：

```kotlin
class DemoFragment : Fragment() {
    private var param1: Int by argument()
    private var param2: String by argument()
    companion object {
        fun newInstance(param1: Int, param2: String): DemoFragment =
            DemoFragment().apply {
                this.param1 = param1
                this.param2 = param2
            }
    }
}
```

看起来很整洁吧？  


## SharedPreferences 委托

我们经常需要存储一些数据以便App下次启动时能够快速获取。例如，我们可能想存储一些用户偏好以便让用户自定义应用的功能。普遍采用的方式是使用 `SharedPreferences` 来存储键值对。  

假设我们有一个类用户读取和存储三个参数：   

```kotlin
class Settings(context: Context) {

    private val prefs: SharedPreferences = 
        PreferenceManager.getDefaultSharedPreferences(context)

    fun getParam1(): String? {
        return prefs.getString(PrefKeys.PARAM1, null)
    }

    fun saveParam1(param1: String?) {
        prefs.edit().putString(PrefKeys.PARAM1, param1).apply()
    }

    fun getParam2(): Int {
        return prefs.getInt(PrefKeys.PARAM2, 0)
    }

    fun saveParam2(param2: Int) {
        prefs.edit().putInt(PrefKeys.PARAM2, param2).apply()
    }

    fun getParam3(): String {
        return prefs.getString(PrefKeys.PARAM3, null) 
            ?: DefaulsValues.PARAM3
    }

    fun saveParam3(param3: String) {
        prefs.edit().putString(PrefKeys.PARAM2, param3).apply()
    }

    companion object {
        private object PrefKeys {
            const val PARAM1 = "param1"
            const val PARAM2 = "param2"
            const val PARAM3 = "special_key_param3"
        }

        private object DefaulsValues {
            const val PARAM3 = "defaultParam3"
        }
    }
}
```

这里我们获取了默认的`SharedPreferences`并提供了方法用户读取和存储参数的值。我们还把`param3`变得特别一点——它使用了特别的键并且有一个非标准的默认值。  


我们又一次看到我们写了重复的代码，我们当然可以重复的逻辑移到方法里，但还是会留下很笨重的代码。除此之外，如果我们想在别的类里复用这些逻辑呢？让我们来看看委托是如何简化代码的吧。  

为了让事情变得有趣些，我们尝试一种稍微不同的方式。这次我们将会使用[对象表达式](https://kotlinlang.org/docs/reference/object-declarations.html)并给`SharedPreferences`创建一个扩展函数。  


```kotlin
fun SharedPreferences.string(
    defaultValue: String = "",
    key: (KProperty<*>) -> String = KProperty<*>::name
): ReadWriteProperty<Any, String> =
    object : ReadWriteProperty<Any, String> {
        override fun getValue(
            thisRef: Any,
            property: KProperty<*>
        ) = getString(key(property), defaultValue)

        override fun setValue(
            thisRef: Any,
            property: KProperty<*>,
            value: String
        ) = edit().putString(key(property), value).apply()
    }
```   

这里我们创建了 `SharedPreferences` 的扩展函数，它返回了一个 `ReadWriteProperty` 子类的对象作为我们的委托。   

这个委托用函数`key`提供的值作为键，从`SharedPreferences`读取`String`类型的值。默认情况下，键为属性的名字，所以我们不用维护和传递任何常量。同时，如果为了避免键冲突或者想访问该键，我们还可以提供一个自定义的键。我们还可以为属性提供一个默认值，以防在`SharedPreferences`没有找到值。   

这个委托也可以使用相同的键来在`SharedPreferences`存储属性的新值。   

为了让我们的例子能工作，我们还需要为`String?`和`Int`增加委托，这和前面是一样的：
```kotlin
fun SharedPreferences.stringNullable(
    defaultValue: String? = null,
    key: (KProperty<*>) -> String = KProperty<*>::name
): ReadWriteProperty<Any, String?> =
    object : ReadWriteProperty<Any, String?> {
        override fun getValue(
            thisRef: Any,
            property: KProperty<*>
        ) = getString(key(property), defaultValue)

        override fun setValue(
            thisRef: Any,
            property: KProperty<*>,
            value: String?
        ) = edit().putString(key(property), value).apply()
    }

fun SharedPreferences.int(
    defaultValue: Int = 0,
    key: (KProperty<*>) -> String = KProperty<*>::name
): ReadWriteProperty<Any, Int> =
    object : ReadWriteProperty<Any, Int> {
        override fun getValue(
            thisRef: Any,
            property: KProperty<*>
        ) = getInt(key(property), defaultValue)

        override fun setValue(
            thisRef: Any,
            property: KProperty<*>,
            value: Int
        ) = edit().putInt(key(property), value).apply()
    }
```


现在我们终于可以简化`Settings`类的代码了：
```kotlin
class Settings(context: Context) {

    private val prefs: SharedPreferences =
        PreferenceManager.getDefaultSharedPreferences(context)

    var param1 by prefs.stringNullable()
    var param2 by prefs.int()
    var param3 by prefs.string(
        key = { "KEY_PARAM3" },
        defaultValue = "default"
    )
}
```   

代码现在看起来好多了，如果需要增加一个属性，一行代码就够了。   


## View 委托  

假设我们有一个自定义View，它包含三个文本字段——一个标题，一个子标题，还有描述——布局如下：   

```xml
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical">

    <TextView
        android:id="@+id/tvTitle"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <TextView
        android:id="@+id/tvSubtitle"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <TextView
        android:id="@+id/tvDescription"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

</LinearLayout>
```    

我们想要`CustomView`提供用于修改和获取三个字段的方法：
```kotlin
class CustomView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : FrameLayout(context, attrs) {
    var title: String
        get() = tvTitle.text.toString()
        set(value) {
            tvTitle.text = value
        }
    var subtitle: String
        get() = tvSubtitle.text.toString()
        set(value) {
            tvSubtitle.text = value
        }
    var description: String
        get() = tvDescription.text.toString()
        set(value) {
            tvDescription.text = value
        }
    init {
        inflate(context, R.layout.custom_view, this)
    }
}
```   

*这里我们使用了[Kotlin Android Extension](https://kotlinlang.org/docs/tutorials/android-plugin.html)的[视图绑定](https://kotlinlang.org/docs/tutorials/android-plugin.html#view-binding)来获取布局中的控件。*   

很明显有一些代码可以很容易的移动到另一个类里，让我们借助委托来完成。  


让我们写一个 TextView 的扩展函数，它返回一个委托用来处理它的文本内容：

```kotlin
fun TextView.text(): ReadWriteProperty<Any, String> =
    object : ReadWriteProperty<Any, String> {
        override fun getValue(
            thisRef: Any,
            property: KProperty<*>
        ): String = text.toString()

        override fun setValue(
            thisRef: Any,
            property: KProperty<*>, value: String
        ) {
            text = value
        }
    }
```  

然后在`CustomView`中使用它：

```kotlin
class CustomView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : FrameLayout(context, attrs) {

    init {
        inflate(context, R.layout.custom_view, this)
    }
    
    var title by tvTitle.text()
    var subtitle by tvSubtitle.text()
    var description by tvDescription.text()
}
```  

**确保在init方法渲染布局之后初始化属性，因为控件不能为`null`。**   


这跟源代码比起来可能并没有很大的改进，关键是展示委托的力量。除此之外，这写起来很有趣。   

当然，不仅限于`TextView`。比如，这里有一个控件可见性的委托（`keepBounds`决定了当控件不可见时是否占用空间）:   

```kotlin
fun View.isVisible(keepBounds: Boolean): ReadWriteProperty<Any, Boolean> =
    object : ReadWriteProperty<Any, Boolean> {
        override fun getValue(
            thisRef: Any,
            property: KProperty<*>
        ): Boolean = visibility == View.VISIBLE

        override fun setValue(
            thisRef: Any,
            property: KProperty<*>,
            value: Boolean
        ) {
            visibility = when {
                value -> View.VISIBLE
                keepBounds -> View.INVISIBLE
                else -> View.GONE
            }
        }
    }
```  

这里有一个`ProgressBar`进度的委托，返回从 0 到 1 的浮点数。  

```kotlin
fun ProgressBar.progress(): ReadWriteProperty<Any, Float> =
    object : ReadWriteProperty<Any, Float> {
        override fun getValue(
            thisRef: Any,
            property: KProperty<*>
        ): Float = if (max == 0) 0f else progress / max.toFloat()

        override fun setValue(
            thisRef: Any,
            property: KProperty<*>, value: Float
        ) {
            progress = (value * max).toInt()
        }
    }
```  

下面是如果`CustomView`中有一个`ProgressBar`该如何使用它：
```kotlin
class CustomView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : FrameLayout(context, attrs) {

    init {
        inflate(context, R.layout.custom_view, this)
    }
    
    var title by tvTitle.text()
    var subtitle by tvSubtitle.text()
    var description by tvDescription.text()

    var progress by progressBar.progress()
    var isProgressVisible by progressBar.isVisible(keepBounds = false)
}
```  

如你所见，你可以给任何东西委托，没有限制。  


## 总结
我们看来一些在 Android 开发中使用 Kotlin 委托属性的例子。当然了，你也可以用别的方式来使用它。  
这篇文章的目标是展示委托属性是多么强大，以及我们可以用它做什么。   

希望你现在已经有了想要使用委托的想法了。






