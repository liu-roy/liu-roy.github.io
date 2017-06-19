---
title: Sqlite3常用的插入方法及性能测试
date: 2015-10-20 22:34:45
categories: sqlite3
tags: [sqlite3,c++,性能测试]
---

# Sqlite3常用的插入方法及性能测试  
最近在做一个大数据量缓存重传系统的优化，其中用到的sqlite技术，把自己的学习心得整理了一下。  

SQLite，是一款轻型的数据库，是遵守ACID的关系型数据库管理系统，它包含在一个相对小的C库中。同时能够跟很多程序语言相结合，比如 Tcl、C#、PHP、Java等，还有ODBC接口，同样比起Mysql、PostgreSQL这两款开源的世界著名数据库管理系统来讲，它的处理速度比他们都快。SQLite数据库由于其简单、灵活、轻量、开源，已经被越来越多的被应用到中小型应用中。因此在许多软件中例如(QQ，微信)等许多软件中都有广泛应用。
![这里写图片描述](http://img.blog.csdn.net/20170618222259467)
![这里写图片描述](http://img.blog.csdn.net/20170618222349843)

## 慢插入-暴力插入
![这里写图片描述](http://img.blog.csdn.net/20170618222506079)
调用sqlite3_exec()函数，会隐式地开启了一个事务，其次，sqlite3_exec() 是sqlite3_perpare()，sqlite3_step()，  sqlite3_finalize()的一个结合，每调用一次这个函数，就会重复的执行这三条语句，对于相同的语句，其中sqlite3_perpare相当于编译sql语句，如果语句相同且重复操作，就会增加很多重复操作。如果插入一条数据，就调该函数一次，事务就会被反复地开启、关闭，会增大IO量。所以当大批量数据插入时，此方法简直无法忍受。
<!--more-->
## 事务插入-显示的开启事务
![这里写图片描述](http://img.blog.csdn.net/20170618222544016)
所谓”事务“就是指一组SQL命令，这些命令要么一起执行，要么都不被执行。如果在插入数据前显式开启事务，插入后再一起提交，则会大大提高IO效率，进而加数据快插入速度。

## 同步关闭模式-synchronous = OFF
![这里写图片描述](http://img.blog.csdn.net/20170618222701657)
当synchronous设置为FULL, SQLite数据库引擎在紧急时刻会暂停以确定数据已经写入磁盘。这使系统崩溃或电源出问题时能确保数据库在重起后不会损坏。FULL synchronous很安全但很慢。
当synchronous设置为NORMAL, SQLite数据库引擎在大部分紧急时刻会暂停，但不像FULL模式下那么频繁。 NORMAL模式下有很小的几率(但不是不存在)发生电源故障导致数据库损坏的情况。但实际上，在这种情况 下很可能你的硬盘已经不能使用，或者发生了其他的不可恢复的硬件错误。
当设置为synchronous OFF时，SQLite在传递数据给系统以后直接继续而不暂停。若运行SQLite的应用程序崩溃， 数据不会损伤，但在系统崩溃或写入数据时意外断电的情况下数据库可能会损坏。另一方面，在synchronous OFF时 一些操作可能会快50倍甚至更多。在SQLite 2中，缺省值为NORMAL.而在3中修改为FULL。

## 预处理（执行前准备）-sqlite3_prepare_v2
![这里写图片描述](http://img.blog.csdn.net/20170618222945117)
此方法就是“执行准备”（类似于存储过程）操作，即先将SQL语句编译好，然后再一步一步（或一行一行）地执行。如果采用前者的话，就算开起了事务，SQLite仍然要对循环中每一句SQL语句进行“词法分析”和“语法分析”，这对于同时插入大量数据的操作来说，简直就是浪费时间。因此，要进一步提高插入效率的话，就应该使用此方法
## 测试结果展示
![这里写图片描述](http://img.blog.csdn.net/20170618223154426)
## 测试源代码
```c++
extern "C"
{
	#include "sqlite3.h"
};

#include<sstream>
#include <string>
#include <iostream>
#include <stdlib.h>
#include <ctime>
#include<windows.h>


#define MAX_TEST_COUNT 200

using namespace std;


int main()
{
	char cmdCreatTable[256] = "create table SqliteTest (id integer , x integer , y integer, weight real)" ;
	sqlite3* db = NULL;
	char * errorMessage = NULL;
	int iResult = sqlite3_open("SqliteTest.db", &db);
	do
	{
		if (SQLITE_OK != iResult)
		{
			cout<<"创建InsertTest.db文件失败"<<endl;
			break;
		}

		sqlite3_exec(db,"drop table if exists SqliteTest",0,0,0);  

		iResult = sqlite3_exec(db, cmdCreatTable, NULL, NULL, &errorMessage);
		if (SQLITE_OK != iResult)
		{
			cout<<"创建表SqliteTest失败"<<endl;
			break;
		}
		DWORD timeStart;
		DWORD timeStop;
		timeStart = GetTickCount();
		for (int i = 0; i< MAX_TEST_COUNT; ++i)
		{
			stringstream ssm;  
			ssm<<"insert into SqliteTest values("<<i<<","<<i*2<<","<<i/2<<","<<i*i<<")"; 
		    iResult = sqlite3_exec(db,ssm.str().c_str(),0,0,0); 
		}
		timeStop = GetTickCount();
		cout<< "直接Insert"<<MAX_TEST_COUNT<<"条数据操作执行时间" << timeStart<<"结束时间:"<<timeStop<<"共耗时:"<<timeStop-timeStart<<"ms"<<endl;

		timeStart = GetTickCount();
		sqlite3_exec(db,"PRAGMA synchronous = OFF; ",0,0,0);   
		for(int i = MAX_TEST_COUNT; i < MAX_TEST_COUNT*2; ++i)  
		{  
			stringstream ssm;  
			ssm<<"insert into SqliteTest values("<<i<<","<<i*2<<","<<i/2<<","<<i*i<<")";  
			sqlite3_exec(db,ssm.str().c_str(),0,0,0);  
		} 
		timeStop = GetTickCount();

		cout<< "同步写关闭+直接Insert"<<MAX_TEST_COUNT<<"条数据操作执行时间" << timeStart<<"结束时间:"<<timeStop<<"共耗时:"<<timeStop-timeStart<<"ms"<<endl;


		timeStart = GetTickCount();
		sqlite3_exec(db,"PRAGMA synchronous = FULL; ",0,0,0); 
		sqlite3_exec(db,"begin;",0,0,0);  
		for(int i= MAX_TEST_COUNT*2; i< MAX_TEST_COUNT*3; ++i)  
		{  
			stringstream ssm;  
			ssm<<"insert into SqliteTest values("<<i<<","<<i*2<<","<<i/2<<","<<i*i<<")";  
			sqlite3_exec(db,ssm.str().c_str(),0,0,0);  
		}  
		sqlite3_exec(db,"commit;",0,0,0); 
		timeStop = GetTickCount();
		cout<< "事务Insert"<<MAX_TEST_COUNT<<"条数据操作执行时间"<< timeStart<<"结束时间:"<<timeStop<<"共耗时:"<<timeStop-timeStart<<"ms"<<endl;


        timeStart = GetTickCount();
		sqlite3_exec(db,"PRAGMA synchronous = OFF; ",0,0,0);  
		sqlite3_exec(db,"begin;",0,0,0);  
		for(int i = MAX_TEST_COUNT*3; i < MAX_TEST_COUNT*4; ++i)  
		{  
			stringstream ssm;  
			ssm<<"insert into SqliteTest values("<<i<<","<<i*2<<","<<i/2<<","<<i*i<<")";  
			sqlite3_exec(db,ssm.str().c_str(),0,0,0);  
		}  
		sqlite3_exec(db,"commit;",0,0,0); 
		timeStop = GetTickCount();

		cout<< "事务+同步写关闭Insert"<<MAX_TEST_COUNT<<"条数据操作执行时间" << timeStart<<"结束时间:"<<timeStop<<"共耗时:"<<timeStop-timeStart<<"ms"<<endl;

		timeStart = GetTickCount();
		//sqlite3_exec(db,"PRAGMA synchronous = FULL; ",0,0,0); 
		sqlite3_exec(db,"begin;",0,0,0);  
		sqlite3_stmt *stmt;  
		const char* sql = "insert into SqliteTest values(?,?,?,?)";  
	    sqlite3_prepare(db,sql,strlen(sql),&stmt,0);  
		for(int i = MAX_TEST_COUNT*4; i<MAX_TEST_COUNT*5; ++i)  
		{         
			sqlite3_reset(stmt);  
		    sqlite3_bind_int(stmt,1,i);  
		    sqlite3_bind_int(stmt,2,i*2);  
		    sqlite3_bind_int(stmt,3,i/2);  
			sqlite3_bind_double(stmt,4,i*i);  
			sqlite3_step(stmt); 
		 }  
		 sqlite3_finalize(stmt);  
		 sqlite3_exec(db,"commit;",0,0,0);  

		 timeStop = GetTickCount();
		 cout<< "事务+执行准备+同步写关闭Insert"<<MAX_TEST_COUNT<<"条数据操作执行时间:"<< timeStart<<"结束时间:"<<timeStop<<"共耗时:"<<timeStop-timeStart<<"ms"<<endl;


	}while(0);

	cout<<"插入测试结束"<<endl;
	Sleep(2000);
	sqlite3_close(db);
	system("pause");
	
}
```