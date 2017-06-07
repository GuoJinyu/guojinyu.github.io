title: Android HTTPS正传
date: 2017-04-05 22:00:00
update: 2017-04-05 22:00:00
categories: 技术
tags: [Android]
---
随着HTTPS的越来越普及，我们在客户端开发过程中越来越多的被要求从HTTP转换为HTTPS，纵然你开发的是普通APP而非浏览器，正确的编写HTTPS网络代码也是至关重要的。但现实是，大部分的HTTPS相关代码中总是存在各种各样的问题，这些问题使得我们的APP在网络通信中存在严重的漏洞，从而丧失了使用HTTPS的意义。

因此，本文将从以下三个方面来展开介绍，一是对于HTTPS的理解；二是常见的HTTPS的使用误区；三是如何编写正确的HTTPS通信代码，包括使用HttpClient或HttpUrlConnection。

## HTTPS的理解

熟悉的读者可以直接跳过，简单来说HTTPS=HTTP+SSL/TLS，而SSL/TLS=加密+认证+完整性保护，HTTPS协议简单理解就是在进行传统的HTTP通信之前先通过SSL/TLS协议握手建立安全的通道。

其实SSL/TLS说白了就是密码学里的一个混合密码系统，用对称加密来保护机密性，用消息认证码(MAC)来做认证和完整性保护，同时为了解决对称密码的密钥配送问题，采用了公钥加密方法，那么为了保证公钥的合法性，就由认证机构(CA)来对公钥及相关信息进行数字签名，即我们口中的证书。

其实不只是HTTPS，很多应用层的网络传输协议为了保证安全都和SSL/TLS进行了联姻，如FTPS，采用SSL/TLS的IMAP、POP3/SMTP、Telnet协议等。

## HTTPS的使用误区

