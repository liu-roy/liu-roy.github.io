---
title: sqlite3的图片的（二进制数据）存取操作
date: 2015-08-18 21:54:02
categories: sqlite3
tags: [sqlite3, c++]
commants: false
---
# sqlite3的图片的（二进制数据）存取操作
&emsp;
## 前言
上篇介绍了**sqlite3**的一些常用插入操作方法和注意事项，在实际项目中遇到了图片缓存的问题，由于服务器不是很稳定，且受到外界环境的干扰（例如断电，图片存储挂掉，图片存储速度过慢，造成的接口调用失败等等），一个数据结构中除了普通字段（int string），还包括图片数据，所以还需要将图片数据进行缓存，图片缓存与普通的数据库字段值缓存有所不同，下面介绍一下简单方法。

## 开发示例
此demo仅供学习使用。

sqlite3支持对二进制数据的缓存，在实际的编程开发当中我们经常要处理一些大容量二进制数据的存储，如图片、音乐、视频等等。对于这些二进制数据，我们不能像处理普通的文本那样，但是我们可以用**blob**来存储。sqlite官方文档<https://www.sqlite.org/datatype3.html#section_1>对**blob** 字段的解释是
>  BLOB. The value is a blob of data, stored exactly as it was input。 

即数据不做任何转换，以输入形式存储。因此 BOLB通常用来存储二进制大对象。

## sqlite3\_bind\_blob示例代码 

```c++
char* cmdCreatBlobTable = "create table SqliteBlobTest (id integer , pic blob);  //首先创建一个可插入blob类型的表 。
sqlite3* db = NULL;
char * errorMessage = NULL;
int iResult = sqlite3_open("SqliteTest.db", &db);
sqlite3_exec(db,"drop table if exists SqliteBlobTest",0,0,0);  

iResult = sqlite3_exec(db, cmdCreatBlobTable, NULL, NULL, &errorMessage);
if (SQLITE_OK != iResult)
{
    cout<<"创建表SqliteBlobTest失败"<<endl;
    break;
}

sqlite3_stmt *stmt;                                            //声明
const char* sql = "insert into SqliteBlobTest values(1,?)";  
char* pPicData = "this is a pic data" ;
sqlite3_prepare(db,sql,strlen(sql),&stmt,0);                   //完成对sql语句的解析
{  
    sqlite3_bind_blob(stmt,1,pPicData, strlen(pPicData), NULL);//1代表第一个？
    sqlite3_step(stmt);                                        //将数据写入数据库中
} 
<!--more-->
sqlite3_prepare(db, "select * from SqliteBlobTest", -1, &stmt, 0);
int result = sqlite3_step(stmt);
int id = 0,len = 0; 
char picData[128] = {0}; 
if (result == SQLITE_ROW)                                     //查询成功返回的是SQLITE_ROW
{
    cout<<"read success from sqlite"<<endl;
    id = sqlite3_column_int(stmt, 0);                         //从0开始计算，id为0，picdata 为1；
    const void * pReadPicData = sqlite3_column_blob(stmt, 1); //读取数据，返回一个指针
    len = sqlite3_column_bytes(stmt, 1);                      //返回数据大小
    memcpy(picData, pReadPicData, len);                       //把数据拷贝出来
}
else
{
    cout<<"read fail from sqlite"<<endl;
}
sqlite3_finalize(stmt);                                      //把刚才分配的内容析构掉
cout<<id<<" "<<picData<<endl;
```

### 测试结果
![这里写图片描述](http://img.blog.csdn.net/20160621162555449)
![这里写图片描述](http://img.blog.csdn.net/20160621162509842)

## 总结

经过这一个月工作之余的优化，终于把项目的缓存给做好了，其中也遇到了很多问题，例如sqlite的编码转换，图片缓存速度慢，还有db-journal文件操作慢，以及如何直观的让sqliteDb大小自动展现，自己也是查了官方英文文档才一步步解决各种坑。总结的好处就在于能够温故知新。