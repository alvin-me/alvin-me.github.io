---
layout: post
title: "文件监测系统"
tagline: "FileWatcher"
category : tech
tags : ["FileSystemWatcher"]
---

总结下前段时间使用 WPF 实现的一个简单的文件监测系统，主要功能是监测某个指定文件夹下指定类型的文件的添加、修改、删除、重命名事件。


## 一、UI

先来看下整个 UI 呈现的内容

![UI截图](https://cloud.githubusercontent.com/assets/14179733/10129144/882b2d76-65ed-11e5-8411-844bff8d0d7f.png)

**工具栏**包括

* "开始/停止"，开始或停止监听，如果监听文件夹不存在或未设置，则在状态栏会有提示。
* "清空"，清空显示列表。
* "保存"，保存显示列表内容为html格式文件。
* "预览"，关闭/打开右侧图像预览框。
* "设置"，弹出二级窗口，设置监听路径，日志保存路径，监听文件类型，以及是否监听子级目录。

**主体内容**左侧是一个列表用于显示监测信息，右侧是图像预览框，可以通过工具栏中的"预览"按键进行隐藏与显示。

**状态栏**主要用来显示当前监测状态，以及监听目录。

## 二、实现文件监听

#### 1、FileSystemWatcher

文件监听主要是通过 System.IO.FileSystemWatcher 接口实现的，该接口的使用比较简单，

``` csharp
var watcher = new FileSystemWatcher();
watcher.NotifyFilter = NotifyFilters.FileName | NotifyFilters.DirectoryName | NotifyFilters.Size; 	// 设置文件的哪些属性的变动会触发Changed事件，同时监控多个属性变动可以按“或”组合
watcher.Path = "D:/tmp";	// 设置监听目录路径
watcher.IncludeSubdirectories = true;	// 设置是否监听子级目录
watcher.Created += new FileSystemEventHandler(this.fileSystemWatching);	// 当创建文件和目录时发生并调用回调函数
watcher.Deleted += new FileSystemEventHandler(this.fileSystemWatching);	// 删除文件或目录时发生并调用回调函数
watcher.Changed += new FileSystemEventHandler(this.fileSystemWatching); // 当更改文件和目录时发生并调用回调函数，可以通过NotifyFilter属性设置触发该事件的需要文件更改的属性
watcher.Renamed += new RenamedEventHandler(this.fileSystemWatching);	// 重命名文件或目录时发生并调用回调函数
watcher.EnableRaisingEvents = true;	// 设置是否开始监控，默认为false
```

#### 2、注意事项

##### 1) 线程问题

因为 FileSystemWatcher 类本身就是多线程的控件，在回调函数中处理监听事件，向显示列表中添加一条监听信息。该线程需要更新界面上的元素，就会有一个线程安全性问题，需要按照下面的方式更新窗体元素。

``` csharp
Application.Current.Dispatcher.Invoke((Action)(() =>
{
	// update UI
}));
```

##### 2) 监听指定文件类型

FileSystemWatcher.Filter 设置筛选字符串，用于确定在目录中监视哪些类型的文件，如`watcher.Filter="*.txt"`将会只监听文本文件的状态，其他文件类型不会监听，默认值是`"*.*"`表示监听全部文件。<br/>
因为 FileSystemWatcher.Filter 不支持"或"组合，当我们需要监听两种以上指定文件类型时，一种方式是通过创建多个 FileSystemWatcher，并为每个 FileSystemWatcher 指定监听其中一种文件类型。但这种方式会创建多个线程，同时会让代码看起来比较冗余。<br/>
在这里我们可以使用一个小技巧：首先将监听文件类型设置为默认值，即监听全部文件，然后在监听事件回调函数中，通过代码判断所监听到的文件是否为需要指定文件类型。

``` csharp
//Filter Watching Files.
string[] watchingFilter = { ".tif", ".tiff" };
var ext = (Path.GetExtension(e.FullPath) ?? string.Empty).ToLower();
var all = ".*";
if (watchingFilter.Any(ext.Equals) || watchingFilter.Any(all.Equals) || watchingFilter.Length == 0)
{
	// do something
}
```

##### 3) 多次事件触发

还有一个问题是 FileSystemWatcher 类本身的问题，在创建或复制新文件到监听目录时，往往会连续触发 Created 和 Changed 事件，而我们需要只触发一次 Created 事件。Vipul Prashar 实现了一个增强的 FileSystemWatcher， 通过简单的比较当前事件与上一次事件发生的时间间隔，在时间间隔较小的情况时，合并两个事件，具体可以参考[这里](http://www.codeproject.com/Articles/102493/Enhanced-FileSystemWatcher)。

## 三、其它功能

* 缩略图实现，[Disposing a WPF Image or BitmapImage so the Source picture file can be modified](http://stackoverflow.com/questions/690150/delete-an-image-bound-to-a-control)；
* 添加将显示列表保存为html格式文件；
* 添加列表项右键菜单功能，可以快速对选中监听项指定文件进行操作，如打开文件、打开文件位置，以及删除文件。
![menu](https://cloud.githubusercontent.com/assets/14179733/10384924/08712622-6e75-11e5-8b29-0c5d432d7340.png)

具体实现请参考[我的代码仓库](https://github.com/alvin-me/FileWatcher)。

