# Android里面为什么要设计出Bundle而不是直接用Map结构

来源:[https://github.com/android-cn/android-discuss/issues/142](https://github.com/android-cn/android-discuss/issues/142)

* 一楼

[Bundle](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.0.2_r1/android/os/Bundle.java) 父类 [BaseBundle](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.0.2_r1/android/os/BaseBundle.java) 内部确实有个 ArrayMap<String, Object> 类型的 mMap 成员。

我觉得之所以封装为 Bundle 应该和 Android 特有的 Parcelable 序列化方式有关（比 JDK 自带的 Serializable 效率高）。通过源码可以看到内部各种 put 和 get 方法都调用了这个 [unparcel](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.0.2_r1/android/os/BaseBundle.java#BaseBundle.unparcel%28%29) 方法。

* 二楼

补充下上面的回答：

Map里实现了Serializable接口，而在Bundle实现了Parcelable的接口

* 三楼

[http://stackoverflow.com/questions/6674771/hashmap-vs-bundle-in-android-efficiency-and-performance-comparison](http://stackoverflow.com/questions/6674771/hashmap-vs-bundle-in-android-efficiency-and-performance-comparison) 可以看看这个

* 四楼

* Bundle内部是由ArrayMap实现的，ArrayMap的内部实现是两个数组，一个int数组是存储对象数据对应下标，一个对象数组保存key和value，内部使用二分法对key进行排序，所以在添加、删除、查找数据的时候，都会使用二分法查找，只适合于小数据量操作，如果在数据量比较大的情况下，那么它的性能将退化。而HashMap内部则是数组+链表结构，所以在数据量较少的时候，HashMap的Entry Array比ArrayMap占用更多的内存。因为使用Bundle的场景大多数为小数据量，我没见过在两个Activity之间传递10个以上数据的场景，所以相比之下，在这种情况下使用ArrayMap保存数据，在操作速度和内存占用上都具有优势，因此使用Bundle来传递数据，可以保证更快的速度和更少的内存占用。

* 另外一个原因，则是在Android中如果使用Intent来携带数据的话，需要数据是基本类型或者是可序列化类型，HashMap使用Serializable进行序列化，而Bundle则是使用Parcelable进行序列化。而在Android平台中，更推荐使用Parcelable实现序列化，虽然写法复杂，但是开销更小，所以为了更加快速的进行数据的序列化和反序列化，系统封装了Bundle类，方便我们进行数据的传输。

* 参考资料

[Android内存优化（使用SparseArray和ArrayMap代替HashMap）](http://www.codes51.com/article/detail_163576.html)<br/>
Android开发艺术探索，第47页

更多可关注：[https://github.com/ZhaoKaiQiang/AndroidDifficultAnalysis](https://github.com/ZhaoKaiQiang/AndroidDifficultAnalysis)