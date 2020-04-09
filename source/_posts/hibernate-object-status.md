---
title: hibernate的对象状态分析
date: 2017-07-15 16:21:24
categories: hibernate
tags: [hiberate-objectState]
---
## 开发框架
- springMVC
- hibernate5.0.1
## hibernate三种状态
Hibernate定义并支持下列对象状态(state):

### 临时状态(Transient)
当new一个实体对象后, 这个对象处于临时状态, 即这个对象只是一个保存临时数据的内存区域, 如果没有变量引用这个对象, 则会被jre垃圾回收机制回收. 这个对象所保存的数据与数据库没有任何关系, 除非通过Session的save或者SaveOrUpdate把临时对象与数据库关联, 并把数据插入或者更新到数据库, 这个对象才转换为持久对象. 
<!--more-->
### 持久状态(Persistent)  
持久(Persistent)的实例在数据库中有对应的记录，并拥有一个持久化标识(identifier)。 持久(Persistent)的实例可能是刚被保存的，或刚被加载的，无论哪一种，按定义，它存在于相关联的Session作用范围内。 Hibernate会检测到处于持久(Persistent)状态的对象的任何改动，在当前操作单元执行完毕时将对象数据与数据库同步。 开发者不需要手动执行UPDATE。将对象从持久(Persistent)状态变成临时(Transient)状态同样也不需要手动执行DELETE语句。
### 游离状态(Detached)  
当Session进行了Close、Clear或者evict后, 持久化对象虽然拥有持久化标识符和与数据库对应记录一致的值, 但是因为会话已经消失, 对象不在持久化管理之内, 所以处于游离. 游离状态的对象与临时状态对象是十分相似的, 只是它还含有持久化标识.游离状态(Detached)对象如果重新关联到某个新的Session上， 会再次转变为持久(Persistent)的 。

具体可以看下图（图片来自网络） 

![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/hibernate-object-status/1.jpg)



## 代码演示
为了演示三种状态的区别，做几个小例子

### 持久态自动更新演示
如下代码，从数据库里取出catgegory对象，此时的对象处于持久态，通过set方法，更改category的名字，此时Category已经发生变化，为了演示自动更新，我们采用另一个对象，通过save另一个对象，来顺便更新category。注意，此时scriptService save的是一个script对象，会看到hibernate日志，同时也更新了category对象。该实验必须在一个session上下文中进行。

#### 演示代码和hibernate日志
``` java
 @PostMapping(value = "/groupEdit")
    public @ResponseBody RestResult groupEdit(@RequestParam Integer id, @RequestParam String name){
        ScriptCategory category = scriptCategoryService.findByName(name);
        if (category != null){
            return new RestResult(false, "已存在同名分组");
        }
        category = scriptCategoryService.findById(id);
        category.setName(name);
        //scriptCategoryService.save(category);
        Script script = scriptService.findById(1);
        script.setAuthor("llll");
        scriptService.save(script);
        OperationLogHolder.builder().record(OperationAction.EDIT_SCRIPT_GROUP, "编辑脚本分组: "+ name ,
                OperationResult.SUCCESS);
        return new RestResult(true, "编辑分组成功");
    }
```
![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/hibernate-object-status/2.jpg)

### 从持久态变为游离态演示
假如说你不想对category进行自动更新，则可以通过将持久态转化为游离态，也就是托管状态，通过session.evict 方法，删除session中的category,此时scriptService.save(script)只会保存自己的对象。 

在自己的service行加入以下方法，移除session上下文中的对象，就可以变为游离态。

#### 演示代码和hibernate日志
```java
@Service("scriptService")
@Transactional(readOnly = true)
public class ScriptServiceImpl implements ScriptService {

    @Autowired
    private EntityManager entityManager;

    @Autowired
    private ScriptRepo scriptRepo;

    //此处省略若干行  

    @Override
    public void sessionEvict(Object object){
        Session session = (Session)entityManager.getDelegate();
        session.evict(object);
    }
}
```
```java
    @PostMapping(value = "/groupEdit")
    public @ResponseBody RestResult groupEdit(@RequestParam Integer id, @RequestParam String name){
        ScriptCategory category = scriptCategoryService.findByName(name);
        if (category != null){
            return new RestResult(false, "已存在同名分组");
        }
        category = scriptCategoryService.findById(id);
        category.setName(name);
        //scriptCategoryService.save(category);
        scriptService.sessionEvict(category);
        Script script = scriptService.findById(1);
        script.setAuthor("lgggg");
        scriptService.save(script);
        OperationLogHolder.builder().record(OperationAction.EDIT_SCRIPT_GROUP, "编辑脚本分组: "+ name ,
                OperationResult.SUCCESS);
        return new RestResult(true, "编辑分组成功");
    }
```

![这里写图片描述](http://roy-markdown.oss-cn-qingdao.aliyuncs.com/hibernate-object-status/3.jpg)
