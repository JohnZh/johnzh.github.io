---
layout: post_no_cmt
title:  "HTTP 缓存策略（okHttp 的实现）"
date:   2020-08-18 01:00:00 +0800
categories: [android]
---

# 总流程

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818165112924.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70#pic_center)

# 代码分析

```java
// CacheInterceptor#intercept
Response cacheCandidate = cache != null
  ? cache.get(chain.request())
  : null;

long now = System.currentTimeMillis();

CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();

Request networkRequest = strategy.networkRequest;
Response cacheResponse = strategy.cacheResponse;
```

```java
// CacheStrategy#Factory
public Factory(long nowMillis, Request request, Response cacheResponse) {
   this.nowMillis = nowMillis;
   this.request = request;
   this.cacheResponse = cacheResponse;

  // cacheResponse == null 对应步骤 1 的 NO
   if (cacheResponse != null) {
     this.sentRequestMillis = cacheResponse.sentRequestAtMillis();
     this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis();
     Headers headers = cacheResponse.headers();
     // 从缓存的响应里解析出 header 里面和缓存相关的信息
     for (int i = 0, size = headers.size(); i < size; i++) {
       String fieldName = headers.name(i);
       String value = headers.value(i);
       if ("Date".equalsIgnoreCase(fieldName)) {
       // 服务端响应的时间 eg. Date: Tue, 15 Nov 2019 10:12:31 GMT
         servedDate = HttpDate.parse(value);
         servedDateString = value;
       } else if ("Expires".equalsIgnoreCase(fieldName)) {
       // 缓存过期时间，超过该时间认定为过期 eg. Expires: Tue, 15 Nov 2019 10:12:31 GMT
         expires = HttpDate.parse(value);
       } else if ("Last-Modified".equalsIgnoreCase(fieldName)) {
       // 缓存响应最后修改时间。格式同上
         lastModified = HttpDate.parse(value);
         lastModifiedString = value;
       } else if ("ETag".equalsIgnoreCase(fieldName)) {
       // 缓存响应的特定版本标识符.eg. ETag: "737060cd8c284d8af7ad3082f209582d"
         etag = value;
       } else if ("Age".equalsIgnoreCase(fieldName)) {
       // 缓存响应在缓存中存在的时间，以秒为单位
         ageSeconds = HttpHeaders.parseSeconds(value, -1);
       }
     }
   }
 }
```

如果 cacheResponse 是 null，也不用进行什么判断了，直接调用下一个拦截器去请求；如果 cacheResponse 不为 null，即为步骤 1 的 YES，把缓存响应 header 里面和缓存相关的东西先解析出来，后面的判断会用到，然后继续执行 CacheStrategy$Factory.get

```java
// CacheStrategy.Factory#get
public CacheStrategy get() {
  CacheStrategy candidate = getCandidate();

  if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
    // We're forbidden from using the network and the cache is insufficient.
    return new CacheStrategy(null, null);
  }

  return candidate;
}
```

