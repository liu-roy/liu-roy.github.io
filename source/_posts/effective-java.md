---
title: effective java读书笔记
date: 2017-05-24 15:53:24
categories: 读书笔记
tags: effective java
---
---
[TOC]

---
# effective java读书笔记

## 序
去年就把这本javaer必读书——effective java中文版第二版 读完了，第一遍感觉比较肤浅，今年打算开始第二遍，顺便做一下笔记，后续会持续更新。
<!--more-->

## 第2章-创建和销毁对象
### 1、考虑用静态工厂方法替代构造器
优点  

- 静态工厂方法与构造器不同的第一大优势在于，他们有名称，比多个通过不同参数的构造器更具有辨识度。
- 静态工厂方法与构造器不同的第二大优势在于，不必在每次调用他们的时候都创建一个新对象。
- 静态工厂方法与构造器不同的第三大优势在于，他可以返回原返回类型的任何子类型的对象
	- 服务提供者框架。
	```java
	public interface Provider {
		Service newService();
	}
	
	public interface Service {
	// Service-specific methods go here
	}

	public class Services {
		private Services() {
		} // Prevents instantiation (Item 4)

		// Maps service names to services
		private static final Map<String, Provider> providers = new ConcurrentHashMap<String, Provider>();
		public static final String DEFAULT_PROVIDER_NAME = "<def>";

		// Provider registration API
		public static void registerDefaultProvider(Provider p) {
			registerProvider(DEFAULT_PROVIDER_NAME, p);
		}

		public static void registerProvider(String name, Provider p) {
			providers.put(name, p);
		}

		// Service access API
		public static Service newInstance() {
			return newInstance(DEFAULT_PROVIDER_NAME);
		}

		public static Service newInstance(String name) {
			Provider p = providers.get(name);
			if (p == null)
				throw new IllegalArgumentException(
						"No provider registered with name: " + name);
			return p.newService();
		}
	}

	public class Test {
		public static void main(String[] args) {
			// Providers would execute these lines
			Services.registerDefaultProvider(DEFAULT_PROVIDER);
			Services.registerProvider("comp", COMP_PROVIDER);
			Services.registerProvider("armed", ARMED_PROVIDER);

			// Clients would execute these lines
			Service s1 = Services.newInstance();
			Service s2 = Services.newInstance("comp");
			Service s3 = Services.newInstance("armed");
			System.out.printf("%s, %s, %s%n", s1, s2, s3);
		}

		private static Provider DEFAULT_PROVIDER = new Provider() {
			public Service newService() {
				return new Service() {
					@Override
					public String toString() {
						return "Default service";
					}
				};
			}
		};

		private static Provider COMP_PROVIDER = new Provider() {
			public Service newService() {
				return new Service() {
					@Override
					public String toString() {
						return "Complementary service";
					}
				};
			}
		};

		private static Provider ARMED_PROVIDER = new Provider() {
			public Service newService() {
				return new Service() {
					@Override
					public String toString() {
						return "Armed service";
					}
				};
			}
		};
	}
	```
- 静态工厂方法的第四大优势在于，在创建参数化类型实例的时候，它们使代码变得更加简洁。
	- 类型推倒 
	```java
	Map<String, List<String>> m = new HashMap<String, List<String>>();
	            
	public static<K,V> HashMap<K,V> newInstance(){
	 	return new Hash<K,V>();
	}
	Map<String, List<String>> m = HashMap.newInstance();
	```
 
缺点 
- 静态工厂方法的主要缺点在于， 类如果不含公有的或者受保护的构造器，就不能被子类化。
- 第二个缺点 它们与其他的静态方法实际上没有任何区别。
	- valueOf
 	- of
 	- getInstance
 	- newInstance
 	- getType
 	- newType


### 2、遇到多个构造器参数时要考虑用构建器
静态工厂和构造器共同的局限，他们都不能很好的扩展到大量的可选参数

#### 可选参数扩展方法
- 重叠构造器-大量的构造器，客户端编写困难，难以阅读
- javaBean set-构造过程分配到多个调用中，可能造成不一致状态，很难保证线程安全
- Builder模式 (使代码冗长，建议构造器有大量参数时使用)
```java
public class NutritionFacts {
	private final int servingSize;
	private final int servings;
	private final int calories;
	private final int fat;
	private final int sodium;
	private final int carbohydrate;

	public static class Builder {
		// Required parameters
		private final int servingSize;
		private final int servings;

		// Optional parameters - initialized to default values
		private int calories = 0;
		private int fat = 0;
		private int carbohydrate = 0;
		private int sodium = 0;

		public Builder(int servingSize, int servings) {
			this.servingSize = servingSize;
			this.servings = servings;
		}

		public Builder calories(int val) {
			calories = val;
			return this;
		}

		public Builder fat(int val) {
			fat = val;
			return this;
		}

		public Builder carbohydrate(int val) {
			carbohydrate = val;
			return this;
		}

		public Builder sodium(int val) {
			sodium = val;
			return this;
		}

		public NutritionFacts build() {
			return new NutritionFacts(this);
		}
	}

	private NutritionFacts(Builder builder) {
		servingSize = builder.servingSize;
		servings = builder.servings;
		calories = builder.calories;
		fat = builder.fat;
		sodium = builder.sodium;
		carbohydrate = builder.carbohydrate;
	}

	public static void main(String[] args) {
		NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
				.calories(100).sodium(35).carbohydrate(27).build();
	}
}
```

### 3、用私有构造器或者枚举类型强化Singleton属性