Android APP在HTTPS上存在的最大问题就是中间人攻击漏洞，所谓中间人攻击，就是攻击人介入到通信双方的中间，假冒彼此与对方就行通信，从而套取隐私信息。而导致这一漏洞的最大的原因，就是代码中对HTTPS证书的不合理使用，确切地说，就是客户端没有校验服务端的HTTPS证书（包含签名CA是否合法、域名是否匹配、是否自签名证书、证书是否过期，此处建议认真阅读[Android HTTPS中间人劫持漏洞浅析](https://jaq.alibaba.com/blog.htm?id=60)这篇博文，通俗易懂）。

常见于以下两种情况：

- 1.自定义X509TrustManager但是没有对SSL证书进行校验，如checkServerTrusted()方法实现为空，即不检查服务器是否可信；

- 2.没有对域名进行校验，如使用setHostnameVerifier(ALLOW_ALL_HOSTNAME_VERIFIER)或使用自定义HostnameVerifier但是verify()方法返回始终为True，即不校验域名。

开发人员这样写的目的往往是规避SSL异常，但是不仅没有发挥HTTPS的能力反而暴露出了漏洞。

当然，导致中间人攻击的原因可能还有其他，如CA被攻击导致私钥被盗等等，这种情况不在本文讨论范围内。

<!--more-->

## HTTPS的正确实现

如何正确地实现HTTPS，关键在于如何正确地处理证书。

前文提到，证书认证机构CA会对我们HTTPS通信中用到的公钥进行签名来制作证书，其实就是服务器有一套公私钥，CA也有一套公私钥，CA要用它的私钥对我们的公钥就行数字签名，此即为认证。所以一份证书的主要内容便是三部分，a、服务器生成的公钥，b、申请证书时填写的一些相关信息，c、CA用自己的私钥对a和b内容生成的数字签名。那么当客户端在验证服务端的证书时，客户端必须拥有证书CA的公钥，才能去验证签名。

所以根据客户端是否拥有CA的公钥，处理证书便分为以下几种情况：

### 1. CA是受信任的

这是HTTPS使用中最推荐的方式，即使用知名的证书颁发机构如Symantec(收购了VeriSign)，这类颁发机构一般在浏览器或者Android系统内置的信任CA列表里，这就意味着客户端拥有该CA的公钥，可以直接按照HTTPS握手协议从服务端获取证书，并对该CA认证的证书进行验签，获取服务端公钥。当然不足就是此类证书可能是需要收费的。

代码实现如下：

- HttpURLConnection(该API在Android 4.4版本及以后已替换为OkHttp实现)

```java
    HttpURLConnection urlConnection = null;
    try {
        URL url = new URL("https://www.baidu.com");
        urlConnection = (HttpURLConnection) url.openConnection();
        InputStream in = urlConnection.getInputStream();
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        if (urlConnection != null) {
            urlConnection.disconnect();
        }
    }
```

这种情况代码实现简单，其实和使用普通的HTTP没有任何区别，因为当url是HTTPS格式时，url.openConnection()返回的实际是HttpsURLConnection对象。

- HttpClient(Android 6.0版本及以后已去除该框架)

```java
    HttpClient httpClient = new DefaultHttpClient();
    HttpGet httpGet = new HttpGet("https://www.baidu.com/");
    try {
        HttpResponse httpResponse = httpClient.execute(httpGet);
        if (httpResponse.getStatusLine().getStatusCode() == 200){
            HttpEntity entity = httpResponse.getEntity();
            String response = EntityUtils.toString(entity,"utf-8");
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
```

同样和HTTP代码的编写没有任何区别。

我们这里说的第一种方式前提是颁发证书的CA是受信任的，即使用HTTPS协议的设备或浏览器已信任目标服务端证书的签名机构。可以看到这种情况下代码的编写很简单，但是，在日常Android开发中，如此简单的方式未必能覆盖我们的需求，为什么这样说呢？由于Android碎片化现象的严重，不同版本不同设备信任的CA列表可能不尽相同，所以往往会出现一套代码在这个设备上运行正常，但在另一个设备上就javax.net.ssl.SSLException了。

### 2. 未知的CA

所谓未知的CA包含三种情况，一是设备未添加到信任列表的公共CA，二是政府、公司或教育机构等组织发放的私有CA(国内很常见，如12306网站的证书)，三是自签名的证书(这种情况严格来说并不是CA的认证，而是自己使用Keytool或OpenSSL签发证书，好比自己是自己的CA)。由于这三种情况在开发中的处理差别不大，因此归为一类。对于此类CA，通常的做法是本地内置CA证书文件或内容字符串。

- HttpURLConnection

```java
    InputStream caInput = null;
    Certificate ca = null;
    try {
        // Load CAs from an InputStream
        // (could be from a resource or ByteArrayInputStream or ...)
        CertificateFactory cf = CertificateFactory.getInstance("X.509");
        caInput = context.getAssets().open("srca.cer");
        ca = cf.generateCertificate(caInput);
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        try {
            caInput.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    try {
        // Create a KeyStore containing our trusted CAs
        String keyStoreType = KeyStore.getDefaultType();
        KeyStore keyStore = KeyStore.getInstance(keyStoreType);
        keyStore.load(null, null);
        keyStore.setCertificateEntry("ca", ca);
        // Create a TrustManager that trusts the CAs in our KeyStore
        String tmfAlgorithm = TrustManagerFactory.getDefaultAlgorithm();
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(tmfAlgorithm);
        tmf.init(keyStore);
        // Create an SSLContext that uses our TrustManager
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, tmf.getTrustManagers(), null);
        // Tell the URLConnection to use a SocketFactory from our SSLContext
        URL url = new URL("https://kyfw.12306.cn/otn/");
        HttpsURLConnection urlConnection = (HttpsURLConnection) url.openConnection();
        urlConnection.setSSLSocketFactory(sslContext.getSocketFactory());
        InputStream in = urlConnection.getInputStream();
    } catch (Exception e) {
        e.printStackTrace();
    }
```

上面代码中srca.cer证书文件是从12306官网下载的，当然也可以获取到证书内容并在代码里定义为字符串常量。

- HttpClient

```java
    InputStream caInput = null;
    Certificate ca = null;
    try {
        // Load CAs from an InputStream
        // (could be from a resource or ByteArrayInputStream or ...)
        CertificateFactory cf = CertificateFactory.getInstance("X.509");
        caInput = context.getAssets().open("srca.cer");
        ca = cf.generateCertificate(caInput);
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        try {
            caInput.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    try {
        // Create a KeyStore containing our trusted CAs
        String keyStoreType = KeyStore.getDefaultType();
        KeyStore keyStore = KeyStore.getInstance(keyStoreType);
        keyStore.load(null, null);
        keyStore.setCertificateEntry("ca", ca);
        SSLSocketFactory ssl = new SSLSocketFactory(keyStore);
        ssl.setHostnameVerifier(ssl.getHostnameVerifier());
        Scheme scheme = new Scheme("https", ssl, 443);
        HttpClient httpclient = new DefaultHttpClient();
        httpclient.getConnectionManager().getSchemeRegistry().register(scheme);
        HttpGet httpGet = new HttpGet("https://kyfw.12306.cn/otn/");
        HttpResponse httpResponse = httpclient.execute(httpGet);
        if (httpResponse.getStatusLine().getStatusCode() == 200) {
            HttpEntity entity = httpResponse.getEntity();
            String response = EntityUtils.toString(entity, "utf-8");          
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
```

与HttpURLConnection相比，其实代码套路基本一致，都是先读取证书生成一个Certificate对象，然后根据证书构建KeyStore对象，最后构建SSLSocketFactory对象并和HTTP协议关联起来。

### 3. 缺少中间CA

何谓缺少中间证书颁发机构？其实大多数公共CA不直接签署服务器证书，而是使用自己的根CA签署中间 CA，然后用中间CA去满足我们的证书签发的请求，以降低根CA泄露风险。那么问题来了，Android 等操作系统通常仅直接信任根CA，因此中间CA就又变成了未知CA。解决这个问题主要有两个办法，分别为服务端解决和客户端解决（参考谷歌官方文档[通过HTTPS和SSL确保安全](https://developer.android.com/training/articles/security-ssl.html)）。

- 服务端
配置服务器以便在服务器链中添加中间CA，使得服务器在SSL 握手期间不只向客户端发送它的证书，而是发送一个证书链，包括服务器CA以及到达可信的根CA所需要的任意中间证书。

- 客户端
不用我说大家都知道了吧，按照前文，像对待其他任何未知CA一样对待中间CA。

对于Android开发中HTTPS的实现方式，本文分HttpURLConnection和HttpClient两套框架分别做了讲解，但实际上我们用到更多的可能是其他的一些HTTP协议开源框架，如okhttp、retrofit、volley、android-async-http等，其实万变不离其宗，这些框架大多是互相包含的关系，因此在HTTPS使用上也没有太大差别，顶多就是有些已经做了一定程度的封装，网上相关的资料也有很多，只要理解其背后的含义，相信都不难实现。

此外，在实际的HTTPS开发中我们遇到的需求或者问题可能千奇百怪，有时候我们不得不通过自定义TrustManager或HostnameVerifier来实现，但是切记这是有风险的，一定要尽可能地完成校验（签名CA是否合法、域名是否匹配、是否自签名证书、证书是否过期），在力所能及的范围内写出最为安全的代码。

文中所有代码可以在[个人github主页](https://github.com/GuoJinyu/AndroidUtils/blob/master/HttpsUtil.java)查看和下载。

另，转载请注明出处！文中若有什么错误希望大家探讨指正！


