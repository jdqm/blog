---
title: java注解(1)
date: 2017-04-11 16:18:53
updated: 2017-04-11 16:18:53
tags:
	- Java注解
	- 元注解
categories: java
---

## 1、Java提供的5个基本Annotation

- @Override
用于限定重写父类方法，在编译时检查父类是否包含被重写的方法。主要帮助程序员避免一些低级错误。只能修饰方法（修饰范围将在后面的元注解中解释）。
<!-- more -->
```
public class Fruit {
	
	public void info(){
		System.out.println("水果的Info方法...");
	}

}

class Apple extends Fruit{

	@Override
	public void info() {
		System.out.println("重写水果的Info方法...");
	}
	
}
```
上面的例子Apple类继承了Fruit类并重写了Fruit中的info()方法，其中@Override是可选的，可以不写。写上的好处有两个:
①告诉阅读代码的人这个方法是重写父类的方法；
②避免开发人员本想重写父类方法，确因拼写错误而导致错误，例如info()写成inf0(),这种错误后期也是很难发现的。

- @Deprecated
用于表示某个程序元素已过时，不推荐使用，后续迭代版本有可能去掉元素，当其他程序使用已过时的类、方法时编译器就会发出警告。

![方法过时eclipse提示效果截图][1]

上面的例子Apple类的info()方法使用@Deprecated标记，在使用时eclipse会将方法名加上中划线告诉开发人员此方法已过时。细心的读者肯定会注意到在文档注释中也有一个@deprecated，它们的作用基本相同，但是前者是jdk5才支持的注解，例如：
```
class Apple extends Fruit{
	
	/**
	 * @deprecated 此方法以过时, 由 Apple.info2() 替代
	 */
	@Deprecated
	@Override
	public void info() {
		System.out.println("重写水果的Info方法...");
	}
	
	public void info2() {
		System.out.println("苹果的Info方法...");
	}
	
}
```

可以理解为 @deprecated 是给javac.exe生成文档使用的，而 @Deprecate 则不是。



- @SuppressWarnings
抑制警告，如下代码myList没有声明泛型，会有警告，使用@SuppressWarnings("rawtypes")这个注解后警告就不再提示了。
```
@SuppressWarnings("rawtypes")
public class SupressWarningsTest {
	
	public static void main(String[] args) {
		List myList = new ArrayList<>();
		System.out.println(myList);
	}
}
```
这个注解个人不推荐使用，警告本身说明程序可能会存在风险，尽量通过实际手段消除警告。

- @SafeVarags（Java7新增）

先看一个例子
```
//不使用泛型
List list = new ArrayList<Integer>();
		
//添加新元素引发 unchecked 异常 
list.add(20);
		
//下面代码引起“未经检查的转换”的警告，编译、运行时完全正常，
List<String> ls = list; //①
		
//但是访问ls里的元素，如下面的代码就会引起运行时异常 java.lang.ClassCastException
System.out.println(ls.get(0));
```
Java把引发这种错误的原因称为“堆污染”（heap pollution），当把一个不带泛型的的对象赋值给一个带泛型的变量时，往往就会发生这种“堆污染”，如上面带①的行。

例子：
```
@SafeVarargs // 不是真的安全
static void m(List<String>... stringLists) {
	Object[] array = stringLists;
	List<Integer> tmpList = Arrays.asList(42);
	array[0] = tmpList; // 语义是无效的, 但编译器无警告
	String s = stringLists[0].get(0); // Oh no, ClassCastException at runtime!
}
```

- @FunctionalInterface（Java8新增）

Java8中规定，只有一个抽象方法的接口（可以有其他的方法，但是抽象方法只有一个），称为函数式接口。@FunctionalInterface就是指定某个接口必须是函数式接口。函数式接口就是为Java8 中的Lamda表达式准备的。

```
@FunctionalInterface
public interface FunInterface {
	
	static void foo(){
		System.out.println("foo");
	}
	
	default void bar(){
		System.out.println("bar");
	}
	
	//这个接口中唯一的抽象方法
	void test();
}

```

## 2、JDK的元注解
一般地，称修饰数据的数据为元数据，自然元注解是用来修饰注解的。

- @Retation
该注解只能用来修饰Annotation的定义，用于指定被修饰的Annotation可以保留多长时间。

|取值|含义|
|:--|:--|
|RetentionPolicy.SOURCE|只保留在源码中，编译器直接丢弃这种类型的注解|
|RetentionPolicy.CLASS|编译器把这类注解保留在class字节码文件中，当运行Java程序时，JVM不可获取这类注解信息，这个默认值|
|RetentionPolicy.RUNTIME|编译器把这类注解保留在class字节码文件中，当运行Java程序时，JVM可以获取这类注解信息，程序可以通过反射获取这类注解的信息|

如果需要通过反射获取注解的信息，需要value属性的值为RetentionPolicy.RUNTIME的@Retation修饰的注解，可以采用如下代码为value赋值。
```
@Retention(value=RetentionPolicy.RUNTIME)
```
或者
```
@Retention(RetentionPolicy.RUNTIME)
```

- @Target
该注解也只能修饰Annotation的定义，用于指定被修饰的Annotation能修饰哪些程序单元，@Target也包含一个名为value的成员变量，该变量的取值只能是如下几个。


|取值|可修饰单元类型|
|:--|:--|
| ElementType.TYPE|类, 接口 (包括注解类型), 或者枚举定义|
| ElementType.FIELD|成员变量 (包括枚举常量)|
| ElementType.METHOD|方法|
| ElementType.PARAMETER|参数|
| ElementType.CONSTRUCTOR|构造函数|
| ElementType.LOCAL_VARIABLE|局部变量|
| ElementType.ANNOTATION_TYPE|注解类型|
| ElementType.PACKAGE|包|
| ElementType.TYPE_PARAMETER|参数类型, since 1.8|
| ElementType.TYPE_USE|Use of a type， since 1.8|


- @Documented

如果使用了@Documented修饰的注解，该注解会被javac工具提取成文档，API文档中将包含该注解的说明。

- @Inherited
使用@Inherited修饰的注解具有继承性。举一个例子

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface Inheritable {
	
}
```

```
//使用 @Inheritable修饰的基类
@Inheritable
class Base{
	
}

//InheritableTest,只是继承了Base类，并未直接使用@Inheritable这个注解修饰
public class InheritableTest extends Base{
	
	public static void main(String[] args) {
		
		boolean result = InheritableTest.class.isAnnotationPresent(Inheritable.class);
		//打印InheritableTest这个类是否有@Inheritable修饰
		System.out.println(result);
	}
	
}
```
上面的例子结果为 true，InheritableTest通过继承Base类，把Base上的注解@Inheritable也继承了。今天到此为止，下一篇将介绍自定义注解和注解处理器。


  [1]: https://raw.githubusercontent.com/jdqm/hello-world/master/annotation/1.png