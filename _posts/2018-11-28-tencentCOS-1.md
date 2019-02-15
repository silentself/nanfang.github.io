---
layout: post
title:  tencentCOS对象存储（springboot+springMVC）
excerpt: "腾讯云出的对象存储sdk真心感觉有点坑，那个所谓的高级存储，虽然时将连接对象池化了，但是缺只能接收File类型"
categories: [java]
tags: [COS]
comments: true
---

#### 写在前面：

腾讯云java sdk关于上传对象给出的就两种，一个是接收InputStream另一个是接收File，流的形式没啥说的，主要是这个File，如果是用springMvc接收一般都是使用MultipartFile类型来接收这样就需要先吧文件临时缓存到服务器上再上传，感觉相当麻烦

#### 1、使用java sdk编写工具类

1）引入pom依赖

```xml
<!-- 腾讯云SDK -->
<dependency>
    <groupId>com.qcloud</groupId>
    <artifactId>cos_api</artifactId>
    <version>5.4.6</version>
</dependency>
```

2)编写工具类

①必须参数，共有读取私有写，需要提供签名

```java
public interface COSConstant {
    // 初始化秘钥信息（用来验证签名）
    public static final String SECRETID = "*************";
    public static final String SECRETKEY = "*************";
    // 设置APPID(根目录文件夹名+appid)
    public static final String APPID = "*********";
    // 设置默认桶（即文件存储根目录）
    public static final String BUCKETNAME = "**********";
    // 设置区域(腾讯接口文档有地区对应的编号)
    public static final String REGIONID = "ap-beijing";
    // 设置文件url前缀
    public static final String URL_PREFIX = "https://<桶>-<appid>.cos.ap-beijing.myqcloud.com/";
}
```

②返回连接对象COSClient

```java
public static COSClient getCOSClient() {
    // 初始化秘钥信息
    COSCredentials cred = new BasicCOSCredentials(COSConstant.SECRETID, COSConstant.SECRETKEY);
    // 初始化客户端配置
    ClientConfig clientConfig = new ClientConfig();
    // 初始化客户端配置(如设置园区)
    Region region = new Region(COSConstant.REGIONID);
    clientConfig.setRegion(region);
    // 初始化cosClient
    COSClient cosClient = new COSClient(cred, clientConfig);
    return cosClient;
}
```

③上传文件的通用方法（InputStream）

```java
    /**
	 * 方式1 将本地文件上传到 COS 简单文件上传, 最大支持 5 GB, 适用于小文件上传, 建议 20M以下的文件使用该接口
	 * 
	 * @param bucketName
	 * @param key
	 * @param file
	 * @return
	 * @throws Exception
	 */
public static String putObject(String bucketName, String key, MultipartFile file) throws Exception {
    if (file.getSize() > 30 * 1024 * 1024) {
        throw new Exception("上传图片大小不能超过10M！");
    }
    try {
        InputStream inputStream = file.getInputStream();
        String url = putObject(bucketName, key, inputStream);
        return url;
    } catch (Exception e) {
        logger.error("putObject 文件上传失败 --->{}", e.getMessage());
        return null;
    }
}
    /**
	 * 方式2 输入流上传到 COS,并返回文件的url
	 */
public static String putObject(String bucketName, String key, InputStream inputStream) throws IOException {
    // 方法 2 从输入流上传(需提前告知输入流的长度, 否则可能导致 oom)
    // 创建上传Object的Metadata
    ObjectMetadata objectMetadata = new ObjectMetadata();
    // 设置输入流长度为 ，否侧可能会出现OOM异常
    objectMetadata.setContentLength(inputStream.available());
    objectMetadata.setCacheControl("no-cache");
    objectMetadata.setHeader("Pragma", "no-cache");
    // 设置 Content type, 默认是 application/octet-stream
    objectMetadata.setContentType(getcontentType(key.substring(key.lastIndexOf("."))));
    objectMetadata.setContentDisposition("inline;filename=" + key);
    COSClient cosClient = getCOSClient();
    try {
        cosClient.putObject(bucketName, key, inputStream, objectMetadata);
        String url = String.valueOf(COSConstant.URL_PREFIX + key);
        return url;
    } catch (Exception e) {
        logger.error("putObject 文件上传失败 --->{}", e.getMessage());
        return null;
    } finally {
        // 关闭流
        cosClient.shutdown();
    }
}
```

#### 3、controller接口

