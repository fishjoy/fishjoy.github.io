---
title: 变参调用：scala和java的一个不同点
date: 2016-11-11 15:20:02
categories: Scala日常
tags:
- scala
- 变参
---
scala和java几乎没有区别，可以互相调用。注意这里说的是几乎，总有那么少数，出人意料的惊喜在告诉你，scala就是scala。
<!--more-->
# 一个例子
```
import com.alibaba.fastjson.JSON

object test {
  def main(args: Array[String]) = {
    val map = new util.HashMap[CharSequence, CharSequence]()
    map.put("123", "22333")
    map.put("test", null)
    val ret = JSON.toJSONString(map)
    println(ret)
   }
}
```
如上所示，这个例子很简单，把一个jabva的map转换成json字符串。
其中`JSON.toJSONString`的代码如下：
```
public static String toJSONString(Object object) {
    return toJSONString(object, emptyFilters, new SerializerFeature[0]);
}

public static String toJSONString(Object object, SerializerFeature... features) {
    return toJSONString(object, DEFAULT_GENERATE_FEATURE, features);
}
```
问题来了，上面的测试用例报错了：
```
ambiguous reference to overloaded definition,both method toJSONString in object JSON of
type (x$1: Any, x$2: com.alibaba.fastjson.serializer.SerializerFeature*)String
and method toJSONString in object JSON of
type (x$1: Any)String 
match argument types (java.util.HashMap[CharSequence,CharSequence])
val ret = JSON.toJSONString(map)
```
错误的原因很明显，编译器在编译scala调用java依赖包里面的`toJSONString`函数时发生了歧义。
# scala含有变参的重载函数
看一段代码：
```
object Foo {
  def apply (x1: Int): Int = 2
  def apply (x1: Int, x2: Int*): Int = 3
  def apply (x1: Int*): Int = 4

}

object Test11 extends App {
  Console println Foo(7)
  Console println Foo(2, Array(3).toSeq: _*)
  Console println Foo(Array(3, 4).toSeq: _*)
}
```
上述代码分别输出：`2、3、4`
对于前两个构造函数，刚好对应了文章开头的例子，说明，scala调用类似的scala依赖是没有问题的。
我们来注意一下scala如何调用变参：当你使用`Foo(2, 3)`调用时候，会出现歧义，因为`(2, 3)`可以匹配到第二个和第三个构造函数。所以，就只能用看起来很奇怪的写法`Foo(2, Array(3).toSeq: _*)`来进行区分。`Array(3).toSeq: _*`就是在告诉编译器，这个参数是变参的，别匹配错了。
然后，我们来看看Java怎么做的。
# java含有变参的重载函数
代码就不写了，总之，java调用java类似的函数也是没有问题的。
那么问题来了，为什么scala调用java类似的函数就有问题呢？要回答这个问题，我们先来看看java在进行重载调用时，编译器都做了些啥？
>* 调用方法时，能与固定参数函数以及可变参数都匹配时，优先调用固定参数方法。
* 调用方法时，两个变长参数都匹配时，编译无法通过。
```
public class VariVargsTest2 {
  public static void main(String[] args) {
   test("hello"); //1
  }
  public static void test(String ...args)
  {
    System.out.println("变长参数1");
  }
  public static void test(String string,String...args)
  {
    System.out.println("变长参数2");
  }
}
```
当没有1处代码时，程序是能够编译通过的。但是当添加了1处代码后无法通过编译，给出的错误是:`The method test(String[]) is ambiguous for the type VariVargsTest2`。编译器不知道选取哪个方法。

# 总结
从scala和java对含有变参的重载函数处理的方式上，我们可以知道文章开头的代码为什么编译不过。
* scala首先会自己匹配，自己匹配不了的时候，使用者可以手动来标识变参参数。
* java自己匹配，并有一个自己的调用优先级顺序，实在分不清就编译不过了。
回到开头的问题，scala调用java出问题了，就应该是scala编译器和java编译器在处理这个问题时的差异导致的吧。

# 参考文献
[1] [Java语法糖初探（三）--变长参数](https://jssyjam.github.io/2016/05/02/Java%E8%AF%AD%E6%B3%95%E7%B3%96%E5%88%9D%E6%8E%A2%EF%BC%88%E4%B8%89%EF%BC%89-%E5%8F%98%E9%95%BF%E5%8F%82%E6%95%B0-md/)