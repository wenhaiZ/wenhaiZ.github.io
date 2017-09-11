---
layout: post
title: "ButterKnife 源码解析"
date: 2017-08-19 23:01:00 +0800
tags: [SourceCode]
comments: true
subtitle: " V 8.7.0"
published: true
---
![ButterKnife](https://github.com/JakeWharton/butterknife/raw/master/website/static/logo.png)
  
[ButterKnife](https://github.com/JakeWharton/butterknife) 是 `Android` 大神 [Jake Wharthon](https://github.com/JakeWharton) 的杰作，用于 `Android` 开发中的视图和资源绑定，可以让我们省去写诸如 `findViewById()`，`setOnClickListener()`和`getString()` 等代码，提升开发效率。   
这篇文章是我初步研究了项目源码后写的，只分析了视图绑定，所以只能算是源码简析。    
## ButterKnife 的使用
ButterKnife 使用起来很简单， 这里就简单示例一下。
- 添加依赖     

```java
dependecies{
    //ButterKnife
    compile 'com.jakewharton:butterknife:8.7.0'
    //ButterKnife 注解处理工具
    annotationProcessor 'com.jakewharton:butterknife-compiler:8.7.0'
}
   
```
- 在 Activity 中使用
```java
public class MainActivity extends AppCompatActivity {

    //通过 @BindView 注解来标记 View 组件
    @BindView(R.id.my_button)
    Button mButton;//注意不能声明为 private

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        //通过bind()方法来绑定视图
        ButterKnife.bind(this);
        //使用控件
        mButton.setText("ButterKnife");
    }

    //设置监听
    @OnClick(R.id.my_button)
    public void click(){
        Toast.makeText(this, "Button Click!", Toast.LENGTH_SHORT).show();
    }
}
```
更多用法和介绍可以参考[官方文档](http://jakewharton.github.io/butterknife/)。

## 源码解析(*省略 log 和异常信息* )
我们从调用入口 `ButterKnife.bind()` 开始追踪。
- `ButterKnife#bind()`
```java
@NonNull @UiThread
public static Unbinder bind(@NonNull Activity target) {
    //获得根布局
    View sourceView = target.getWindow().getDecorView();
    //调用 createBinding()
    return createBinding(target, sourceView);
}
```
这里传入的 `target` 就是上面的 `MainActivity` ，然后获取根布局，最后调用了 `createBinding()`。   
`bind()` 方法还有几个重载的方法，用以支持不同类型的 `target`，比如 `Dialog`、`View`，它们最后都会调用 `createBinding()` 方法。   
- `ButterKnife#createBinding()`
```java
private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    //获取 target 对应的类
    Class<?> targetClass = target.getClass();

    //通过 findBindingConstructorForClass() 方法找到与 target 对应的 UnBinder 实现类的构造方法
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

    if (constructor == null) {
        return Unbinder.EMPTY;
    }
    //通过反射创建 Unbinder 实例    
    return constructor.newInstance(target, source);
}
```
再看 `findConstructorForClass()` 方法
- `ButterKnife#findConstructorForClass()`
```java
@Nullable @CheckResult @UiThread
private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
    //static final Map<Class<?>, Constructor<? extends Unbinder>> BINDINGS = new LinkedHashMap<>();
    
    //先在 BINDINGS 缓存中查找
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    //如果缓存中存在对应 Constructor，直接返回
    if (bindingCtor != null) {
        return bindingCtor;
    }

    //如果不存在，通过反射查找
    String clsName = cls.getName();

    //如果类名以"android."或者"java."开头，说明查找到框架层，未找到，直接 return null
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
        if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
        return null;
    }
    
    try {
        //加载 clsName_ViewBinding.class
        Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
        //通过反射获取构造方法
         bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
        } catch (ClassNotFoundException e) {
            //没有找到就通过递归在父类的UnBinder中查找
            bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
        } catch (NoSuchMethodException e) {
            throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    //将查找到的 constructor 放入 BINDINGS 中缓存
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
}
```

可以看出，`Butterknife.bind(Object target) `方法就是寻找 `target类名_ViewBinding.class` （这个类实现了 `UnBinder` 接口），然后通过反射获得它的构造方法并创建 `UnBinder` 实例。  
`UnBinder` 接口中只有一个 `unbind()` 方法。
- `UnBinder`
```java
public interface Unbinder {
    //解绑view时调用 
    @UiThread void unbind();

    Unbinder EMPTY = new Unbinder() {
        @Override public void unbind() { 

        }
    };
}
```    

那这些后缀为`_ViewBinding` 的类的内容是什么呢？   

我们可以在`\app\build\generated\source\apt\debug\packegename\`下找到这些类。比如，这里 `ButterKnife` 为我生成的 Java 文件为 `MainActivity_ViewBinding.java`。   
来看一下里面的内容吧。
- `MainActivity_ViewBinding.java`
```java
// Generated code from Butter Knife. Do not modify!
public class MainActivity_ViewBinding implements Unbinder {
    private MainActivity target;
    private View view2131427422;

    //构造方法
    @UiThread
    public MainActivity_ViewBinding(MainActivity target) {
        //调用第二个构造方法  
        this(target, target.getWindow().getDecorView());
    }

    //这个构造方法就是 findBindingConstructorForClass()里通过反射获取的构造方法
    @UiThread
    public MainActivity_ViewBinding(final MainActivity target, View source) {
        this.target = target;
        View view;
        //找到需要的 View
        view = Utils.findRequiredView(source, R.id.my_button, "field 'mButton' and method 'click'");
        //转换为指定 View 类型然后为 MainActivity.mButton 赋值
        //因此 mButton 不能声明为 private
        target.mButton = Utils.castView(view, R.id.my_button, "field 'mButton'", Button.class);
        view2131427422 = view;
        //设置点击事件监听
        view.setOnClickListener(new DebouncingOnClickListener() {
            @Override
            public void doClick(View p0) {
                //调用 Activity 中 @onClick 标记的方法
                target.click();
            }
        });
    }


    //解除绑定
    @Override
    @CallSuper
    public void unbind() {
        MainActivity target = this.target;
        if (target == null) throw new IllegalStateException("Bindings already cleared.");
        this.target = null;
        target.mButton = null;
        view2131427422.setOnClickListener(null);
        view2131427422 = null;
    }
}
```   

由于我只绑定了一个 `Button`，所以这个类的代码很简单，但足以说明 `ButterKnife` 的工作原理。  

在 `_ViewBinding` 类的构造方法中，先通过 `findRequiredView()` 找到需要的 `View`，再调用 `castView()` 将 `View` 转换成指定类型，最后给 `Activity` 中的成员变量赋值并设置监听（如果需要的话）。   

接下来看一下 `findRequiredView()` 和 `castView()` 这两个方法。 

- `Utils.findRequiredView()` 
```java
public static View findRequiredView(View source, @IdRes int id, String who) {   
    View view = source.findViewById(id);
    if (view != null) {
        return view;
    }
    //view=null,抛出异常...
}
```
可以看到 `findRequiredView()` 也是通过 `findViewById()` 的方法来查找 `View` 的。
- `Utils.castView()` 这个方法主要用于类型转换
```java
 public static <T> T castView(View view, @IdRes int id, String who, Class<T> cls) {
    try {
        //cls 是要转成的 View 类型(Button.class,TextView.class....)
        return cls.cast(view);  
    } catch (ClassCastException e) {
        //异常处理
    }
}
```  

再看设置监听时用到的 `DebouncingOnClickListener`
- `DebouncingOnClickListener`
```java
public abstract class DebouncingOnClickListener implements View.OnClickListener {
    static boolean enabled = true;
    private static final Runnable ENABLE_AGAIN = new Runnable() {
        @Override 
        public void run() {
            enabled = true;
        }
    };

    @Override public final void onClick(View v) {
        if (enabled) {
            enabled = false;
            v.post(ENABLE_AGAIN);
            //在 onClick()方法中调用了 doClick(),
            doClick(v);
        }
   }

    public abstract void doClick(View v);
}
```
`DebouncingOnClickListener` 类实现了 `View.OnClickListener`，在`onClick()`方法中调用了`doClick()`方法，而在 `_ViewBinding.java` 中 `doClick()` 又调用了我们自己定义的用 `@onClick` 标记的方法。   

到此，`ButterKnife` 视图绑定的源码分析就完成了。   

可以看出 `ButterKnife` 的逻辑并没有很复杂——只是把我们平时写的 `findViewById()`封装到工具类中，然后为需要绑定的字段生成一个名称以 `_ViewBinding` 为后缀的 `Java` 文件，我们只需要通过注解标记一下要绑定的字段，然后通过 `ButterKnife.bind()` 就可以完成 `View` 的查找和字段的赋值。   

如果只介绍这些，就不用写这篇文章了，接下来还有更有意思的内容。

## 注解处理过程
了解了 `_ViewBinding` 类的工作原理，再来看一下这些类的源码是怎么生成的吧。这部分的源码在 `butterknife.compiler` 包下的 `ButterKnifeProcessor.java` 中。   

`ButterKnifeProcessor` 继承了 `javax.annotation.processing.AbstractProcessor`，对注解的处理主要是在 `processs()` 方法中。
- `ButterKnifeProcessor#process()`
```java
@Override 
public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
    //通过 findAndParseTarget() 生成绑定映射 bindingMap
    Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);
    
    //遍历所有 TypeElement-BindingSet 键值对    
    for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
        //获取 TypeElement,TypeElement 对应 Activity/View/Dialog 
        TypeElement typeElement = entry.getKey();
        //获取 TypeElement 对应的 BindingSet，BindingSet 就是这个 TypeElement 中所有的视图绑定，包括字段(@BindView)和方法(@onClick)
        BindingSet binding = entry.getValue();
        //通过 BindingSet.brewJava() 生成 JavaFile
        JavaFile javaFile = binding.brewJava(sdk, debuggable);
        try {
            //通过 javaFile.writeTo() 生成源代码文件 
            javaFile.writeTo(filer);
        } catch (IOException e) {
           //...
        }
    }
    return false;
  }
```      

先来了解一下涉及到的几个类：
- `TypeElement`：`javax.lang.model.element`中的类，代表编程元素中的类或者接口，它提供了一些方法用于访问内部成员（字段、方法等）的信息，在这里它就代表包含视图绑定的 `Activity/Dialog/View`  
- `BindingSet`：一个`TypeElement`（比如一个`Activity`）对应的所有绑定元素的集合，它有一个 `BindingSet` 类型的 `parent` 的字段。这个类的实例可通过 `BindingSet.Builder` 创建

`process()` 的逻辑就是为每个 `TypeElment` 生成 `BindingSet`，再通过 `BindingSet.brewJava()` 生成 Java 源代码，也就是那些 `_ViewBinding`类。    
>`brew`的中文意思是「酿制」

生成 `BindingSet` 的逻辑就在 `findAndParseTarget()` 方法中，这里只分析 `@BindView` 这种情况。  

- `ButterKnifeProcessor#findAndParseTarget()`    
```java
private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
    //builderMap 存储 TypeElement 对应的 BindingSet.Builder
    //BindingSet.Builder在 build() 之后就会生成 BindingSet
    Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
    
    //已处理完毕的 TypeElement 集合
    Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();

    //...初始化

    //...处理其他绑定(@BindAnim @BindInt...)
    
    //处理 @BindView 修饰的元素
    for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
        //对所有@BindView修饰的元素调用 parseBindView() 方法
        parseBindView(element, builderMap, erasedTargetNames);
    }

    //...处理其他绑定
    
    //由 builderMap 创建队列 entries
    Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries =
        new ArrayDeque<>(builderMap.entrySet());
    //创建要返回的 bindingMap
    Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
    
    //遍历 entries 队列
    while (!entries.isEmpty()) {
        //获取队头元素 entry
        Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();
        //获取 type 和对应的 builder
        TypeElement type = entry.getKey();
        BindingSet.Builder builder = entry.getValue();
        //获取父元素
        TypeElement parentType = findParentType(type, erasedTargetNames);
        
        if (parentType == null) {
            //没有父元素,直接通过 builder.build() 生成 BindingSet,然后放入 bindingMap
            bindingMap.put(type, builder.build());
        } else {
            //从 bindingMap 寻找父元素对应的 BindingSet
            BindingSet parentBinding = bindingMap.get(parentType);
            if (parentBinding != null) {
                //为 builder 设置父元素
                builder.setParent(parentBinding);
                //build()之后添加到 bindingMap 中
                bindingMap.put(type, builder.build());
            } else {
                //父类型对应的 BindingSet.Builder 还没被 build，重新加入队列稍后处理
                entries.addLast(entry);
            }
        }
    }
    return bindingMap;
}
```   

这个方法有点长，可以先不管后面的部分，只需要知道这里有一个 `builderMap`，它保存了 `TypeElement` 与 `BindingSet.Builder` 的映射。    

方法中对每个用 `@BindView` 修饰的元素都调用了 `parseBindView()` 方法，来看一下这个方法。
- `ButterKnifeProcessor#parseBindView()`
```java
private void parseBindView(Element element, 
    Map<TypeElement，BindingSet.Builder> builderMap,
    Set<TypeElement> erasedTargetNames) {

    //获取 enclosingElement,就是被 @BindView 修饰的元素的外层元素
    //比如 Activity 中有个 Button 被 @BindView 修饰，那么这里的 
    //enclosingElement 就是 Activity
    TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

    //elementType主要用于判断 element 是否继承自 View
    TypeMirror elementType = element.asType();
    //....

    //获取 enclosingElement 的带包名的类名,比如 "com.wenhaiz.Test.MainActivity"
    Name qualifiedName = enclosingElement.getQualifiedName();
    //获取元素的类名，比如"Button"
    Name simpleName = element.getSimpleName();
    
    //获取绑定 View 的 id，就是在 @BindView 中传入的 id
    int id = element.getAnnotation(BindView.class).value();

    //为 element 生成 QualifiedId(包含包名和 id)
    QualifiedId qualifiedId = elementToQualifiedId(element, id);

    //从 builderMap 中获取 enclosingElement 对应的 BindingSet.Builder 实例
    BindingSet.Builder builder = builderMap.get(enclosingElement);
    
    if (builder != null) {
        //判断是否重复绑定
        String existingBindingName = builder.findExistingBindingName(getId(qualifiedId));
        if (existingBindingName != null) {
            //重复绑定，输出 log 信息
        return;
        }
    } else {
        //builderMap 中没有找到 enclosingElement 对应的 BindingSet.Builder
        //则获取或者创建一个实例，并加入到 builderMap 中
        builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
    }

    String name = simpleName.toString();
 
    //判断字段是否必须
    boolean required = isFieldRequired(element);
    //为 builder 添加字段
    builder.addField(getId(qualifiedId), new FieldViewBinding(name, type, required));
    //将 enclosingElement 添加到已处理集合中
    erasedTargetNames.add(enclosingElement);
}
```
>方法有点长，我省略了不必要的代码，注意结合我的注释看，不要着急。  

`parseBindView()` 的主要逻辑就是为 `enclosingElement` 创建对应的  `builder` ，然后为 `builder` 添加 `element` 绑定字段，然后将 `builder` 添加到 `builderMap` 中。   

需要注意这里涉及到两个 `TypeElement`，一个是`element`,一个是`enclosingElement`。   
`element` 对应 `Activity/View/Dialog` 中用 `@BindView` 修饰的 **字段** ，而 `enclosingElement` 就对应 `Activity/View/Dialog`，还要注意 `BindingSet.Builder` 是与 `enclosingElement` 对应的。      

下面是 `parseBindView()` 中涉及到的几个方法。
- `ButterKnifeProcessor#elementToQualifiedId()` 由 element 的包名和 id 生成一个 QualifiedId
```java
private QualifiedId elementToQualifiedId(Element element, int id) {
    return new QualifiedId(elementUtils.getPackageOf(element).getQualifiedName().toString(), id);
}
```
- `ButterKnifeProcessor#getId()` 通过 `QualifiedId` 获取 `Id`
```java
private Id getId(QualifiedId qualifiedId) {    
    //private final Map<QualifiedId, Id> symbols = new LinkedHashMap<>();
    //生成id并添加进symbols
    if (symbols.get(qualifiedId) == null) {
        symbols.put(qualifiedId, new Id(qualifiedId.id));
    }
    return symbols.get(qualifiedId);
  }
```
- `BindingSet.Builder#findExistingBindingName()`  寻找 `id` 对应的 `builder`，并获取 `builder` 的 `fieldBinding` 的名称
```java
String findExistingBindingName(Id id) {
       
    ViewBinding.Builder builder = viewIdMap.get(id);
    if (builder == null) {
        return null;
    }
    FieldViewBinding fieldBinding = builder.fieldBinding;
    if (fieldBinding == null) {
        return null;
    }
    return fieldBinding.getName();
}
```   
- `ButterKnifeProcessor#getOrCreateBindingBuilder()` 获取 `enclosingElement` 对应的 `BindingSet.Builder`，如果不存在，就创建一个
```java
private BindingSet.Builder getOrCreateBindingBuilder(
    Map<TypeElement, BindingSet.Builder> builderMap, TypeElement enclosingElement) {
    //从 builderMap 中获取 builder
    BindingSet.Builder builder = builderMap.get(enclosingElement);
    //如果 builderMap 中不存在，则为 enclosingElement 创建一个 builder 并添加到 builderMap 中
    if (builder == null) {
        builder = BindingSet.newBuilder(enclosingElement);
        builderMap.put(enclosingElement, builder);
    }
    return builder;
}
```
- `ButterKnifeProcessor#isFieldRequired`  判断字段是否是必须的，如果字段被 `@Nullable` 修饰，则返回 `false`
```java
private static boolean isFieldRequired(Element element) {  
    //private static final String NULLABLE_ANNOTATION_NAME = "Nullable";
    return !hasAnnotationWithName(element, NULLABLE_ANNOTATION_NAME);
  }
```
- `BindingSet.Builder#addField()` 为 `BindingSet.Builder` 添加一个绑定字段
```java
void addField(Id id, FieldViewBinding binding) {
    //生成 ViewBinding.Buider 并添加字段
    getOrCreateViewBindings(id).setFieldBinding(binding);
}
```
`addField()` 方法又调用了如下两个方法。
- `BindingSet#getOrCreateViewBindings()`
```java
private ViewBinding.Builder getOrCreateViewBindings(Id id) {
    //private final Map<Id, ViewBinding.Builder> viewIdMap = new LinkedHashMap<>();
    
    //先从 viewIdMap 中获取buider,如果没有就创建一个添加到 viewId 中
    ViewBinding.Builder viewId = viewIdMap.get(id);
    if (viewId == null) {
        viewId = new ViewBinding.Builder(id);
        viewIdMap.put(id, viewId);
    }
    return viewId;
}
```
- `ViewBinding.Builder#setFieldBinding()`
```java
 public void setFieldBinding(FieldViewBinding fieldBinding) {
    if (this.fieldBinding != null) {
        throw new AssertionError();
    }
    this.fieldBinding = fieldBinding;
}
```    

这里涉及到了两个类：
- `ViewBinding`: 代表一个 `View` 上的所有绑定，包括绑定的字段、方法，可以通过 `ViewBinding.Builder` 创建实例
- `FieldViewBinding`：表示一个字段与 `View` 之间的绑定 。
>除此之外，ButterKnife 中还有 `FieldAnimationBinding`，`FieldDrawableBinding`，`MethodViewBinding` 等表示绑定的类。   

现在调用栈有点深，涉及的方法也多了，我们现在还在 `parseBindView()`方法中，上面这些方法都是 `parseBinidView()` 中用到的，大致知道他们做了什么工作就可以了。    


到此，`parseBindView()` 方法执行完成，对所有`@BindView`修饰的元素执行此方法后，`builderMap` 中就存储了 `enclosingElement`(一定要注意是 `enclosingElement` ) 与 `BindingSet.Builder` 的映射。   

接下来就又回到了`findAndParseTarget()`方法中，可以继续往下进行了，代码如下：
- `ButterKnifeProcessor#findAndParseTarget()`    
```java
private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
    //.....
    //buiderMap 已经通过 parseBindView() 等方法生成完毕

    //由 builderMap 创建队列 entries
    Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries =
        new ArrayDeque<>(builderMap.entrySet());

    //创建要返回的 bindingMap
    Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
    
    //遍历 entries 队列
    while (!entries.isEmpty()) {
        //获取队头元素 entry
        Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();
        //获取 type 和对应的 builder
        //这里的 type 就是之前提到的 enclosingElement
        TypeElement type = entry.getKey();
        BindingSet.Builder builder = entry.getValue();
        //获取 type 的父元素
        TypeElement parentType = findParentType(type, erasedTargetNames);
        
        if (parentType == null) {
            //没有父元素,直接通过 builder.build() 生成 BindingSet,然后放入 bindingMap
            bindingMap.put(type, builder.build());
        } else {
            //从 bindingMap 寻找父元素对应的 BindingSet
            BindingSet parentBinding = bindingMap.get(parentType);
            if (parentBinding != null) {
                //找到后为 builder 设置父元素
                builder.setParent(parentBinding);
                //build()之后添加到 bindingMap 中
                bindingMap.put(type, builder.build());
            } else {
                //父类型对应的 BindingSet为空，也就是 BindingSet.Builder 还没被 build
                //重新加入队列稍后处理
                entries.addLast(entry);
            }
        }
    }
    return bindingMap;
}
```
方法中涉及到了 `BindingSet.Builder.setParent()`，上面介绍 `Bindingset` 的时候也提到了，它有一个 `parent`字段，这个方法就是为这个字段赋值。
- `BindingSet.Builder#setParent()`
```java
void setParent(BindingSet parent) {
     this.parentBinding = parent;
}  
```    

这样`findAndParseTargets()` 返回，我们就拿到了 `bindingMap`，它存储了所有 `enclosingElement` 以及对应的 `BindingSet`。   

终于可以回到 `process()` 方法中了。
- `ButterKnifeProcessor#process()`
```java
@Override 
public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
    //通过 findAndParseTarget() 生成绑定映射 bindingMap
    Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);
    
    //遍历所有 TypeElement-BindingSet 键值对    
    for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
        //获取 TypeElement
        TypeElement typeElement = entry.getKey();
        //获取 TypeElement 对应 BindingSet
        BindingSet binding = entry.getValue();
        //通过 BindingSet.brewJava() 生成 JavaFile
        JavaFile javaFile = binding.brewJava(sdk, debuggable);
        try {
            //通过 javaFile.writeTo() 生成源代码文件 
            javaFile.writeTo(filer);
        } catch (IOException e) {
           //...
        }
    }
    return false;
  }
```
然后看一下`BindingSet.brewJava()`方法。

- `BindingSet#brewJava()`
```java
JavaFile brewJava(int sdk, boolean debuggable) {
    //使用 JavaPoet 生成 JavaFile
    return JavaFile.builder(bindingClassName.packageName(), createType(sdk, debuggable))
        .addFileComment("Generated code from Butter Knife. Do not modify!")
        .build();
        //可以看到添加的注释刚好就是生成的_ViewBinding.java 中的第一句
  }
```
`brewJava()`方法使用了 JavaPoet 框架来生成 Java 源代码。
> [JavaPoet](https://github.com/square/javapoet) 也是 [Square](https://github.com/square) 出品的 Java 源码生成框架。   

这里有一个`createType()` 方法，代码如下

- `BindingSet#createType()`  
>`TypeSpec` 在 `JavaPoet` 中对应一个类或接口    

```java
private TypeSpec createType(int sdk, boolean debuggable) {
    // 添加 public 修饰符
    TypeSpec.Builder result = TypeSpec.classBuilder(bindingClassName.simpleName())
        .addModifiers(PUBLIC);

    //如果需要，则添加 final 修饰符
    if (isFinal) {
        result.addModifiers(FINAL);
    }

    //如果 parentBinding 不为空，添加继承；否则就添加 implements Unbinder
    //parentBinding 就是通过 BindingSet.Builder.setParent() 设置的
    if (parentBinding != null) {
        result.superclass(parentBinding.bindingClassName);
    } else {
        result.addSuperinterface(UNBINDER);
    }

    //添加 target 字段
    if (hasTargetField()) {
        result.addField(targetTypeName, "target", PRIVATE);
    }

    //根据元素类型，添加对应构造方法
    if (isView) {
        result.addMethod(createBindingConstructorForView());
    } else if (isActivity) {
        result.addMethod(createBindingConstructorForActivity());
    } else if (isDialog) {
        result.addMethod(createBindingConstructorForDialog());
    }
    if (!constructorNeedsView()) {
         result.addMethod(createBindingViewDelegateConstructor());
    }

    result.addMethod(createBindingConstructor(sdk, debuggable));

    if (hasViewBindings() || parentBinding == null) {
         result.addMethod(createBindingUnbindMethod(result));
    }
    return result.build();
  }
```
这里主要涉及两个方法：`createBindingConstructorForView()` 和 `createBindingConstructor()`   

- `BindingSet#createBindingConstructorForView()`  
>`MethodSpec` 在 `JavaPoet` 中对应一个 `Method` 元素    

```java
private MethodSpec createBindingConstructorForView() {
    //构造方法用 @UI_Thread 修饰，并且是 public 的
    //这正是刚才在生成的源码中看到的构造方法
    MethodSpec.Builder builder = MethodSpec.constructorBuilder()
        .addAnnotation(UI_THREAD)
        .addModifiers(PUBLIC)
        .addParameter(targetTypeName, "target");
        
    //调用另外的构造方法
    if (constructorNeedsView()) {
        builder.addStatement("this(target, target)");
    } else {
        builder.addStatement("this(target, target.getContext())");
    }
    return builder.build();
  }
```
- `BindingSet#createBindingConstructor()`
```java
private MethodSpec createBindingConstructor(int sdk, boolean debuggable) {
    MethodSpec.Builder constructor = MethodSpec.constructorBuilder()
        .addAnnotation(UI_THREAD)
        .addModifiers(PUBLIC);

    //是否绑定了方法
    if (hasMethodBindings()) {
        constructor.addParameter(targetTypeName, "target", FINAL);
    } else {
        constructor.addParameter(targetTypeName, "target");
    }

    if (constructorNeedsView()) {
        //生成第二个参数为 View 类型的构造方法，就是 MainActivity_ViewBinding.java 中第二个构造方法
        constructor.addParameter(VIEW, "source");
    } else {
        constructor.addParameter(CONTEXT, "context");
    }

    //....
    return constructor.build();
  }
```
`Bindset.brewJava()` 执行完成后，就生成了 `JavaFile` 对象，再通过 `JavaFile#wrtieTo()` 方法，一个 Java 源文件就生成了。   
>有关通过 `JavaPoet` 生成 java 源文件的详细内容可以了解一下 [JavaPoet](https://github.com/square/javapoet)。

到此，`ButterKnife` 为我们生成源代码的原理就分析完了。    
现在再来简单梳理一下思路：   
1. 获得所有用 `@BindView` 修饰的元素，他们都有对应的 `enclosingElement`(可以是 Activity/Dialog/View)
2. 为所有`enclosingElement` 生成与之对应的 `BindingSet` 对象，它保存了这个`enclosingElment` 中所有的绑定信息
3. 通过 `BindingSet.brewJava()`方法，借助开源框架 `JavaPoet` 来生成文件名以`_ViewBinding`为后缀的 `Java` 源代码。

## 总结
本文从 `ButterKnife` 的简单使用出发，进行了原理说明和源码分析，也对 `ButterKnife` 注解处理过程进行了简要介绍。需要注意的是，本文主要针对的是绑定 `View` 的情况，其他绑定（如资源、方法等）原理大致相同。   

`ButterKnife` 是个实用又小巧的开源框架，它的代码量没有很多，非常适合作为练习阅读源码能力的项目，也建议大家可以去看一看，肯定会有很大的收获。   

最后，还是感谢大神 Jake Wharthon 吧。(完)
