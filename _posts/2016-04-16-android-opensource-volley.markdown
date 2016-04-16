---
layout:     post
title:      "Volley"
subtitle:   "Volley学习总结"
date:       2016-04-16
author:     "lwenkun"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - OS
    - Unix
    - Linux
---


> 简介

Volley 是谷歌提供的 android 网络通信框架。在安卓中，谷歌提供了两种网络通信有关的类 `HttpClient` 和 `HttpUrlConnection`，但是如果不适当封装，很容写很多重复的代码。谷歌也意识到了这个问题，所以提供了 Volley 来解决这个问题。

> 用法

### 1.StringRequest的用法
<pre>
RequestQueue mQueue = Volley.newRequestQueue(context);
StringRequest stringReuqest = new StirngRequest("http://www.baidu.com", new Response.Listener&lt;String&gt;() {
   @Override
   public void onResponse(String response) {
      Log.d("Tag", response);
   }, new Response.ErrorListener&lt;String&gt;(){
         @Override
         public void onErrorResponse(String response) {
            Log.d("Tag", response);
         });
mQueue.add(stringRequest);
</pre>

当然，Volley 需要用到网络，故应记得添加网络请求权限。

运行之后，在 logcat 中我们就可以看到一长串的 html 代码了。这就是百度返回给我们的结果。

默认的是发送 `get` 请求，但是我们如果要发送其他的请求，我们就要用到`StringRequest`类的构造方法的重载，如我们要发送 `post` 请求：
<pre>
StringRequest stringRequest = new StringRequest(Method.POST, url, listener, errorListener);
</pre>
`post` 请求我们知道参数不能直接写在url中，那我们应该怎样发送参数呢？Easy! 看下面的代码：
<pre>
StringRequest stringRequest = new StringRequest(Method.POST, url,  listener, errorListener) {  
    @Override  
    protected Map&lt;String, String&gt; getParams() throws AuthFailureError {  
        Map&lt;String, String&gt; map = new HashMap&lt;String, String&gt;();  
        map.put("params1", "value1");  
        map.put("params2", "value2");  
        return map;  
    }  
};
</pre>
我们重写了 `StringRequest` 的方法中 `getParams` 方法，在这里我们构造参数表。

### 2.JsonObjectRequest&amp;JsonArrayRequest的用法
```java
JsonObjectRequest jsonObjectRequest = new
JsonObjectRequest("http://m.weather.com.cn/data/101010100.html", null, new Response.Listener&lt;JsonObject&gt;() {
   @Override
   public void onResponse(JsonObject response) {
      Log.d("tag", response.toString());
   }, new Response.ErrorListener(JsonObject response) {
         @Override
         public void onErrorResponse() {
            log.d("tag", response.toString();
         }
      });
```
然后将这个加入请求队列中:
<pre>
mRequest.add(jsonObjectRequest);
</pre>
`JsonArrrayRequest` 的用法类似。

### 3.ImageRequest的用法
<pre>
ImageRequest imageRequest = new ImageRequest("http://developer.android.com/images/home/aw_dac.png", new Response.Listener&lt;Bitmap&gt;() {
   @Override
   public void onResponse&lt;Bitmap bitmap) {
      imageView.setImageBitmap(bitmap);
   }
}, 0, 0, new Response.ErrorListener() {
   @Override
   public void onErrorResponse(VolleyError error) {
      imageView.setImageResource(R.drawable.default_image);  
   }
});
</pre>
未完待续。。。