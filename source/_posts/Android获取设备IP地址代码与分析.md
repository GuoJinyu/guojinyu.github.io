title: Android获取设备IP地址代码与分析
date: 2016-07-26 19:00:00 
update: 2016-08-04 11:06:00
categories: 技术
tags: [Java,Android]
---
一直以来，好像没有一段标准的代码能提供Android设备此刻的IP地址，究其原因，Android设备的网卡可能不只一个，如蜂窝网卡、WiFi网卡，而且同一个网卡也可能拥有不止一个IP地址。基于此，一个Android终端很有可能同时拥有多个IP地址（不只是同时拥有IPv4和IPv6地址），比如开启热点共享蜂窝网络的时候，蜂窝网卡拥有一个IPv4地址来访问外网，WiFi网卡拥有一个IPv4地址来作为内网的网关。

网上比较流行的获取Android设备IP地址的代码有以下几种，下面我们来一一分析一下。

## 1. 不可行的方法
```java
String ipAddress = Inet4Address.getLocalHost().getHostAddress()
```
这个是Java提供的API，在Android上执行需要以下权限（经测试Android版本6.0.1的一部机器不需要该权限，比较纳闷，求解答）
```xml
<uses-permission android:name="android.permission.INTERNET"/>
```
此外，由于该方法使用了网络通信，因此不能在UI线程执行。

该方法顾名思义是获取本地主机的IP地址，在某些Java平台上可以得到想要的结果，但是我截取了Android官方给出的关于该方法的部分说明如下：
> Returns an InetAddress for the local host if possible, or the loopback address otherwise. This method works by getting the hostname, performing a DNS lookup, and then taking the first returned address. 
Note that if the host doesn't have a hostname set – as Android devices typically don't – this method will effectively return the loopback address, albeit by getting the name localhost and then doing a lookup to translate that to 127.0.0.1.

可以看出，一般在Android平台上，由于网络通信设备没有设置hostname，因此无法进行DNS检索得到其相应的IP地址，因此该方法会返回本地回环地址，即127.0.0.1，也就是说这个方法在Android平台上无法达到我们一般的获取本机IP地址的目的，经过测试，结果也确实如此。
## 2. 部分可行的方法
```java
WifiManager wm = (WifiManager) getSystemService(WIFI_SERVICE);
int ipAddressInt = wm.getConnectionInfo().getIpAddress();
String ipAddress = String.format(Locale.getDefault(), "%d.%d.%d.%d", (ipAddressInt & 0xff), (ipAddressInt >> 8 & 0xff), (ipAddressInt >> 16 & 0xff), (ipAddressInt >> 24 & 0xff));
```
方法执行所需权限为：
```xml
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
```
需要说明的是，上述代码第二行返回的是一个int类型的值，如1795336384，它对应的十六进制值6b02a8c0每两位便对应IPv4地址的每一项（逆序，如c0转化为十进制为192）。

经测试，通过该方法可以获得当前WiFi网络中Android设备的IPv4地址，但是显然，该方法是通过WifiManager获取当前网络连接下的IP地址的，因此它只局限于使用WiFi网络的情况，当使用蜂窝等其他网络设备时，该方法无效，会返回0值。另外，如果你是通过比较hacker的方式比如没有通过系统Framework层打开WiFi，而是自己通过Linux命令创建的WiFi网络,那么像这种Framework层提供的API也是不起作用的。
## 3. 基本可行的方法
```java
    public static String getIpAddressString() {
        try {
            for (Enumeration<NetworkInterface> enNetI = NetworkInterface
                    .getNetworkInterfaces(); enNetI.hasMoreElements(); ) {
                NetworkInterface netI = enNetI.nextElement();
                for (Enumeration<InetAddress> enumIpAddr = netI
                        .getInetAddresses(); enumIpAddr.hasMoreElements(); ) {
                    InetAddress inetAddress = enumIpAddr.nextElement();
                    if (inetAddress instanceof Inet4Address && !inetAddress.isLoopbackAddress()) {
                        return inetAddress.getHostAddress();
                    }
                }
            }
        } catch (SocketException e) {
            e.printStackTrace();
        }
        return "";
    }
```
方法执行所需权限为：
```xml
<uses-permission android:name="android.permission.INTERNET"/>
```
这段代码不难理解，其实就是双重循环获取终端中所有网络接口的所有IP地址，然后返回第一个遇到的非本地回环的IPv4地址。这种方式可以很好的覆盖我们一般的需求。根据Android系统的运行机制，当WiFi网络开启时蜂窝网络会自动关闭，因此遍历到的第一个地址是WiFi网卡的IP地址；同样，当关闭WiFi网络，打开蜂窝网络时，遍历到的第一个地址是蜂窝网卡的IP地址。

那么，为什么我叫这种方式为基本可行的方法呢，因为它返回的结果并不是百分百“正确”的，确切地说并不一定是开发人员想要的结果。比如当Android手机开启热点的时候，实际上是通过WiFi网卡共享其蜂窝网络，因此此时，WiFi网卡和蜂窝网卡分配了不同的IP地址，但由于蜂窝网卡对应的NetworkInterface对象出现的位置要先于WiFi网卡，因此该方法返回的实际上是蜂窝网卡的IP地址。如果想要始终获取WiFi网卡的IP地址可以在上述的两个循环间添加如下筛选代码：
```java
if (netI.getDisplayName().equals("wlan0") || netI.getDisplayName().equals("eth0"))
```
其中"wlan0"和"eth0"为常见的WLAN网卡的DisplayName名称，绝大部分为"wlan0"，比较老的机型可能会是"eth0"或其他。

这里只是举了一个简单的例子，其实还有很多特殊的情况，比如开启USB网络共享的情况、开启网络代理的情况、之前提到的Hacker手段同时打开蜂窝网络和WiFi网络（非WiFi热点）的情况等等，这些网络环境下都会存在多IP的情况，因此该方法不一定完全适用了。

正如文章开头所说，由于一个Android设备同一时刻可能不只有一个IP地址，因此可以说没有任何一段通用的代码能获取每个人心中想要获取的IP地址，重要的还是根据自己具体的需求来进行相应的代码修改，通过对获取的IP地址列表进行筛选来得到想要的结果。

本文的讨论是围绕IPv4地址的，如果想要获取IPv6地址，Android API也提供了相应的类或方法，只需要在上述代码的基础上作出微小修改即可。

最后附上在[StackOverFlow](http://stackoverflow.com/questions/9481865/getting-the-ip-address-of-the-current-machine-using-java?noredirect=1&lq=1)上看到的关于IP地址筛选的总结，供大家参考。
> * Any address in the range 127.xxx.xxx.xxx is a "loopback" address. It is only visible to "this" host.
> * Any address in the range 192.168.xxx.xxx is a private (aka site local) IP address. These are reserved for use within an organization. The same applies to 10.xxx.xxx.xxx addresses, and 172.16.xxx.xxx through 172.31.xxx.xxx.
> * Addresses in the range 169.254.xxx.xxx are link local IP addresses. These are reserved for use on a single network segment.
> * Addresses in the range 224.xxx.xxx.xxx through 239.xxx.xxx.xxx are multicast addresses.
> * The address 255.255.255.255 is the broadcast address.
> * Anything else should be a valid public point-to-point IPv4 address.

文中所有代码可以在[个人github主页](https://github.com/GuoJinyu/AndroidUtils/tree/master)查看和下载。

另，转载请注明出处！文中若有什么错误希望大家探讨指正！