```java
// CacheStrategy.Factory#getCandidate
private CacheStrategy getCandidate() {
  // No cached response.
  if (cacheResponse == null) {
  // 对应的还是步骤 1 NO 的情况
    return new CacheStrategy(request, null);
  }

  // Drop the cached response if it's missing a required handshake.
  if (request.isHttps() && cacheResponse.handshake() == null) {
    return new CacheStrategy(request, null);
  }

  // If this response shouldn't have been stored, it should never be used
  // as a response source. This check should be redundant as long as the
  // persistence store is well-behaved and the rules are constant.
  if (!isCacheable(cacheResponse, request)) {
    // 步骤 2 的 NO 的情况：isCacheable 可缓存的。可缓存（返回 true）的情况：
    // 响应的这些响应码允许缓存（200，203，204，300，301，404，405，410，414，501，308）
    // 以及响应码是 302，307，且响应头部 Expires 有值|有 maxAge|Cache-Control:public|private
    // 关于哪些响应可以被缓存的参考 http://tools.ietf.org/html/rfc7234#section-3
    // 响应的头部没有 Cache-Control:no-store
    // 请求的头部没有 Cache-control: no-store
    // 如果以上情况都不满足，那也可以去发网络请求了
    
    // 注意：isCacheable 方法是发送前后通用的
    // 发送请求响应回来后也会调用这个方法判断是否要进行缓存
    // 因此，这个地方调用主要是来判断请求头部的有没有 Cache-control: no-store
    
    return new CacheStrategy(request, null);
  }

  // 根据是当前请求的头部里面有 Cache-Control:no-cache，这也是步骤 2 的 NO 的情况
  // 此外，还有步骤 3 的 NO 的情况，即
  // 头部里面有 If-Modified-Since 或者 If-None-Match 的存在
  // 服务端处理优先级上 If-None-Match > If-Modified-Since
  // If-Modified-Since：检查自某个时间后数据是否有改变
  // If-None-Match：配合 etag，检查某个版本是否匹配
  // 目的是忽略了本地新鲜度的判断直接进行服务端的新鲜度验证
  CacheControl requestCaching = request.cacheControl();
  if (requestCaching.noCache() || hasConditions(request)) {
    return new CacheStrategy(request, null);
  }

  CacheControl responseCaching = cacheResponse.cacheControl();

  long ageMillis = cacheResponseAge();
  long freshMillis = computeFreshnessLifetime();

  if (requestCaching.maxAgeSeconds() != -1) {
    freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
  }

  long minFreshMillis = 0;
  if (requestCaching.minFreshSeconds() != -1) {
    minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
  }

  long maxStaleMillis = 0;
  if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
    maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
  }

  // 可缓存，当前新鲜度还在范围内，未超过腐朽时间，步骤 3 的 YES
  if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
    Response.Builder builder = cacheResponse.newBuilder();
    if (ageMillis + minFreshMillis >= freshMillis) {
      builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
    }
    long oneDayMillis = 24 * 60 * 60 * 1000L;
    if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
      builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
    }
    return new CacheStrategy(null, builder.build());
  }

  // 到这，说明新鲜度不行了，于是进入步骤 3 的 NO
  // 使用 If-None-Match，If-Modified-Since 从服务端检测缓存的新鲜度
  // 注意：这些新的头部是追加的

  // Find a condition to add to the request. If the condition is satisfied, the response body
  // will not be transmitted.
  String conditionName;
  String conditionValue;
  if (etag != null) {
    conditionName = "If-None-Match";
    conditionValue = etag;
  } else if (lastModified != null) {
    conditionName = "If-Modified-Since";
    conditionValue = lastModifiedString;
  } else if (servedDate != null) {
    conditionName = "If-Modified-Since";
    conditionValue = servedDateString;
  } else {
    return new CacheStrategy(request, null); // No condition! Make a regular request.
  }

  Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
  Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);

  Request conditionalRequest = request.newBuilder()
      .headers(conditionalRequestHeaders.build())
      .build();
  return new CacheStrategy(conditionalRequest, cacheResponse);
}
```

回到 get 方法：对于 Cache-Control: only-If-Cached 这个头的处理，这个地方其实我看的挺不懂的，按 http 协议的定义这个头部理论上就是直接不使用网络，直接使用缓存，但是这里缓存策略的处理居然是把缓请求和缓存响应都 null 了，然后后面回到 CacheInterceptor.intercept，就会被处理成 504 Gateway Timeout。真的无法理解

```java
if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
  // We're forbidden from using the network and the cache is insufficient.
  return new CacheStrategy(null, null);
}

return candidate;
```

回到 `CacheInterceptor#intercept`

