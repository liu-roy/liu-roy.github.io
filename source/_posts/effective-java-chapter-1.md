---
title: 第2章-创建和销毁对象
date: 2017-07-24 15:53:24
categories: effective java
tags: 读书笔记
---

## 1、考虑用静态工厂方法替代构造器
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
 	

## 2、遇到多个构造器参数时要考虑用构建器
静态工厂和构造器共同的局限，他们都不能很好的扩展到大量的可选参数

### 可选参数扩展方法
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