- 构造器私有 public static final Elvis INSTANCE = new Elvis(); 可以通过反射调用私有构造器，为了抵御这种攻击，第二次创建该实例的时候抛出异常
- 构造器私有 private static final Elvis INSTANCE = new Elvis()； public static Elvis getInstance() {return INSTANCE;}
- 枚举单例 
	```java
	public enum Elvis {
		INSTANCE;
		public void leaveTheBuilding() {
			System.out.println("Whoa baby, I'm outta here!");
		}

		// This code would normally appear outside the class!
		public static void main(String[] args) {
			Elvis elvis = Elvis.INSTANCE;
			elvis.leaveTheBuilding();
		}
	}
	```
### 4、通过私有构造器强化不可实例化的能力
- 同时也造成了不能子类化。（因为所有子类构造器都必须显式和隐式的调用了超类的构造器）

### 5、避免创建不必要的对象
- String s = new String("stringette"); 反例  -》String s = "stringette";
- 当心无意识的自动装箱拆箱
	```java
	public class Sum {
    	// Hideously slow program! Can you spot the object creation?
    	public static void main(String[] args) {
    	    Long sum = 0L;
    	    //long to Long is not a good idea.
    	    for (long i = 0; i < Integer.MAX_VALUE; i++) {
    	        sum += i;
    	    }
    	    System.out.println(sum);
    	}
	}
    ```
### 6、消除过期的对象引用
- 栈内过期引用
- 缓存
- 监听器和回调函数

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
#### 6.1原因分析
上述程序并没有明显的错误，但是这段程序有一个内存泄漏，随着GC活动的增加，或者内存占用的不断增加，程序性能的降低就会表现出来，严重时可导致内存泄漏，但是这种失败情况相对较少。
代码的主要问题在pop函数，下面通过这张图示展现
假设这个栈一直增长，增长后如下图所示
![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/java-memory-leak/1.jpg)

当进行大量的pop操作时，由于引用未进行置空，gc是不会释放的，如下图所示
![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/java-memory-leak/2.jpg)

从上图中看以看出，如果栈先增长，在收缩，那么从栈中弹出的对象将不会被当作垃圾回收，即使程序不再使用栈中的这些队象，他们也不会回收，因为栈中仍然保存这对象的引用，俗称过期引用，这个内存泄露很隐蔽。
##### 1.2解决方法

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
![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/java-memory-leak/3.jpg)

#### 6.2 缓存泄漏
内存泄漏的另一个常见来源是缓存，一旦你把对象引用放入到缓存中，他就很容易遗忘，对于这个问题，可以使用WeakHashMap代表缓存，此种Map的特点是，当除了自身有对key的引用外，此key没有其他引用那么此map会自动丢弃此值
##### 6.2.1代码示例
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
![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/java-memory-leak/4.jpg)

引用weakd1,weakd2,d1,d2都会消失，此时只有静态map中保存中对字符串对象的引用，可以看到，调用gc之后，hashmap的没有被回收，而WeakHashmap里面的缓存被回收了。
#### 6.3监听器和回调
内存泄漏第三个常见来源是监听器和其他回调，如果客户端在你实现的API中注册回调，却没有显示的取消，那么就会积聚。需要确保回调立即被当作垃圾回收的最佳方法是只保存他的若引用，例如将他们保存成为WeakHashMap中的键。

### 7、避免使用终结方法（finalizer）
- 终结方法不能保证被及时的执行或者根本不能被执行
- 终结方法带来严重的性能损失
- 尽量提供显示的终止方法，例如try{} finally{（connect.close（）；）}
- 除非是作为安全为，或者是为了终止非关键的本地资源，否则不要使用终结方法

---
## 第3章-对所有对象都通用的方法
### 8、覆盖equals时请遵守通用约定
不需要覆盖equals的情况
- 类的每个实例本质上都是唯一的
- 不关心类是否提供了“逻辑相等（logical equality）”的测试功能
- 超类已经覆盖了equals，从超类继承过来的行为对于子类也是合适的
- 类是私有的或是包级私有的，可以确定他的equlas方法永远不会被调用
- 值类大部分需要重写（枚举除外,枚举每个值之多只有一个对象）

覆盖equals的需要遵守的约定
- 自反性 x.equal(x) == true
- 对称性 y.equal(x)==true,则x.equal(y) == true
- 传递性 x=y,y=z,则x=z
- 一致性 x=y 两者没有修改，则x永远=y
- x.equals(null)永远=false


实现高质量equals方法的诀窍

- 使用==操作符检查“参数是否为这个对象的引用”；
- 使用instanceof操作符检查“参数是否为正确的类型”；
- 把参数转换成正确的类型；
- 对于该类中的每个“关键”域，检查参数中的域是否与该对象中对应的域相匹配
- 当编写完equals方法后，应该问自己三个问题：它是否是对称的、传递的和一致的？

最后的告诫

- 覆盖equals时总要覆盖hashCode。
- 不要企图让equals方法过于智能。
- 不要将equals声明中的Object对象替换为其他类型。

### 9、覆盖equlas时总要覆盖hashCode
忘记覆盖hashCode会违反Object.equlas的通用约定，导致一些基于散列的结合不能正常工作（HashMap,HashTable,HashSet）  
Obejct 规范
- 只要equals所用到的信息没修改，那么多次调用hashCode 返回的值必须是相同的整数结果
- 两个对象equlas相等，那么他们的hashCode产生的必须是相同的证书结果
- 如果两个对象equlas不相等，那么调用他们的hashCode，不一定要产生不同的整数结果，但是产生不一样的整数结果，有利于提高散列表的性能
