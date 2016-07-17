# Gson通过借助TypeToken获取泛型参数的类型的方法

来源:[http://blog.csdn.net/zzp_403184692/article/details/8266575](http://blog.csdn.net/zzp_403184692/article/details/8266575)

最近在使用Google的Gson包进行Json和Java对象之间的转化，对于包含泛型的类的序列化和反序列化Gson也提供了很好的支持，感觉有点意思，就花时间研究了一下。

由于Java泛型的实现机制，使用了泛型的代码在运行期间相关的泛型参数的类型会被擦除，我们无法在运行期间获知泛型参数的具体类型（所有的泛型类型在运行时都是Object类型）。

但是有的时候，我们确实需要获知泛型参数的类型，比如将使用了泛型的Java代码序列化或者反序列化的时候，这个时候问题就变得比较棘手。

``` 
class Foo<T> {
  T value;
}
Gson gson = new Gson();
Foo<Bar> foo = new Foo<Bar>();
gson.toJson(foo); // May not serialize foo.value correctly
gson.fromJson(json, foo.getClass()); // Fails to deserialize foo.value as Bar
```

对于上面的类`Foo<T>`，由于在运行期间无法得知T的具体类型，对这个类的对象进行序列化和反序列化都不能正常进行。Gson通过借助TypeToken类来解决这个问题。

```
TestGeneric<String> t = new TestGeneric<String>();
t.setValue("Alo");
Type type = new TypeToken<TestGeneric<String>>(){}.getType();
  
String gStr = GsonUtils.gson.toJson(t,type);
System.out.println(gStr);
TestGeneric t1 = GsonUtils.gson.fromJson(gStr, type);
System.out.println(t1.getValue());
 
```

TypeToken的使用非常简单，如上面的代码，只要将需要获取类型的泛型类作为TypeToken的泛型参数构造一个匿名的子类，就可以通过getType()方法获取到我们使用的泛型类的泛型参数类型。

下面来简单分析一下原理。

要获取泛型参数的类型，一般的做法是在使用了泛型的类的构造函数中显示地传入泛型类的Class类型作为这个泛型类的私有属性，它保存了泛型类的类型信息。

```
public class Foo<T>{
 
 public Class<T> kind;
 
 public Foo(Class<T> clazz){
  this.kind = clazz;
 }
 
 public T[] getInstance(){
  return (T[])Array.newInstance(kind, 5);
 }
 
 public static void main(String[] args){
  Foo<String> foo = new Foo(String.class);
  String[] strArray = foo.getInstance();
 }
}
```

这种方法虽然能解决问题，但是每次都要传入一个Class类参数，显得比较麻烦。Gson库里面对于这个问题采用了了另一种解决办法。

同样是为了获取Class的类型，可以通过另一种方式实现：

```
public abstract class Foo<T>{
 
	Class<T> type;
 
 	public Foo(){
		this.type = (Class<T>) getClass();
	}
	
	public static void main(String[] args) {
		Foo<String> foo = new Foo<String>(){};
		Class mySuperClass = foo.getClass();
	}
}
```

声明一个抽象的父类Foo，匿名子类将泛型类作为Foo的泛型参数传入构造一个实例，再调用getClass方法获得这个子类的Class类型。

这里虽然通过另一种方式获得了匿名子类的Class类型，但是并没有直接将泛型参数T的Class类型传进来，那又是如何获得泛型参数的类型的呢，这要依赖Java的Class字节码中存储的泛型参数信息。Java的泛型机制虽然在运行期间泛型类和非泛型类都相同，但是在编译java源代码成class文件中还是保存了泛型相关的信息，这些信息被保存在class字节码常量池中，使用了泛型的代码处会生成一个signature签名字段，通过签名signature字段指明这个常量池的地址。

关于class文件中存储泛型参数类型的具体的详细的知识可以参考这里：[http://stackoverflow.com/questions/937933/where-are-generic-types-stored-in-java-class-files](http://stackoverflow.com/questions/937933/where-are-generic-types-stored-in-java-class-files)

JDK里面提供了方法去读取这些泛型信息的方法，再借助反射，就可以获得泛型参数的具体类型。同样是对于第一段代码中的foo对象，通过下面的代码可以得到foo<T>中的T的类型：

```
Type mySuperClass = foo.getClass().getGenericSuperclass();
Type type = ((ParameterizedType)mySuperClass).getActualTypeArguments()[0];
System.out.println(type);
```

运行结果是class java.lang.String。

分析一下这段代码，Class类的getGenericSuperClass()方法的注释是：

> Returns the Type representing the direct superclass of the entity (class, interface, primitive type or void) represented by thisClass.
> 
> If the superclass is a parameterized type, the Type object returned must accurately reflect the actual type parameters used in the source code. The parameterized type representing the superclass is created if it had not been created before. See the declaration of ParameterizedType for the semantics of the creation process for parameterized types. If thisClass represents either theObject class, an interface, a primitive type, or void, then null is returned. If this object represents an array class then theClass object representing theObject class is returned

概括来说就是对于带有泛型的class，返回一个ParameterizedType对象，对于Object、接口和原始类型返回null，对于数组class则是返回Object.class。ParameterizedType是表示带有泛型参数的类型的Java类型，JDK1.5引入了泛型之后，Java中所有的Class都实现了Type接口，ParameterizedType则是继承了Type接口，所有包含泛型的Class类都会实现这个接口。

实际运用中还要考虑比较多的情况，比如获得泛型参数的个数避免数组越界等，具体可以参看Gson中的TypeToken类及ParameterizedTypeImpl类的代码。



