如何方式应用进程被终止

#参考链接：
* http://www.cnblogs.com/wlrhnh/p/3529926.html
* https://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&amp;mid=403254393&amp;amp;idx=1&amp;sn=8dc0e3a03031177777b5a5876cb210cc&amp;utm_source=tuicool&;utm_medium=referral

#一、示例代码
示例代码主要参考了微信的实现方式，但只保留实现了一个进程和一个后台运行的Service，同时也保留了AlarmManager的机制。请从附件，或从github中下载：https://github.com/ruibinliu/DeamonService

#二、进程被清理的原因
按照自动和手动清理进行分类，进程被清理的原因如下：
## 自动关闭：
1. 应用自己关闭自己；
2. 手机内存资源不足。

## 手动关闭：
1. 在设置中，停止服务；
2. 在设置中停止应用；
3. 系统Recent Task中的清理；
4. 用户使用第三方应用清理。

#三、如何避免进程被清除
避免进程被清除，需要程序和用户相互配合。    

##    程序需要实现：
1. 在Service类的onStartCommand中flag设置为START_STICKY，使得Service在被杀掉之后会自动重启；
2. 提供进程优先级，可以避免应用由于内存不足被系统清理。主要是通过startForeground的方式：
    * 对于 API level < 18 ：调用startForeground(ID， new Notification())，发送空的Notification ，图标则不会显示。
    * 对于 API level >= 18：在需要提优先级的service A启动一个InnerService，两个服务同时startForeground，且绑定同样的 ID。Stop 掉InnerService ，这样通知栏图标即被移除。
    备注：我尝试了几种方式，这些方式都没有效果：
    1）在AndroidManifest.xml中设置android:persistent=“true”，由于该属性只有系统应用才能使用（目前只有Phone正在使用），所以该方法不适用；
    2）在Service类的onDestroy中startService，在很多情况下，进程被杀掉的时候Service类是收不到onDestroy回调的；
3. 使用AlarmReceiver每隔300秒唤醒一次Service。微信这样做的目的是为了TCP长连接保活，对于我们来说，只要每天0点和12点发起一次网络请求，所以我觉得这一步不是必须的，但保留了参考代码；
4. 在AndroidManifest.xml中给Activity设置android:excludeFromRecents=“true”。
5. 通过系统的常用广播来自启动：BOOT_COMPLETED，CONNECTIVITY_CHANGED，USER_PRESENT；

##    用户需要配合：
1. 从Android 3.1开始，当用户设置-应用中“强行停止”应用的话，应用的是没有办法再启动的。因此，用户需要保证不在设置中强行停止应用。
2. 从系统Recent Task中清除任务，效果也类似上一条。在适配这种场景的时候，我发现为什么微信、支付宝等应用不会被系统清理掉？最后发现，这需要用户修改手机的“手机管家”应用，将我们的应用添加到白名单中。
备注：这一条规则可能在不同的手机上不一样。
3. 在Meizu手机上，按照第2调进行设置之后，测试使用360手机管家也无法停止我们的进程。