```java
   /**
	 * 上传图片，并返回图片的url
	 * 
	 * @param file
	 * @return
	 */
@PostMapping(value = "/uploadImage", consumes = "multipart/form-data", produces = "application/json")
public UploadInfo uploadImage(MultipartFile file, HttpServletRequest request, HttpServletResponse response) {
    String roleAndPermission = "";
    UserSimpleForm userSimpleForm = new UserSimpleForm();
    if (null == file) {
        logger.error("上传图片失败-- uploadImage ---->{}", COSConstant.PARAMS_EMPTY);
        return UploadInfo.dofailure(COSConstant.PARAMS_EMPTY, userSimpleForm, roleAndPermission);
    }
    String name = file.getOriginalFilename();
    if (null == name) {
        logger.error("上传图片失败-- uploadImage ---->{}", COSConstant.FILENAME_FORMAT_ERROR);
        return UploadInfo.dofailure(COSConstant.FILENAME_FORMAT_ERROR, userSimpleForm, roleAndPermission);
    }
    // image（video）+fileName
    String key = String.valueOf(COSConstant.IMAGE + name);
    String url = null;
    try {
        url = COSUtil.putObject(COSConstant.BUCKETNAME, key, file);
    } catch (Exception e) {
        logger.error("上传图片失败-- uploadImage ---->{}", e.getMessage());
        return UploadInfo.dofailure(e.getMessage(), userSimpleForm, roleAndPermission);
    }
    return UploadInfo.dosuccess(url, userSimpleForm, roleAndPermission);
}
```

#### 4、可选配置（application*.yml）

```yml
 spring: 
  servlet:
    multipart:
      max-file-size: 30Mb #根据自己情况设置
      max-request-size: 30Mb
```

#### 最后：吐槽一下官方给出的高级上传方法，传言适合大文件上传

##### 1、将COSClient池化

```java
   /**
	 * 将COSClient池化，并返回连接池
	 */
public static TransferManager getTransferManager() {
    // 线程池大小，建议在客户端与COS网络充足(如使用腾讯云的CVM，同园区上传COS)的情况下，设置成16或32即可, 可较充分的利用网络资源
    // 对于使用公网传输且网络带宽质量不高的情况，建议减小该值，避免因网速过慢，造成请求超时。
    ExecutorService threadPool = Executors.newFixedThreadPool(32);
    COSClient cosClient = getCOSClient();
    // 传入一个 threadpool, 若不传入线程池, 默认 TransferManager 中会生成一个单线程的线程池。
    TransferManager transferManager = new TransferManager(cosClient, threadPool);
    return transferManager;
}
```

##### 2、高级文件上传

```java
   /**
	 * 方式3 多线程高级上传文件（需要临时缓存到当前服务器上，这点有点坑d）
	 */
@SuppressWarnings("unused")
public static String upload(String bucketName, String key, MultipartFile file) throws Exception {
    if (StringUtils.isEmpty(bucketName) || StringUtils.isEmpty(key) || null == file) {
        throw new Exception("存在空参数！");
    }
    TransferManager transferManager = getTransferManager();
    try {
        String tempName = UUID.randomUUID().toString();
        File localFile = File.createTempFile(tempName, null,new File(COSConstant.TEMP_ROOT));
        file.transferTo(localFile);
        // 获取链接
        final PutObjectRequest putObjectRequest = new PutObjectRequest(bucketName, key, localFile);
        // 本地文件上传
        Upload upload = transferManager.upload(putObjectRequest);
        // 等待传输结束（如果想同步的等待上传结束，则调用 waitForCompletion）
        UploadResult uploadResult = upload.waitForUploadResult();
        //正常退出程序时删除文件
        localFile.deleteOnExit();
        String url = String.valueOf(COSConstant.URL_PREFIX + key);
        return url;
    } catch (Exception e) {
        logger.error("upload 文件上传失败 --->{}", e.getMessage());
        return null;
    } finally {
        // 关闭 TransferManger
        transferManager.shutdownNow();
    }
}
```

##### 附加1根据文件后缀匹配媒体类型

```java
public static String getcontentType(String filenameExtension) {
    if (filenameExtension.equalsIgnoreCase("bmp")) {
        return "image/bmp";
    }
    if (filenameExtension.equalsIgnoreCase("gif")) {
        return "image/gif";
    }
    if (filenameExtension.equalsIgnoreCase("jpeg") || filenameExtension.equalsIgnoreCase("jpg")
        || filenameExtension.equalsIgnoreCase("png")) {
        return "image/jpeg";
    }
    if (filenameExtension.equalsIgnoreCase("html")) {
        return "text/html";
    }
    if (filenameExtension.equalsIgnoreCase("txt")) {
        return "text/plain";
    }
    if (filenameExtension.equalsIgnoreCase("vsd")) {
        return "application/vnd.visio";
    }
    if (filenameExtension.equalsIgnoreCase("pptx") || filenameExtension.equalsIgnoreCase("ppt")) {
        return "application/vnd.ms-powerpoint";
    }
    if (filenameExtension.equalsIgnoreCase("docx") || filenameExtension.equalsIgnoreCase("doc")) {
        return "application/msword";
    }
    if (filenameExtension.equalsIgnoreCase("xml")) {
        return "text/xml";
    }
    return "image/jpeg";
}
```



