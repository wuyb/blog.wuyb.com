---
title: 微信支付历险记
date: 2017-1-10 16:20:10
tags: [tech]
layout: true
---

之前一直听说微信支付的 API 是个坑货，坑到有专门的封装支付的平台。这次自己体验了一下，果然很坑。做个笔记。

<!-- more -->

首先必须说一下，无论什么结果都返回 `200 OK` 是个很脑残的偷懒设计。

**统一下单接口返回空**
在调试统一下单接口的时候，发现微信返回 `200 OK` 但是不会去调用回调接口。这就有点尴尬了。我一直以为接口调用是成功的，只是回调接口没配置好（必须是外网服务器，所以用了第三方的 tunnel）。然而无论怎么试，外网接口都是通的。随后看文档发现，如果调用成功是应该返回 success 的，而不是空。这才想到会不会调用就是失败的。仔细看了一下代码，发现 `Java POJO -> XML` 的转化写的有问题。`XStream` 会自动给把下划线转义，变成俩下划线。而我为了简单，直接把 `POJO` 里的变量都按接口要求对应了（即按规范应该是 `totalFee` 的地方，都写了 `total_fee`）。重载了一下 `PrettyPrintWriter` 的 `encodeNode` 函数直接返回变量名，终于，看到返回内容了。

坑点：失败了就不要返回 `200 OK` 嘛！返回 `200 OK` 也行，起码不要把错误扔掉嘛！另外，那个回调也并不是在统一下单接口调用的。文档写得不明不白。

**统一下单接口签名错误**
我出的问题是签名时没有对字符串做正确的编码处理。简单说，MD5Digest 接收的是一个字节数组，需要将字符串转换一下，转换时需要加入编码参数：
```
byte[] sign = digest.digest(buffer.toString().getBytes()); // 老实说，这是一个非常弱智的错误
byte[] sign = digest.digest(buffer.toString().getBytes("UTF-8"));
```
签名函数的主逻辑是这样的：
```
/**
 * Signs the given parameters according to wechat payment unfiedorder protocol.
 *
 * @param params the request parameters, key and sign are not included
 *               neither are the null/empty values
 */
public String sign(SortedMap params) {
  // combine the parameters to get the string to be signed on
  StringBuffer buffer = new StringBuffer();
  Iterator iterator = params.entrySet().iterator();
  while (iterator.hasNext()) {
      Map.Entry entry = (Map.Entry) iterator.next();
      String key = (String) entry.getKey();
      String value = (String) entry.getValue();
      buffer.append(key).append("=").append(value).append("&");
  }

  // append the wechat key
  buffer.append("key=").append(wechatKey);

  byte[] sign = digest.digest(buffer.toString().getBytes("UTF-8"));

  // convert the byte array to hex string
  StringBuffer hexString = new StringBuffer();
  for (int i=0; i < sign.length; i++) {
      String hex = Integer.toHexString(0xff & sign[i]);
      if (hex.length() == 1) {
          hexString.append('0');
      }
      hexString.append(hex);
  }

  // the result string must be upper case
  return hexString.toString().toUpperCase();
}
```

主要的问题就这俩，其他的基本上按照文档来就可以了。但是微信支付的接口明显是，第一，没有认真设计过，参数名大小写混着来，有的时候是 `timestamp`， 有的时候是 `timeStamp`。第二，抄支付宝（或者支付宝抄它），变量名都差不多。另外那个回调机制基本上不能保证什么，最靠谱的确认支付状态的方式是服务端轮询。
