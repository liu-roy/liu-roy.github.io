---
title: 第3章-创建和销毁对象
date: 2017-07-26 11:06:00
categories: effective java
tags: 读书笔记
---
## 8、覆盖equals时请遵守通用约定
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

## 9、覆盖equlas时总要覆盖hashCode
忘记覆盖hashCode会违反Object.equlas的通用约定，导致一些基于散列的结合不能正常工作（HashMap,HashTable,HashSet）  
Obejct 规范
- 只要equals所用到的信息没修改，那么多次调用hashCode 返回的值必须是相同的整数结果
- 两个对象equlas相等，那么他们的hashCode产生的必须是相同的证书结果
- 如果两个对象equlas不相等，那么调用他们的hashCode，不一定要产生不同的整数结果，但是产生不一样的整数结果，有利于提高散列表的性能