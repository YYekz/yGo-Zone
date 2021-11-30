# adb操作

### adb是什么

adb在某种程度上有很大权限的，因为adb的设计初衷是为了方便开发人员调试，因此必然需要暴露一些权限以外的接口，利用这个特性可以**绕开权限机制**在非Root非定制机上做一些操作。

**adb架构**

**adb是一个C/S架构的应用程序，由三部分组成**

**(1).运行在pc端的adb client**

命令行程序”adb”用于从shell或脚本中运行adb命令。首先，“adb”程序尝试定位主机上的adb服务器，如果找不到adb服务器，“adb”程序自动启动一个adb服务器。接下来，当设备的adbd和pc端的adb server建立连接后，adb client就可以向adb servcer发送服务请求；

**(2).运行在pc端的adb server**

adb Server是运行在主机上的一个后台进程。它的作用在于检测USB端口感知设备的连接和拔除，以及模拟器实例的启动或停止，adb Server还需要将adb client的请求通过usb或者tcp的方式发送到对应的adbd上；

**(3)运行在设备端的常驻进程adb demon(adbd)**

程序“adbd”作为一个后台进程在Android设备或模拟器系统中运行。它的作用是连接adb服务器，并且为运行在主机上的客户端提供一些服务；

adb默认端口为5037，若端口被占用，解决办法为：

1. 找到使用该端口的进程pid：netstat -apn|grep 5037（windows下：`netstat -aon|findstr 5037`）
2. 通过pid找到对应的进程名（便于定位，可以跳过）：`tasklist /fi "pid eq 5162"`
3. 使用命令终止该命令的运行：`taskkill /pid 5162 /f`

### adb操作

1. adb启动服务：adb start-server
2. adb终止服务：adb kill-server
3. 列出当前进程：adb shell ps | grep 包名
4. 杀死某个包的进程：adb shell am force-stop 包名
5. 获取内存：adb shell dumpsys meminfo 包名
6. 获取cpu：adb shell dumpsys cupinfo 包名 (adb shell top | grep 包名)
7. 获取流畅度：adb shell dumpsys gfxinfo 包名
8. 获取当前显示的activity：adb shell dumpsys activity top | grep ACTIVITY
9. 查询所有pid状态和地址：netstat -v
10. 查看当前监听的TCP连接：netstat -anvp tcp | grep LISTEN
11. 查看当前正在运行的service：adb shell dumpsys activity services <packagename>
12. 查看应用详细信息：adb shell dumpsys package <packagename>
13. 调起Activity：adb shell am start -n <activityName>
14. 冷启动Activity：adb shell am start -S -W <包名/类名> （-S表示重启当前应用）
15. 调起Service：adb shell am startservice -n <serviceName>
16. 发送广播：adb shell am boradcast -a android.intent.action.BOOT_COMPLETED -n "boradcastName"
17. 强制停止应用：adb shell am force-stop <packageName>
18. 禁用应用和启动：
    1. adb shell pm disable-user <packageName>
    2. adb shell pm disable <packageName>
    3. adb shell pm enable <packageName>
19. 向应用授予权限（只能授予应用已经声明的权限）：adb shell pm grant <packageName> <permission>
20. 取消应用授权：adb shell pm revoke <packageName> <permission>
21. 滑动解锁：adb shell input swipe 300 1000 300 500
22. 点亮屏幕：adb shell input keyevent 224 （熄灭屏幕223）
23. 关闭USB调试模式：adb shell settings put global adb_enabled 0
24. 重启手机：adb reboot
25. 重启到recovery模式：adb reboot recovery
    1. 从recovery重启到Android：adb reboot
    2. 重启到FastBoot模式：adb reboot bootloader
26. 开启、关闭wifi：adb shell svc wifi enable/disable
27. 开启、关闭流量：adb shell svc data enable/disable
28. 使用monkey进行压力测试：adb shell monkey -p <packageName> -v 500
29. 查看实时资源占用情况：adb shell top

> windows findstr字段替换成linux中grep即可。
> 

<aside>
💡 更多模拟手机按键输入操作：[https://www.cnblogs.com/zhuminghui/p/10457755.html](https://www.cnblogs.com/zhuminghui/p/10457755.html)

</aside>

### shell操作

1. 进入shell：adb shell
2. 退出shell：exit
3. 删除文件：rm 文件名
4. 强制删除文件：rm -f 文件名
5. 没有权限但进入某个文件下：run-as packagename

### 手机信息操作

1. 获取序列号：adb get-serialno
2. 查看设备型号：adb shell getprop ro.product.model
3. 查看android系统版本：adb shell getprop ro.build.version.release
4. 查看屏幕分辨率：adb shell wm size
5. 查看屏幕密度：adb shell wm density
6. 查看权限：adb shell pm list permissions
7. 查看危险权限：adb shell pm list permissions -d -g
8. 查看电池状况：adb shell dumpsys battery
9. 查看显示屏参数：adb shell dumpsys window displays
10. 获取android_id：adb shell settings get secure android_id
11. 获取IP地址：adb shell ifconfig "| grep Mask"
12. 获取CPU信息：adb shell cat /proc/cpuinfo
13. 获取内存信息：adb shell cat /proc/meminfo

### 查看操作

1. 查看adb版本：adb version
2. 查询当前连接的所有Android设备：adb devices -l
3. 查看手机中所有app包名：adb shell pm list packages
4. 查看所有系统应用的包名：adb shell pm list packages -s
5. 查看手机中所有三方应用的包名：adb shell pm list packages -3

### 卸载操作

1. 卸载包：adb uninstall 包名
2. 卸载包但保留配置和缓存：adb uninstall -k 包名
3. 如果连接多个android设备，则需要使用-s指定具体手机，如：adb -s 设备 uninstall 包名
4. 清楚应用数据缓存： adb shell pm clear 包名

### 安装操作

1. 安装包：adb install apk位置
2. 覆盖安装包（保留数据和缓存文件）：adb install -r apk位置
3. 安装包在android设备内：adb shell pm install apk位置

一些参数命令，例如

- -d：允许降级覆盖安装
- -g：授予所有运行时权限
- -r ：允许覆盖安装
- -t ：允许安装清单文件中 application 指定android:testOnly="true"的应用

### 文件操作

1. 电脑文件发送到手机：adb push local remote
    
    > eg: adb push yyqtest.txt /sdcard/
    > 
2. 上传手机文件到电脑：adb pull remote local
    
    > eg: adb pull /sdcard/yyqtest.txt /
    > 

### 截屏、录屏

1. 截屏：adb shell screencap /sdcard/yyqtest.png
2. 录屏：adb shell screenrecord /sdcard/yyqtest.mp4

### 相关文章

[玩转Android adb命令](https://mp.weixin.qq.com/s/tlWQdThXzEPVACG4047eLw)