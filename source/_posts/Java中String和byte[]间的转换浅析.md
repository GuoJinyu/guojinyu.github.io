title: Java中String和byte[]间的转换浅析
date: 2016-06-21 14:00:00 
update: 2016-08-04 11:06:00
categories: 技术
tags: [Java,Android]
---
Java语言中字符串类型和字节数组类型相互之间的转换经常发生，网上的分析及代码也比较多，本文将分析总结常规的byte[]和String间的转换以及十六进制String和byte[]间相互转换的原理及实现。
## 1. String转byte[]
首先我们来分析一下常规的String转byte[]的方法，代码如下：
``` java
public static byte[] strToByteArray(String str) {
    if (str == null) {
        return null;
    }
    byte[] byteArray = str.getBytes();
    return byteArray;
}
```
很简单，就是调用String类的getBytes()方法。看JDK源码可以发现该方法最终调用了String类如下的方法。
``` java
/* JDK source code */
public byte[] getBytes(Charset charset) {
    String canonicalCharsetName = charset.name();
    if (canonicalCharsetName.equals("UTF-8")) {
        return Charsets.toUtf8Bytes(value, offset, count);
    } else if (canonicalCharsetName.equals("ISO-8859-1")) {
        return Charsets.toIsoLatin1Bytes(value, offset, count);
    } else if (canonicalCharsetName.equals("US-ASCII")) {
        return Charsets.toAsciiBytes(value, offset, count);
    } else if (canonicalCharsetName.equals("UTF-16BE")) {
        return Charsets.toBigEndianUtf16Bytes(value, offset, count);
    } else {
        CharBuffer chars = CharBuffer.wrap(this.value, this.offset, this.count);
        ByteBuffer buffer = charset.encode(chars.asReadOnlyBuffer());
        byte[] bytes = new byte[buffer.limit()];
        buffer.get(bytes);
        return bytes;
    }
}
```
上述代码其实就是根据给定的编码方式进行编码。如果调用的是不带参数的getBytes()方法，则使用默认的编码方式，如下代码所示：
``` java
/* JDK source code */
private static Charset getDefaultCharset() {
    String encoding = System.getProperty("file.encoding", "UTF-8");
    try {
        return Charset.forName(encoding);
    } catch (UnsupportedCharsetException e) {
        return Charset.forName("UTF-8");
    }
}
```
关于默认的编码方式，Java API是这样说的：
> The default charset is determined during virtual-machine startup and typically depends upon the locale and charset of the underlying operating system.

