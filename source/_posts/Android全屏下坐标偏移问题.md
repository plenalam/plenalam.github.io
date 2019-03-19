---
title: Android全屏下坐标偏移问题
date: 2018-10-04 11:07:26
tags: 
    - Android
    - BUG
---
一个全屏(设置了SYSTEM_UI_FLAG_FULLSCREEN)的Activity需要根据动态定位view的位置,于是用传统的方式捕获屏幕宽高.

```
WindowManager windowManager = getWindowManager(); 
Display display = windowManager.getDefaultDisplay();  
screenWidth = display.getWidth();  
screenHeight = display.getHeight(); 
```
横屏情况下,在三星S9上所有view都向左偏移.定位到四个极限点,发现左上和左下没有问,但是右边两个点向左偏移,目测是一个标题的高度.  
打印log果然发现screenWidth少了标题栏高度  

试了试其他几个常用的获取屏幕size的Api,结果都一样.于是转换思路,在界面显示后调用根布局的getMeasuredHeight()和getMeasuredWidth(),返回了正确结果.

后来发现Android在SDK17后多了个新方法getRealMetrics
则
```
DisplayMetrics metrics = new DisplayMetrics();
Display display = getWindowManager().getDefaultDisplay();

 // For JellyBean 4.2 (API 17) and onward
if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.JELLY_BEAN_MR1) {
    display.getRealMetrics(metrics);
    screenWidth = metrics.widthPixels;
    screenHeight = metrics.heightPixels;
    } 
```
这种方法也能成功获取屏幕大小

## 参考
https://stackoverflow.com/questions/10991194/android-displaymetrics-returns-incorrect-screen-size-in-pixels-on-ics