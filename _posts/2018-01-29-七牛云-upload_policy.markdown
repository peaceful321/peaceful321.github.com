---
layout: post
title:  "七牛云-上传策略常用示例"
categories: qiniu
tags: Java, qiniu
author: xuhuanchao
description: 七牛云 上传策略常见业务需求实现 By Java
---

# 七牛云-上传策略常用示例

## 概述
七牛提供的“上传策略”，其实就是资源上传时附带的一组配置设定；通过上传策略，开发者可以设定上传的文件类型、大小； 可以设置上传成功后回调业务服务器的地址及返回什么样的回调信息，还可以设置返回值（包含自定义返回值）；还有其他的相关配置设定， 具体可以查看七牛的开发者中心关于[上传策略](https://developer.qiniu.com/kodo/manual/1206/put-policy)的详细说明。

## 使用示例

### returnBody 代码示例：

使用七牛的java sdk 进行上传，默认返回的是hash & key 两个信息， 但是还有其他很多信息可以通过代码设置 returnBody 进行返回获取； 
在下面的代码中关于七牛提供的 魔法变量 基本都返回了， 如果没有的即为null，具体可以查看七牛的 [魔法变量] (https://developer.qiniu.com/kodo/manual/1235/vars#magicvar) 说明文档；

**代码如下：**

```
    /**
     * 返回七牛 提供的魔法变量value， 给业务提供部分需要的信息
     * @param bucket
     * @param target
     */
    public static void testReturnBody(String bucket, File target) {
        //1.构建Auth 对象
        Auth auth = Auth.create(ACCESS_KEY, SECRET_KEY);
        //2.构建Configuration
        Configuration cfg = new Configuration(Zone.zone0());
        //3. 构建上传管理对象
        UploadManager uploadMgr = new UploadManager(cfg);
        StringMap policy = new StringMap();
        policy.putNotEmpty("returnBody", 
                "{\"key\":\"$(key)\","
                + "\"hash\":\"$(etag)\","
                + "\"bucket\":\"$(bucket)\","
                + "\"fsize\":\"$(fsize)\","
                + "\"hash\":\"$(etag)\","
                + "\"fname\":\"$(fname)\","
                + "\"mimeType\":\"$(mimeType)\","
                + "\"endUser\":\"$(endUser)\","
                + "\"exif\":\"$(exif)\","
                + "\"imageAve\":\"$(imageAve)\","
                + "\"ext\":\"$(ext)\","
                + "\"uuid\":\"$(uuid)\","
                + "\"width\":\"$(imageInfo.width)\","
                + "\"height\":\"$(imageInfo.height)\","
                + "\"year\":\"$(year)\","          //文档写明： 年月日时分秒 不支持在 returnBody and callbackBody 中使用， 此处为展示效果，结果都为null
                + "\"mon\":\"$(mon)\","
                + "\"day\":\"$(day)\","
                + "\"hour\":\"$(hour)\","
                + "\"min\":\"$(min)\","
                + "\"sec\":\"$(sec)\","
                + "}");
        //4. 获取上传token ，并指定上传策略 
        String token = auth.uploadToken(bucket, "20170721/" + target.getName(), 3600, policy);
        try {
            //5. 执行上传操作
            Response resp = uploadMgr.put(target, "20170721/" +target.getName(), token);
            //6. 查看魔法变量返回结果， 可以用Gson包 进行 json 解析， 转换成一个 上传策略中的ReturnBody 对象
            System.out.println(resp.bodyString());
        } catch (QiniuException e) {
            e.printStackTrace();
        }
    }
```

### 回调和自定义变量示例：

在实际业务中，可能需要上传后，通知业务服务器，做一些具体的实际业务处理， 即：回调； 七牛对业务服务器的回调可以设置上传策略中的callbackUrl 、callbackBody、 callBackHost 、callbackBodyType 字段； 
除了回调和七牛默认的魔法变量， 七牛还支持 “自定义变量”， 可以通过直接返回或者回调的方式为业务处理进行获取， 关于其说明，可以查看 [自定义变量] (https://developer.qiniu.com/kodo/manual/1235/vars#xvar) 的说明文档。

**代码如下：**

```

    /**
     * 测试七牛 自定义变量， 也是通过上传策略中的returnBody 来设置
     * @param bucket
     * @param target
     */
    public static void testUserDefinedVar(String bucket, File target) {
        //1.构建Auth 对象
        Auth auth = Auth.create(ACCESS_KEY, SECRET_KEY);

        //2. 通过OKHttp 构建一个表单上传
        OkHttpClient client = new OkHttpClient();
        MediaType mediaType = MediaType.parse("image/jpg");
        RequestBody fileBody = RequestBody.create(mediaType, target);

        //3. 构建上传策略， 指定自定义变量 currTime and content 
        StringMap putPolicy = new StringMap();
        //设置 callbackUrl and callbackBody
//      putPolicy.putNotNull("callbackBody", "{\"currTime\":\"$(x:currTime)\", \"content\":\"$(x:content)\", \"key\":\"$(key)\"}")
//               .putNotNull("callbackUrl", "http://22e7845a.ngrok.io/WebProject/callback");
        //设置 returnBody 返回自定义变量的结果
        putPolicy.putNotEmpty("returnBody", "{\"currTime\":\"$(x:currTime)\", \"content\":\"$(x:content)\", \"key\":\"$(key)\"}");

        //4. 获取token， 包含了上传策略（putPolicy）中的returnBody
        String token = auth.uploadToken(bucket, "20170721/"+target.getName(), 3600, putPolicy);
        RequestBody reqBody = new MultipartBody.Builder()
                                .setType(MultipartBody.FORM)
                                .addFormDataPart("file", target.getName(), fileBody)
                                .addFormDataPart("key", "20170721/"+target.getName())
                                .addFormDataPart("x:currTime", new SimpleDateFormat("yyyy/MM/dd:HH:mm:ss").format(new Date()))
                                .addFormDataPart("x:content", new String("Test User-Defined var by ReturnBody"))
                                .addFormDataPart("token", token)
                                .build();
        Request req = new Request.Builder()
                            .url("http://upload.qiniu.com/")
                            .post(reqBody)
                            .build();
        try {
            okhttp3.Response resp = client.newCall(req).execute();
            System.out.println(new String(resp.body().bytes()));
            System.out.println(resp.code() + ":" + resp.message());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

### 限制文件的上传类型和大小
在一些web 项目中经常会有上传图片做头像或者证明，根据实际业务需求，多数都是只能上传jpg的静态图片， 不能上传gif 这样的动态图片，还有文件大小设置， 七牛在上传策略中都提供了相关设置的字段。

**代码示例：**

```
    /**
     * 限制上传的文件类型和大小
     * @param bucketName
     * @param target
     */
    public static void qualifyFileType(String bucketName, File target) {
        Auth auth = Auth.create(ACCESS_KEY, SECRET_KEY);
        Configuration cfg = new Configuration(Zone.zone0());
        UploadManager uploadMgr = new UploadManager(cfg);
        StringMap putPolicy = new StringMap();
        putPolicy.put("fsizeMin", 1024)                     //文件大小最小为 1KB
                 .put("fsizeLimit", 1024 * 1024 * 10)       //文件大小最大为 10M
                 .putNotEmpty("mimeType", "image/jpg");     //文件类型为jpg图片, 所有图片 设置为 image/*
        //获取upload token， 在token中设置上传策略中的 fsizeMin, fsizeLimit, mimeType
        String token = auth.uploadToken(bucketName, "qualify11/" + target.getName(), 3600, putPolicy);
        try {
            Response resp = uploadMgr.put(target, "qualify11/" + target.getName(), token);
            System.out.println(resp.statusCode + ":" +resp.bodyString());
        } catch (QiniuException e) {
            System.out.println(e.code() + ":" + e.getMessage());
            e.printStackTrace();
        }
    }
```

## 总结

1. 通过returnBody 将默认的返回值进行调整，获取更多的信息 
2. 实现回调和自定义变量的获取， 方便实际业务的数据处理 
3. 设置文件上传的类型和大小，满足业务需求