同样，由上述代码可以看出，默认编码方式是由System类的"file.encoding"属性决定的，经过测试，在简体中文Windows操作系统下，默认编码方式为"GBK"，在Android平台上，默认编码方式为"UTF-8"。
## 2. byte[]转String
接下来分析一下常规的byte[]转为String的方法，代码如下：
``` java
public static String byteArrayToStr(byte[] byteArray) {
    if (byteArray == null) {
        return null;
    }
    String str = new String(byteArray);
    return str;
}
```
很简单，就是String的构造方法之一。那我们分析Java中String的源码，可以看出所有以byte[]为参数的构造方法最终都调用了如下代码所示的构造方法。需要注意的是Java中String类的数据是Unicode类型的，因此上述的getBytes()方法是把Unicode类型转化为指定编码方式的byte数组；而这里的Charset为读取该byte数组时所使用的编码方式。
``` java
/* JDK source code */
public String(byte[] data, int offset, int byteCount, Charset charset) {
    if ((offset | byteCount) < 0 || byteCount > data.length - offset) { 
        throw failedBoundsCheck(data.length, offset, byteCount);
    }
    // We inline UTF-8, ISO-8859-1, and US-ASCII decoders for speed and because
    // 'count' and 'value' are final.
    String canonicalCharsetName = charset.name();
    if (canonicalCharsetName.equals("UTF-8")) {
        byte[] d = data;
        char[] v = new char[byteCount];
        int idx = offset;
        int last = offset + byteCount;
        int s = 0;
        outer:
        while (idx < last) {
            byte b0 = d[idx++];
            if ((b0 & 0x80) == 0) {
                // 0xxxxxxx
                // Range:  U-00000000 - U-0000007F
                int val = b0 & 0xff;
                v[s++] = (char) val;
            } else if (((b0 & 0xe0) == 0xc0) || ((b0 & 0xf0) == 0xe0) ||
                ((b0 & 0xf8) == 0xf0) || ((b0 & 0xfc) == 0xf8) || ((b0 & 0xfe)
                == 0xfc)) {
                int utfCount = 1;
                if ((b0 & 0xf0) == 0xe0) utfCount = 2;
                else if ((b0 & 0xf8) == 0xf0) utfCount = 3;
                else if ((b0 & 0xfc) == 0xf8) utfCount = 4;
                else if ((b0 & 0xfe) == 0xfc) utfCount = 5;
                // 110xxxxx (10xxxxxx)+
                // Range:  U-00000080 - U-000007FF (count == 1)
                // Range:  U-00000800 - U-0000FFFF (count == 2)
                // Range:  U-00010000 - U-001FFFFF (count == 3)
                // Range:  U-00200000 - U-03FFFFFF (count == 4)
                // Range:  U-04000000 - U-7FFFFFFF (count == 5)
                if (idx + utfCount > last) {
                    v[s++] = REPLACEMENT_CHAR;
                    continue;
                }
                // Extract usable bits from b0
                int val = b0 & (0x1f >> (utfCount - 1));
                for (int i = 0; i < utfCount; ++i) {
                    byte b = d[idx++];
                    if ((b & 0xc0) != 0x80) {
                        v[s++] = REPLACEMENT_CHAR;
                        idx--; // Put the input char back
                        continue outer;
                    }
                    // Push new bits in from the right side
                    val <<= 6;
                    val |= b & 0x3f;
                }
                // Note: Java allows overlong char
                // specifications To disallow, check that val
                // is greater than or equal to the minimum
                // value for each count:
                //
                // count    min value
                // -----   ----------
                //   1           0x80
                //   2          0x800
                //   3        0x10000
                //   4       0x200000
                //   5      0x4000000
                // Allow surrogate values (0xD800 - 0xDFFF) to
                // be specified using 3-byte UTF values only
                if ((utfCount != 2) && (val >= 0xD800) && (val <= 0xDFFF)) {
                    v[s++] = REPLACEMENT_CHAR;
                    continue;
                }
                // Reject chars greater than the Unicode maximum of U+10FFFF.
                if (val > 0x10FFFF) {
                    v[s++] = REPLACEMENT_CHAR;
                    continue;
                }
                // Encode chars from U+10000 up as surrogate pairs
                if (val < 0x10000) {
                    v[s++] = (char) val;
                } else {
                    int x = val & 0xffff;
                    int u = (val >> 16) & 0x1f;
                    int w = (u - 1) & 0xffff;
                    int hi = 0xd800 | (w << 6) | (x >> 10);
                    int lo = 0xdc00 | (x & 0x3ff);
                    v[s++] = (char) hi;
                    v[s++] = (char) lo;
                }
            } else {
                // Illegal values 0x8*, 0x9*, 0xa*, 0xb*, 0xfd-0xff
                v[s++] = REPLACEMENT_CHAR;
            }
        }
        if (s == byteCount) {
            // We guessed right, so we can use our temporary array as-is.
            this.offset = 0;
            this.value = v;
            this.count = s;
        } else {
            // Our temporary array was too big, so reallocate and copy.
            this.offset = 0;
            this.value = new char[s];
            this.count = s;
            System.arraycopy(v, 0, value, 0, s);
        }
    } else if (canonicalCharsetName.equals("ISO-8859-1")) {
        this.offset = 0;
        this.value = new char[byteCount];
        this.count = byteCount;
        Charsets.isoLatin1BytesToChars(data, offset, byteCount, value);
    } else if (canonicalCharsetName.equals("US-ASCII")) {
        this.offset = 0;
        this.value = new char[byteCount];
        this.count = byteCount;
        Charsets.asciiBytesToChars(data, offset, byteCount, value);
    } else {
        CharBuffer cb = charset.decode(ByteBuffer.wrap(data, offset, byteCount));
        this.offset = 0;
        this.count = cb.length();
        if (count > 0) {
            // We could use cb.array() directly, but that would mean we'd have to trust
            // the CharsetDecoder doesn't hang on to the CharBuffer and mutate it later,
            // which would break String's immutability guarantee. It would also tend to
            // mean that we'd be wasting memory because CharsetDecoder doesn't trim the
            // array. So we copy.
            this.value = new char[count];
            System.arraycopy(cb.array(), 0, value, 0, count);
        } else {
            this.value = EmptyArray.CHAR;
        }
    }
}
```
具体的转换过程较为复杂，其实就是将byte数组的一个或多个元素按指定的Charset类型读取并转换为char类型（char本身就是以Unicode编码方式存储的），因为String类的核心是其内部维护的char数组。因此有兴趣的同学可以研究下各种编码方式的编码规则，然后才能看懂具体的转换过程。
## 3. byte[]转十六进制String
所谓十六进制String，就是字符串里面的字符都是十六进制形式，因为一个byte是八位，可以用两个十六进制位来表示，因此，byte数组中的每个元素可以转换为两个十六进制形式的char，所以最终的HexString的长度是byte数组长度的两倍。闲话少说上代码：
``` java
public static String byteArrayToHexStr(byte[] byteArray) {
    if (byteArray == null){
        return null;
    }
    char[] hexArray = "0123456789ABCDEF".toCharArray();
    char[] hexChars = new char[byteArray.length * 2];
    for (int j = 0; j < byteArray.length; j++) {
        int v = byteArray[j] & 0xFF;
        hexChars[j * 2] = hexArray[v >>> 4];
        hexChars[j * 2 + 1] = hexArray[v & 0x0F];
    }
    return new String(hexChars);
}
```
上述代码中，之所以要将byte数值和0xFF按位与，是因为我们为了方便后面的无符号移位操作（无符号右移运算符>>>只对32位和64位的值有意义），要将byte数据转换为int类型，而如果直接转换就会出现问题。因为java里面二进制是以补码形式存在的，如果直接转换，位扩展会产生问题，如值为-1的byte存储的二进制形式为其补码11111111，而转换为int后为11111111111111111111111111111111，直接使用该值结果就不对了。而0xFF默认是int类型，即0x000000FF，一个byte值跟0xFF相与会先将那个byte值转化成int类型运算，这样，相与的结果中高的24个比特就总会被清0，后面的运算才会正确。
## 4. 十六进制String转byte[]
没什么好说的了，就是byte[]转十六进制String的逆过程，放代码：
``` java
public static byte[] hexStrToByteArray(String str)
{
    if (str == null) {
        return null;
    }
    if (str.length() == 0) {
        return new byte[0];
    }
    byte[] byteArray = new byte[str.length() / 2];
    for (int i = 0; i < byteArray.length; i++){
        String subStr = str.substring(2 * i, 2 * i + 2);
        byteArray[i] = ((byte)Integer.parseInt(subStr, 16));
    }
    return byteArray;
}
```

文中所有代码可以在[个人github主页](https://github.com/GuoJinyu/AndroidUtils/tree/master)查看和下载。

另，转载请注明出处！文中若有什么错误希望大家探讨指正！