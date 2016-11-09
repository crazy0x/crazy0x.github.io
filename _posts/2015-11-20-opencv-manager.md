---
layout: post
title: Android工程去除OpenCV依赖
categories: [openCV]
tags: [Android, openCV]
description: Android Studio使用OpenCV后，使APP不安装OpenCV Manager即可运行.
---

采用静态初始化的方法，可以戳下边的链接查看官方的文档介绍
http://docs.opencv.org/doc/tutorials/introduction/android_binary_package/dev_with_OCV_on_Android.html#application-development-with-static-initialization

如果项目不包含JNI部分，比较简单的办法就是：

1）注销掉OpenCVLoader.initAsync(OpenCVLoader.OPENCV_VERSION_2_4_3, this, mLoaderCallback); 在语句上边直接设为SUCCESS。

{% highlight yaml %}
public void onResume()
    {
        super.onResume();
        mLoaderCallback.onManagerConnected(LoaderCallbackInterface.SUCCESS);
        //OpenCVLoader.initAsync(OpenCVLoader.OPENCV_VERSION_2_4_3, this, mLoaderCallback);
    }
{% endhighlight %}

2）在Activity类中添加静态的方法

{% highlight yaml %}
static{
        if(!OpenCVLoader.initDebug()){
            //handle initialization error
        }
    }
{% endhighlight %}

如果有其他的自定义原生库需要加载，可以在这里添加else语句：

{% highlight yaml %}
static{
        if(!OpenCVLoader.initDebug()){
            //handle initialization error
        }else{
        　　System.loadLibrary("my_jni_lib1");
        　　System.loadLibrary("my_jni_lib2");
        }
}
{% endhighlight %}

OpenCV for Android 3.0版本里，示例程序直接就可免OpenCV Manager的安装，它的初始代码onResume函数中是这样写的：

{% highlight yaml %}
public void onResume()
    {
        super.onResume();
        if (!OpenCVLoader.initDebug()) {
            Log.d(TAG, "Internal OpenCV library not found. Using OpenCV Manager for initialization");
            OpenCVLoader.initAsync(OpenCVLoader.OPENCV_VERSION_3_0_0, this, mLoaderCallback);
        } else {
            Log.d(TAG, "OpenCV library found inside package. Using it!");
            mLoaderCallback.onManagerConnected(LoaderCallbackInterface.SUCCESS);
        }
    }
{% endhighlight %}

这样的写法在OpenCV 4.xx中同样适用，所以推荐下边的这种方法。
