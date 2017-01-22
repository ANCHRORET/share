refer：
- [https://developer.android.google.cn/training/system-ui/immersive.html](https://developer.android.google.cn/training/system-ui/immersive.html)  
- [https://developer.android.google.cn/reference/android/view/View.html#setSystemUiVisibility(int)](https://developer.android.google.cn/reference/android/view/View.html#setSystemUiVisibility(int))
- [Android Activity 全屏方法总结](http://talentprince.github.io/blog/2015/01/07/android-activity-quan-ping-fang-fa-zong-jie/)  
- [关于状态栏StatusBar（System UI）的各种操作...](http://www.jianshu.com/p/8b3ec46dac39)  


# android 如何实现改变状态栏statusbar的颜色 #
本文主要想了解随着版本的变化，如何实现改变状态栏的颜色

## API 19 ##
从 API 19 开始，statusbar 开始可以设为透明状态。  
主要涉及到以下API：  
void setSystemUiVisibility (int visibility)  **level 11** 改变设置的visibility的值，来实现状态栏的变化
- View.SYSTEM_UI_FLAG_VISIBLE **level 14** 默认值  
- View.SYSTEM_UI_FLAG_LAYOUT_STABLE **Level 16** 