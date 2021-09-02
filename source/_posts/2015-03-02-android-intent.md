---
layout: post
title: "Android 之 Intent 学习"
date: 2015-03-02
tags: Intent
categories: Android
---

## Activity切换

>两种方式

### 直接跳转
```java
Intent intent = new Intent(MainActivity.this,SecondActivity.class);
startActivity(intent);
```
需要携带参数则需使用
```
intent.putExtra("key", "value");
```
目标Activity取参数使用
```
getIntent().getStringExtra("key")
 ```

### 带返回值跳转
```java
Intent intent = new Intent(MainActivity.this,SecondActivity.class);
intent.putExtra("key", "value");
startActivityForResult(intent, 1);
```
目标activity返回
```java
Intent intent = new Intent();
intent.putExtra("code", "OK");
setResult((int)requestCode,intent);                
finish();
```
**finish()**函数 : 执行结束销毁此activity，这个ActivityResult返回回到调用者那里并调用onActivityResult()函数.

原activity接受返回值的操作
```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    //判断操作
    if (requestCode==1 && resultCode==2) {
        mEditText.setText(data.getStringExtra("code"));
        //Toast.makeText(MainActivity.this, data.getStringExtra("code"), Toast.LENGTH_LONG).show();
    }
}
```