```java
Request networkRequest = strategy.networkRequest;
Response cacheResponse = strategy.cacheResponse;
// networkRequest 的存在就说明还要向服务端请求
// 这个请求可能是因为缓存不存在，不需要缓存，或者是为了去检查缓存新鲜度

// 这既是之前说的 Cache-Control: only-If-Cached 导致的 504
// If we're forbidden from using the network and the cache is insufficient, fail.
if (networkRequest == null && cacheResponse == null) {
  return new Response.Builder()
    .request(chain.request())
    .protocol(Protocol.HTTP_1_1)
    .code(504)
    .message("Unsatisfiable Request (only-if-cached)")
    .body(Util.EMPTY_RESPONSE)
    .sentRequestAtMillis(-1L)
    .receivedResponseAtMillis(System.currentTimeMillis())
    .build();
}

// 不需要网络请求，直接用缓存，对应着步骤 3 的 YES
// If we don't need the network, we're done.
if (networkRequest == null) {
  return cacheResponse.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .build();
}


// 继续通过责任链的下一个拦截器去进行网络请求，对应步骤 1/2/3 的 NO
Response networkResponse = null;
try {
  networkResponse = chain.proceed(networkRequest);
} finally {
  // If we're crashing on I/O or otherwise, don't leak the cache body.
  if (networkResponse == null && cacheCandidate != null) {
    closeQuietly(cacheCandidate.body());
  }
}

if (cacheResponse != null) {
// 302 对应步骤 3 的 NO 到步骤 4 的 YES，服务数据未修改，更新缓存新鲜度相关信息
  if (networkResponse.code() == HTTP_NOT_MODIFIED) {
    Response response = cacheResponse.newBuilder()
        .headers(combine(cacheResponse.headers(), networkResponse.headers()))
        .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
        .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();
    networkResponse.body().close();

    // Update the cache after combining headers but before stripping the
    // Content-Encoding header (as performed by initContentStream()).
    cache.trackConditionalCacheHit();
    cache.update(cacheResponse, response);
    return response;
  } else {
    closeQuietly(cacheResponse.body());
  }
}


Response response = networkResponse.newBuilder()
    .cacheResponse(stripBody(cacheResponse))
    .networkResponse(stripBody(networkResponse))
    .build();
    
// 对应步骤 4 的 NO 缓存不新鲜、步骤 6 重新获取数据
if (cache != null) {
  if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
  // 对应步骤 7 的 YES，数据可以缓存，以及步骤 8，存入缓存
    // Offer this request to the cache.
    CacheRequest cacheRequest = cache.put(response);
    return cacheWritingResponse(cacheRequest, response);
  }

// 对于非 GET 的请求，删除缓存，这些是非法缓存
  if (HttpMethod.invalidatesCache(networkRequest.method())) {
    try {
      cache.remove(networkRequest);
    } catch (IOException ignored) {
      // The cache cannot be written.
    }
  }
}

// 到了这里，对应的就是步骤 7 的 NO，不可缓存的响应
return response;
```



# 总结

okhttp 总体实现就是安装 http 协议的规则来做的，具体就是对请求和响应的头部里面和缓存相关的字段做各种对应的处理。

- 先检查本地缓存有没有，没有的话不用继续了，网络请求吧
- 有缓存，那么从将发出的请求的头部判断是否允许缓存，不允许的话就网络请求吧
  - 这里主要是请求头部的 Cache-Control:no-store，no-cache
  - 注意：isCacheable 方法既用于判断即将发送的请求，又用于判断请求后返回的响应
- 请求头的设置是允许缓存的，那么检查缓存是否新鲜
  - 是否新鲜要选择是去服务端检查还是本地检查
  - 如果请求头里面有 If-Modified-Since 和 If-None-Match 就是要去服务器检查缓存新鲜度
  - 没有那两请求头，做本地检查，根据缓存响应头里面的 Date，Expires，Last-Modified，ETag，Age，还有本地发送请求和收到响应的时间来判断缓存是否新鲜
- **从代码实现上来，本地新鲜度的判断是唯一有可能不用发生网络请求的地方**
- 向服务器检查新鲜度，缓存没过期就会返回 304，然后更新缓存响应的头部以及这次请求的发生时间和响应收到时间；过期了收到正常响应数据，处理参考下方
- 本地检查新鲜度，过期了，发送网络请求，收到响应数据，用 isCacheable 方法判断是否要缓存，需要则缓存。最后不管需不需要缓存，响应数据都要返回给用户
- 根据 CacheInterceptor 的实现，请求方法为 POST/PATCH/PUT/DELETE/MOVE 的请求都是不需要有缓存的，都要删除缓存
