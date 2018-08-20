---
title: java程序员--小心你代码中的内存泄漏
date: 2016-06-19 23:14:48
categories: effective java
tags: [内存泄漏]
---

当你从c&c++转到一门具有垃圾回收功能的语言时，程序员的工作就会变得更加容易，因为你用完对象，他们会被自动回收，但是，java程序员真的不需要考虑内存泄露吗？ 其实不然
## 1.举个例子-看你能否找出内存泄漏
<!--more-->
```java
import java.util.Arrays;

public class Stack {
	private Object[] elements;
	private int size = 0;
	private static final int DEFAULT_INITIAL_CAPACITY = 16;

	public Stack() {
		elements = new Object[DEFAULT_INITIAL_CAPACITY];
	}

	public void push(Object e) {
		ensureCapacity();
		elements[size++] = e;
	}

	public Object pop() {
		if (size == 0)
			throw new EmptyStackException();
		return elements[--size];
	}

	private void ensureCapacity() {
		if (elements.length == size)
			elements = Arrays.copyOf(elements, 2 * size + 1);
	}
}
```
### 1.1原因分析
上述程序并没有明显的错误，但是这段程序有一个内存泄漏，随着GC活动的增加，或者内存占用的不断增加，程序性能的降低就会表现出来，严重时可导致内存泄漏，但是这种失败情况相对较少。
代码的主要问题在pop函数，下面通过这张图示展现
假设这个栈一直增长，增长后如下图所示
!![这里写图片描述](http://pdqo4ie3j.bkt.clouddn.com/18-8-20/73672035.jpg)

当进行大量的pop操作时，由于引用未进行置空，gc是不会释放的，如下图所示
![这里写图片描述](http://pdqo4ie3j.bkt.clouddn.com/18-8-20/16777312.jpg)

从上图中看以看出，如果栈先增长，在收缩，那么从栈中弹出的对象将不会被当作垃圾回收，即使程序不再使用栈中的这些队象，他们也不会回收，因为栈中仍然保存这对象的引用，俗称过期引用，这个内存泄露很隐蔽。
### 1.2解决方法

```java
public Object pop() {
    if (size == 0)
	throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null;
    return result;
}
```
一旦引用过期，清空这些引用，将引用置空。
![这里写图片描述](http://pdqo4ie3j.bkt.clouddn.com/18-8-20/53533093.jpg)

## 2.缓存泄漏
内存泄漏的另一个常见来源是缓存，一旦你把对象引用放入到缓存中，他就很容易遗忘，对于这个问题，可以使用WeakHashMap代表缓存，此种Map的特点是，当除了自身有对key的引用外，此key没有其他引用那么此map会自动丢弃此值
### 2.1代码示例
```java
/**
 * Created by liuroy on 2017/2/25.
 */
import java.util.HashMap;
import java.util.Map;
import java.util.WeakHashMap;
import java.util.concurrent.TimeUnit;

public class Test {
    static Map wMap = new WeakHashMap();
    static Map map = new HashMap();
    public static void init(){
        String ref1= new String("obejct1");
        String ref2 = new String("obejct2");
        String ref3 = new String ("obejct3");
        String ref4 = new String ("obejct4");
        wMap.put(ref1, "chaheObject1");
        wMap.put(ref2, "chaheObject2");
        map.put(ref3, "chaheObject3");
        map.put(ref4, "chaheObject4");
        System.out.println("String引用ref1，ref2，ref3，ref4 消失");

    }
    public static void testWeakHashMap(){

        System.out.println("WeakHashMap GC之前");
        for (Object o : wMap.entrySet()) {
            System.out.println(o);
        }
        try {
            System.gc();
            TimeUnit.SECONDS.sleep(20);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        System.out.println("WeakHashMap GC之后");
        for (Object o : wMap.entrySet()) {
            System.out.println(o);
        }
    }
    public static void testHashMap(){
        System.out.println("HashMap GC之前");
        for (Object o : map.entrySet()) {
            System.out.println(o);
        }
        try {
            System.gc();
            TimeUnit.SECONDS.sleep(20);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        System.out.println("HashMap GC之后");
        for (Object o : map.entrySet()) {
            System.out.println(o);
        }
    }
    public static void main(String[] args) {
        init();
        testWeakHashMap();
        testHashMap();
    }
}
/** 结果
String引用ref1，ref2，ref3，ref4 消失
WeakHashMap GC之前
obejct2=chaheObject2
obejct1=chaheObject1
WeakHashMap GC之后
HashMap GC之前
obejct4=chaheObject4
obejct3=chaheObject3
Disconnected from the target VM, address: '127.0.0.1:51628', transport: 'socket'
HashMap GC之后
obejct4=chaheObject4
obejct3=chaheObject3
**/
```
![这里写图片描述](http://pdqo4ie3j.bkt.clouddn.com/18-8-20/72596373.jpg)
上面代码和图示主演演示WeakHashMap如何自动释放缓存对象，当init函数执行完成后，局部变量字符串引用weakd1,weakd2,d1,d2都会消失，此时只有静态map中保存中对字符串对象的引用，可以看到，调用gc之后，hashmap的没有被回收，而WeakHashmap里面的缓存被回收了。
## 监听器和回调
内存泄漏第三个常见来源是监听器和其他回调，如果客户端在你实现的API中注册回调，却没有显示的取消，那么就会积聚。需要确保回调立即被当作垃圾回收的最佳方法是只保存他的若引用，例如将他们保存成为WeakHashMap中的键。