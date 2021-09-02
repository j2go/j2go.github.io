---
layout: post
title: "Android 之 AsyncTask 学习"
date: 2015-03-04
tags: AsyncTask
categories: Android
---

AsyncTask 就是一个封装过的后台任务类，顾名思义就是异步任务。

AsyncTask 直接继承于 Object 类，包位置为 android.os.AsyncTask. 要使用AsyncTask工作要提供三个泛型参数，并重载几个方法(至少重载一个)。


AsyncTask定义了三种泛型类型 Params，Progress和Result。

- Params:   启动任务执行的输入参数，比如HTTP请求的URL。
- Progress: 后台任务执行的百分比。
- Result:   后台执行任务最终返回的结果，比如String。

要使用 AsyncTask 的同学都需要知道一个异步加载数据最少要重写以下这两个方法：

- **doInBackground(Params…):** 后台执行，比较耗时的操作都可以放在这里。注意这里不能直接操作UI。此方法在后台线程执行，完成任务的主要工作，通常需要较长的时间。在执行过程中可以调用publicProgress(Progress…)来更新任务的进度。
- **onPostExecute(Result):**  相当于Handler 处理UI的方式，在这里面可以使用在doInBackground 得到的结果处理操作UI。 此方法在主线程执行，任务执行的结果作为此方法的参数返回


有必要的话还可重写以下这三个方法，但不是必须的：

- **onProgressUpdate(Progress…):**   可以使用进度条增加用户体验度。 此方法在主线程执行，用于显示任务执行的进度。
- **onPreExecute():** 这里是最终用户调用Excute时的接口，当任务执行之前开始调用此方法，可以在这里显示进度对话框。
- **onCancelled():** 用户调用取消时，要做的操作

## 使用AsyncTask类注意点

- Task的实例必须在UI thread中创建；
- execute方法必须在UI thread中调用；
- 不要手动的调用onPreExecute(), onPostExecute(Result)，doInBackground(Params...), onProgressUpdate(Progress...)这几个方法；
- 该task只能被执行一次，多次调用时将会出现异常.