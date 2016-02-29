# Android Annotation基础教程

来源:[阿里云](https://yq.aliyun.com/articles/5561?spm=5176.100240.searchblog.18.2BREHl)

> 在Android源码中，越来越多地使用到了Annotation，我们有必要从头学习一下Annotation的基础知识和在Android中的应用

## Java Annotation

Java 1.5中开始引入的Annotation，类似于注释的一种技术，参考了一些网上的译法，姑且译成注解吧。

我们在开发中，用得最多的Annotation莫过于@Override了。大家天天用，可能很多同学却没有关注过其背后的细节，我们看一下它的定义：
```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```
在Android源码中，这个注解定义在`libcore/luni/src/main/java/java/lang/Override.java`中。

这是一个标称注解，只在源代码级有效，主要被编译器用来判断是否真的继承了父类中的方法。

常用的Annotation还有著名的@Deprecated，过时的建议不用的方法。

它的定义如下：

```
@Documented
@Retention(RetentionPolicy.RUNTIME)
public @interface Deprecated {
}
```
另外还有通知编译器不要做警告的`@SuppressWarnings`

```
@Target( { ElementType.TYPE, ElementType.FIELD, ElementType.METHOD,
        ElementType.PARAMETER, ElementType.CONSTRUCTOR,
        ElementType.LOCAL_VARIABLE })
@Retention(RetentionPolicy.SOURCE)
public @interface SuppressWarnings {

    /**
     * The list of warnings a compiler should not issue.
     */
    public String[] value();
}
```

## 元注解
除了上面几个常用的注解定义于最基本的java.lang包中，用于实现这几个Annotation的Annotation都实现在java.lang.annotation包中。它们是被用于实现其它注解所用的。

### 元素类型 - ElementType
就是这个注解可以用于什么语法单元，比如@Override只能用于方法，方法的类型就是ElementType.METHOD.
这是一个枚举，包括下面的类型：

* TYPE: 类，接口，枚举
* FIELD: 域变量
* METHOD:方法
* PARAMETER:函数参数
* CONSTRUCTOR:构造函数
* LOCAL_VARIABLE:局部变量
* ANNOTATION_TYPE:注解类型
* PACKAGE:包

### @Target

定义了元素类型，就可以通过这些类型来指定注解适用的类型了。

```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target {
    ElementType[] value();
}
```

@Target注解就是一个ElementType的数组，就像上面我们看到的用法：

```
@Target( { ElementType.TYPE, ElementType.FIELD, ElementType.METHOD,
        ElementType.PARAMETER, ElementType.CONSTRUCTOR,
        ElementType.LOCAL_VARIABLE })
```

### @Documented

```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Documented {
}
```

用于描述一个注解是可以生成JavaDoc的，也暗示了这个注解是一个公开的API。

### @Retention

这是4个元注解中最重要的一个，用于定义这个注解的生命周期。

取值是另一个枚举：

* RetentionPolicy.SOURCE:注解只存在于源代码中，编译器生成class代码时就忽略了
* RetentionPolicy.CLASS:会编译进class文件，但是VM执行时调不到
* RetentionPolicy.RUNTIME:在运行时也可以访问到

RetentionPolicy的定义如下：

```
public enum RetentionPolicy {
    /**
     * Annotation is only available in the source code.
     */
    SOURCE,
    /**
     * Annotation is available in the source code and in the class file, but not
     * at runtime. This is the default policy.
     */
    CLASS,
    /**
     * Annotation is available in the source code, the class file and is
     * available at runtime.
     */
    RUNTIME
}
```
有了这些基础，再看@Retention的实现，就可以完全看得懂了：

```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Retention {
    RetentionPolicy value();
}
```

@Retention本身是个运行时可用的注解，公开的API，只对注解本身有效。

它只定义了一个RetentionPolicy枚举的值。

## 通过反射处理注解

所有的@Target可用的对象都支持用getAnnotations()方法去读取注解。
例如，读取一个类的注解：
```
Class clazz = ThreadSafeCounter.class;
Annotation[] as = clazz.getAnnotations();
```
我们通过一个例子来说明：

首先先定义一个注解，这个注解用于说明这类或者方法是线程安全的，有一个value用于保存锁对象的名字。

```
import java.lang.annotation.*;

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE,ElementType.METHOD})
public @interface ThreadSafe {
    String value();
}
```

下面定义一个使用该注解的类, 类和其中的一个方法都使用这个注解。其实有点废话，类都线程安全了，方法还能不安全么，呵呵

```
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;

@ThreadSafe("ThreadSafeCounter")
public class ThreadSafeCounter {
    private int mCounter;

    public ThreadSafeCounter(int counter) {
        mCounter = counter;
    }

    @ThreadSafe("this")
    public int incAndGet() {
        synchronized (this) {
            return mCounter++;
        }
    }
```

下面定义一个main方法去通过反射读注解，先读类的注解：

```
public static void main(String[] args){
	Class clazz = ThreadSafeCounter.class;
	Annotation[] as = clazz.getAnnotations();

	for(Annotation a:as){
		ThreadSafe t= (ThreadSafe)a;
		System.out.println("Annotation type="+clazz.getName());
		System.out.println("lock name is:"+t.value());
	}
```
然后再读取

```
Method[] methods = clazz.getMethods();
for(Method method: methods){
	boolean hasAnno = method.isAnnotationPresent(ThreadSafe.class);
	if(hasAnno){
		ThreadSafe anno = method.getAnnotation(ThreadSafe.class);
		System.out.println("method name="+method.getName()+",lock object="+anno.value());
	}
}
```

本章内容参考文献：《Java程序设计完全手册》，王作启，伍正云著，北京：清华大学出版社，2014

## Android中的Annotation

### Android标准的Annotation
#### @Nullable
定义：

```
@Retention(SOURCE)
@Target({METHOD, PARAMETER, FIELD})
public @interface Nullable {
}
```
源码级的，可以用于方法、参数和域，表示一个方法或域的值可以合法地为空，或者是函数的返回值可以合法为空。代码中已经针对为空的情况做了相应的处理。
这是个标称注解。

#### @NonNull
```
@Retention(SOURCE)
@Target({METHOD, PARAMETER, FIELD})
public @interface NonNull {
}
```
与@Nullable相反，@NonNull要求一定不能为空。
####@UiThread
定义：

```
@Retention(SOURCE)
@Target({METHOD,CONSTRUCTOR,TYPE})
public @interface UiThread {
}
```
表示标有该注解的方法或构造函数应该只在UI线程调用。

如果注解元素是一个类，说明该类的所有方法都应该在UI线程中调用。

#### @MainThread

```
@Retention(SOURCE)
@Target({METHOD,CONSTRUCTOR,TYPE})
public @interface MainThread {
}
```

这个是要求运行在主线程的

#### @WorkerThread

```
@Retention(SOURCE)
@Target({METHOD,CONSTRUCTOR,TYPE})
public @interface WorkerThread {
}
```

要求运行在工作线程

#### @IntRef
用于定义整数值。

```
@Retention(CLASS)
@Target({ANNOTATION_TYPE})
public @interface IntDef {
    /** Defines the allowed constants for this element */
    long[] value() default {};

    /** Defines whether the constants can be used as a flag, or just as an enum (the default) */
    boolean flag() default false;
}
```

我们看一个使用@IntRef例子：

```
@IntDef({HORIZONTAL, VERTICAL})
@Retention(RetentionPolicy.SOURCE)
public @interface OrientationMode {}

public static final int HORIZONTAL = 0;
public static final int VERTICAL = 1;
```

在上面定义的@OrientationMode注释中，可以支持的值是HORIZONTAL, VERTICAL.

然后我们再看一个使用flag的例子：
```
74    @IntDef(flag = true,
75            value = {
76                SHOW_DIVIDER_NONE,
77                SHOW_DIVIDER_BEGINNING,
78                SHOW_DIVIDER_MIDDLE,
79                SHOW_DIVIDER_END
80            })
81    @Retention(RetentionPolicy.SOURCE)
82    public @interface DividerMode {}
```

### View相关的

#### @RemoteView
支持RemoteView机制，这是一个运行时的注释.

定义路径：`/frameworks/base/core/java/android/widget/RemoteViews.java`

```
    /**
     * This annotation indicates that a subclass of View is alllowed to be used
     * with the {@link RemoteViews} mechanism.
     */
    @Target({ ElementType.TYPE })
    @Retention(RetentionPolicy.RUNTIME)
    public @interface RemoteView {
    }
```
