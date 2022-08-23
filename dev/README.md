# 这是空白的占位模板md文件

```java
// 调用 COS 接口之前必须保证本进程存在一个 COSClient 实例，如果没有则创建
// 详细代码参见本页：简单操作 -> 创建 COSClient
COSClient cosClient = createCOSClient();

// 存储桶的命名格式为 BucketName-APPID，此处填写的存储桶名称必须为此格式
String bucketName = "examplebucket-1250000000";
// 对象键(Key)是对象在存储桶中的唯一标识。详情请参见 [对象键](https://cloud.tencent.com/document/product/436/13324)
String key = "exampleobject";

GeneratePresignedUrlRequest req =
        new GeneratePresignedUrlRequest(bucketName, key, HttpMethodName.GET);

// 设置下载时返回的 http 头
ResponseHeaderOverrides responseHeaders = new ResponseHeaderOverrides();
String responseContentType = "image/x-icon";
String responseContentLanguage = "zh-CN";
// 设置返回头部里包含文件名信息
String responseContentDispositon = "filename=\"exampleobject\"";
String responseCacheControl = "no-cache";
String cacheExpireStr =
        DateUtils.formatRFC822Date(new Date(System.currentTimeMillis() + 24L * 3600L * 1000L));
responseHeaders.setContentType(responseContentType);
responseHeaders.setContentLanguage(responseContentLanguage);
responseHeaders.setContentDisposition(responseContentDispositon);
responseHeaders.setCacheControl(responseCacheControl);
responseHeaders.setExpires(cacheExpireStr);

req.setResponseHeaders(responseHeaders);

// 设置签名过期时间(可选)，若未进行设置，则默认使用 ClientConfig 中的签名过期时间(1小时)
// 这里设置签名在半个小时后过期
Date expirationDate = new Date(System.currentTimeMillis() + 30L * 60L * 1000L);
req.setExpiration(expirationDate);

// 填写本次请求的参数
req.addRequestParameter("param1", "value1");
// 填写本次请求的头部
// host 必填
req.putCustomRequestHeader(Headers.HOST, cosClient.getClientConfig().getEndpointBuilder().buildGeneralApiEndpoint(bucketName));
req.putCustomRequestHeader("header1", "value1");

URL url = cosClient.generatePresignedUrl(req);
System.out.println(url.toString());

// 确认本进程不再使用 cosClient 实例之后，关闭之
cosClient.shutdown();
```

