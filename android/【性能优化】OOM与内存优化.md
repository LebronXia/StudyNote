bitmap:

[Android ImageView 使用和回收的方式](https://blog.csdn.net/weixin_35691921/article/details/108581174)

加载的时候，无论是否是本地图片，最好使用第三方库加载，节约内存
Glide.with(this).load(R.mipmap.vehicle_1).into(imageView);
1
回收的时候，最好将ImageView置空，因为无引用，GC更容易回收

              ``` java
              imageView.setImageDrawable(null);
                              imageView = null;
              ```





[Memory Analyzer基本使用](https://blog.csdn.net/OldApple_MrZ/article/details/102647599)

[图片系列（6）不同版本上 Bitmap 内存分配与回收原理对比](https://blog.csdn.net/pengxurui/article/details/126326790)

[云音乐 Android 内存监控探索篇](https://juejin.cn/post/7197032258888646693#heading-2)

[抖音 Android 性能优化系列: Java 内存优化篇](https://juejin.cn/post/6908517174667804680)

