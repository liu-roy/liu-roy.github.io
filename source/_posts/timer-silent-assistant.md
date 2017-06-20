---
title: 定时静音助手
date: 2014-10-18 16:54:30
categories: java
tags: [android]
---
# 定时静音助手 	

## 背景
突发奇想，刚好这学期刚上安卓课程，想设计一个时间助手。工作、学习中经常会被突如其来的电话所打扰，在上班，上课时这突如其来的铃声会惹来别人的反感，而只靠人们的记性是很难在准确的时间记得静音。如果一直静音，那么在休息时间又有可能漏接重要的电话。基于这种考虑，设计了这样一自动静音小助手，来帮助人们在忙碌的生活中定时静音，定时开启正常模式，简单方便。
<!--more-->
## 界面设计
```javascript 
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    android:orientation="vertical" >

    <Button
        android:id="@+id/btnAddAlarm1"
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:text="添加静音开始时间" />
    <TextView
        android:id="@+id/tvAlarmRecord1"
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:textSize="16dp" />
   
     <Button
         android:id="@+id/btnAddAlarm2"
         android:layout_width="match_parent"
         android:layout_height="wrap_content"
         android:text="添加静音停止时间" />
       <TextView
        android:id="@+id/tvAlarmRecord2"
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:textSize="16dp" /
</LinearLayout>
```
点击完按钮的会出现一个时间点设置的对话框 代码如下
```javascript
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
	android:orientation="vertical" android:layout_width="fill_parent"
	android:layout_height="fill_parent">
	<TimePicker android:id="@+id/timepicker1"
		android:layout_width="fill_parent" android:layout_height="wrap_content" />
	
</LinearLayout>
```
## 效果图
![这里写图片描述](http://img.blog.csdn.net/20170620231246393)

## 功能设计
### 原理介绍
先简单介绍一下工作原理。在添加时间点之后，需要将所添加的时间点保存在文件或者数据库中，我使用了SharedPrefences来保存时间点，key和value都是时间点，然后用到AlarmManager每隔一分钟扫描一次，在扫描过程中从文件获取当前时间（时：分）的value，如果成功获得value就说明当前时间为时间点，此时调用audioManager ，当扫描掉button1设置的文件信息，就调用AudioManager.RINGER_MODE_SILENT，如果扫描到button2设置的文件信息，就调用AudioManager.RINGER_MODE_NORMAL，时期出去正常模式。
此程序包含两个java文件，分别是MainActivity.java和TimeReceiver.java，TimeReceiver主要是判断是否到达时间点，MainActivity 主要是整体的框架和逻辑。

### MainActivity代码如下：
```java
package com.example.timesilent;
import android.os.Bundle;
import android.app.Activity;
import android.app.AlarmManager;
import android.app.PendingIntent;
import android.view.View;
import android.view.View.OnClickListener;
import android.widget.Button;
import android.widget.TextView;
import android.widget.TimePicker;
import android.app.AlertDialog;
import android.content.Context;
import android.content.DialogInterface;
import android.content.Intent;
import android.content.SharedPreferences;

public class MainActivity extends Activity implements OnClickListener 
{

	
	private SharedPreferences sharedPreferences1;
	private SharedPreferences sharedPreferences2;
	private TextView tvAlarmRecord1;
	private TextView tvAlarmRecord2;
	@Override
	public void onClick(View v) {
		View view =getLayoutInflater().inflate(R.layout.time,null);
		final TimePicker timePicker1=(TimePicker)view.findViewById(R.id.timepicker1);
		
		timePicker1.setIs24HourView(true);
		
		switch(v.getId())
		{
			 case R.id.btnAddAlarm1:
			 {
				 new AlertDialog.Builder(this).setTitle("设置静音开始时间").setView(view).setPositiveButton("确定",new DialogInterface.OnClickListener(){
					

					@Override
				public void onClick(DialogInterface dialog,int which)
					 {
						 String timeStr=String.valueOf(timePicker1.getCurrentHour())+":"+String.valueOf(timePicker1.getCurrentMinute());
						
						tvAlarmRecord1.setText(tvAlarmRecord1.getText().toString()+"\n"+timeStr);
						 sharedPreferences1.edit().putString(timeStr,timeStr).commit();
					 }
				 }).setNegativeButton("取消",null).show();
				 break;
			 }
			 case R.id.btnAddAlarm2:
			 {
				 new AlertDialog.Builder(this).setTitle("设置静音结束时间").setView(view).setPositiveButton("确定",new DialogInterface.OnClickListener(){
					 @Override
					 public void onClick(DialogInterface dialog,int which)
					 {
						 String timeStr=String.valueOf(timePicker1.getCurrentHour())+":"+String.valueOf(timePicker1.getCurrentMinute());
						
						tvAlarmRecord2.setText(tvAlarmRecord2.getText().toString()+"\n"+timeStr);
						 sharedPreferences2.edit().putString(timeStr,timeStr).commit();
					 }
				 }).setNegativeButton("取消",null).show();
				 break;
			 }
		}
	
		
	}
	@Override
	public void onCreate(Bundle savedInstanceState)
		{
			super.onCreate(savedInstanceState);
			setContentView(R.layout.activity_main);
			Button btnAddAlarm1 = (Button) findViewById(R.id.btnAddAlarm1);
			Button btnAddAlarm2 = (Button) findViewById(R.id.btnAddAlarm2);
			tvAlarmRecord1 = (TextView) findViewById(R.id.tvAlarmRecord1);
			tvAlarmRecord2 = (TextView) findViewById(R.id.tvAlarmRecord2);
			btnAddAlarm1.setOnClickListener(this);
			btnAddAlarm2.setOnClickListener(this);
			
			sharedPreferences1 = getSharedPreferences("alarm_record1",
					Activity.MODE_PRIVATE);
			sharedPreferences2 = getSharedPreferences("alarm_record2",
					Activity.MODE_PRIVATE);

			AlarmManager alarmManager = (AlarmManager) getSystemService(Context.ALARM_SERVICE);
			Intent intent = new Intent(this, TimeReceiver.class);
			PendingIntent pendingIntent = PendingIntent.getBroadcast(this, 0,
					intent, 0);
			alarmManager.setRepeating(AlarmManager.RTC, 0, 60 * 1000, pendingIntent);

		}
	
}
```
### TimeReceiver的代码如下：
```java
package com.example.timesilent;

import java.util.Calendar;
import android.app.Activity;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.media.AudioManager;
import android.content.SharedPreferences;



public class TimeReceiver extends BroadcastReceiver
{

	@Override
	public void onReceive(Context context, Intent intent)
	{
		
		SharedPreferences sharedPreferences1 = context.getSharedPreferences(
				"alarm_record1", Activity.MODE_PRIVATE);
		SharedPreferences sharedPreferences2 = context.getSharedPreferences(
				"alarm_record2", Activity.MODE_PRIVATE);
		String hour = String.valueOf(Calendar.getInstance().get(
				Calendar.HOUR_OF_DAY));
		String minute = String.valueOf(Calendar.getInstance().get(
				Calendar.MINUTE));
		String time1 = sharedPreferences1.getString(hour + ":" + minute, null);
		String time2 = sharedPreferences2.getString(hour + ":" + minute, null);
		AudioManager audioManager = (AudioManager)context.getSystemService(Context.AUDIO_SERVICE);
		if (time1!= null)
		{	
			audioManager.setRingerMode(AudioManager.RINGER_MODE_SILENT);
		}
		if (time2!= null)
		{	
			
			audioManager.setRingerMode(AudioManager.RINGER_MODE_NORMAL);
		}
		
	}

	
	
}
```
## 程序运行效果
### 初始状态
![这里写图片描述](http://img.blog.csdn.net/20170620231623547)
### 开始静音状态
![这里写图片描述](http://img.blog.csdn.net/20170620231704956)
### 恢复正常状态
![这里写图片描述](http://img.blog.csdn.net/20170620231732765)
## 源码地址
[github](https://github.com/liu-roy/timeSilent)