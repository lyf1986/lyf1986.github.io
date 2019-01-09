---
layout: post
title:  "OkHttp3用法全解析"
date:   2018-01-21 22:14:54
categories: java
excerpt: OkHttp3用法全解析
---

# OkHttp3用法全解析

* content
{:toc}

#### 1.使用前准备
Android Studio 配置gradle：
```
compile 'com.squareup.okhttp3:okhttp:3.2.0'
compile 'com.squareup.okio:okio:1.7.0'
```

添加网络权限：
```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

#### 2.异步GET请求
惯例，请求百度：
```java
    private void getAsynHttp() {
    mOkHttpClient=new OkHttpClient();
    Request.Builder requestBuilder = new Request.Builder().url("http://www.baidu.com");
    //可以省略，默认是GET请求
    requestBuilder.method("GET",null);
    Request request = requestBuilder.build();
    Call mcall= mOkHttpClient.newCall(request);
    mcall.enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {
        }
        @Override
        public void onResponse(Call call, Response response) throws IOException {
            if (null != response.cacheResponse()) {
                String str = response.cacheResponse().toString();
                Log.i("wangshu", "cache---" + str);
            } else {
                response.body().string();
                String str = response.networkResponse().toString();
                Log.i("wangshu", "network---" + str);
            }
            runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    Toast.makeText(getApplicationContext(), "请求成功", Toast.LENGTH_SHORT).show();
                }
            });
        }
    });
}
```

与2.x版本并没有什么不同，比较郁闷的是回调仍然不在UI线程。

#### 2.异步POST请求

OkHttp3异步POST请求和OkHttp2.x有一些差别就是没有FormEncodingBuilder这个类，替代它的是功能更加强大的FormBody：
```java
private void postAsynHttp() {
     mOkHttpClient=new OkHttpClient();
     RequestBody formBody = new FormBody.Builder()
             .add("size", "10")
             .build();
     Request request = new Request.Builder()
             .url("http://api.1-blog.com/biz/bizserver/article/list.do")
             .post(formBody)
             .build();
     Call call = mOkHttpClient.newCall(request);
     call.enqueue(new Callback() {
         @Override
         public void onFailure(Call call, IOException e) {
         }
         @Override
         public void onResponse(Call call, Response response) throws IOException {
             String str = response.body().string();
             Log.i("wangshu", str);
             runOnUiThread(new Runnable() {
                 @Override
                 public void run() {
                     Toast.makeText(getApplicationContext(), "请求成功", Toast.LENGTH_SHORT).show();
                 }
             });
         }
     });
 }
 ```

#### 3.异步上传文件

上传文件本身也是一个POST请求，上一篇没有讲，这里我们补上。首先定义上传文件类型：

```java
public static final MediaType MEDIA_TYPE_MARKDOWN=MediaType.parse("text/x-markdown; charset=utf-8");
```

将sdcard根目录的wangshu.txt文件上传到服务器上：

```java
private void postAsynFile() {
    mOkHttpClient=new OkHttpClient();
    File file = new File("/sdcard/wangshu.txt");
    Request request = new Request.Builder()
            .url("https://api.github.com/markdown/raw")
            .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, file))
            .build();
        mOkHttpClient.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
            }
            @Override
            public void onResponse(Call call, Response response) throws IOException {
                Log.i("wangshu",response.body().string());
            }
        });
}
```

当然如果想要改为同步的上传文件只要调用 mOkHttpClient.newCall(request).execute()就可以了。
在wangshu.txt文件中有一行字“Android网络编程（六）OkHttp3用法全解析”我们运行程序点击发送文件按钮，最终请求网络返回的结果就是我们txt文件中的内容 ：


当然不要忘了添加如下权限：

```xml
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```

4.异步下载文件
下载文件同样在上一篇没有讲到，实现起来比较简单，在这里下载一张图片，我们得到Response后将流写进我们指定的图片文件中就可以了。
```java
 private void downAsynFile() {
     mOkHttpClient = new OkHttpClient();
     String url = "http://img.my.csdn.net/uploads/201603/26/1458988468_5804.jpg";
     Request request = new Request.Builder().url(url).build();
     mOkHttpClient.newCall(request).enqueue(new Callback() {
         @Override
         public void onFailure(Call call, IOException e) {
         }
         @Override
         public void onResponse(Call call, Response response) {
             InputStream inputStream = response.body().byteStream();
             FileOutputStream fileOutputStream = null;
             try {
                 fileOutputStream = new FileOutputStream(new File("/sdcard/wangshu.jpg"));
                 byte[] buffer = new byte[2048];
                 int len = 0;
                 while ((len = inputStream.read(buffer)) != -1) {
                     fileOutputStream.write(buffer, 0, len);
                 }
                 fileOutputStream.flush();
             } catch (IOException e) {
                 Log.i("wangshu", "IOException");
                 e.printStackTrace();
            }
            Log.d("wangshu", "文件下载成功");
        }
    });
}
```

#### 5.异步上传Multipart文件

这种场景很常用，我们有时会上传文件同时还需要传其他类型的字段，OkHttp3实现起来很简单，需要注意的是没有服务器接收我这个Multipart文件，所以这里只是举个例子，具体的应用还要结合实际工作中对应的服务器。
首先定义上传文件类型：

```java
private static final MediaType MEDIA_TYPE_PNG = MediaType.parse("image/png");
private void sendMultipart(){
    mOkHttpClient = new OkHttpClient();
    RequestBody requestBody = new MultipartBody.Builder()
            .setType(MultipartBody.FORM)
            .addFormDataPart("title", "wangshu")
            .addFormDataPart("image", "wangshu.jpg",
                    RequestBody.create(MEDIA_TYPE_PNG, new File("/sdcard/wangshu.jpg")))
            .build();
    Request request = new Request.Builder()
            .header("Authorization", "Client-ID " + "...")
            .url("https://api.imgur.com/3/image")
            .post(requestBody)
            .build();
   mOkHttpClient.newCall(request).enqueue(new Callback() {
       @Override
       public void onFailure(Call call, IOException e) {
       }
       @Override
       public void onResponse(Call call, Response response) throws IOException {
           Log.i("wangshu", response.body().string());
       }
   });
}
```

#### 6.设置超时时间和缓存

和OkHttp2.x有区别的是不能通过OkHttpClient直接设置超时时间和缓存了，而是通过OkHttpClient.Builder来设置，通过builder配置好OkHttpClient后用builder.build()来返回OkHttpClient，所以我们通常不会调用new OkHttpClient()来得到OkHttpClient，而是通过builder.build()：

```java
File sdcache = getExternalCacheDir();
int cacheSize = 10 * 1024 * 1024;
OkHttpClient.Builder builder = new OkHttpClient.Builder()
        .connectTimeout(15, TimeUnit.SECONDS)
        .writeTimeout(20, TimeUnit.SECONDS)
        .readTimeout(20, TimeUnit.SECONDS)
        .cache(new Cache(sdcache.getAbsoluteFile(), cacheSize));
OkHttpClient mOkHttpClient=builder.build();
```

#### 7.关于取消请求和封装

取消请求仍旧可以调用call.cancel()，这个没有变化，不明白的可以查看上一篇文章Android网络编程（五）OkHttp2.x用法全解析，这里就不赘述了，封装上一篇也讲过仍旧推荐OkHttpFinal，它目前是基于OkHttp3来进行封装的。
