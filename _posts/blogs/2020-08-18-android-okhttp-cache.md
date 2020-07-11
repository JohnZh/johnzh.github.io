---
layout: post_no_cmt
title:  "HTTP 缓存策略（okHttp 的实现）"
date:   2020-08-18 01:00:00 +0800
categories: [android]
---

# 总流程
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200818165112924.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70#pic_center)
# 代码
```
// CacheInterceptor#intercept
CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
Request networkRequest = strategy.networkRequest;
Response cacheResponse = strategy.cacheResponse;

// CacheStrategy#Factory
public Factory(long nowMillis, Request request, Response cacheResponse) {
   this.nowMillis = nowMillis;
   this.request = request;
   this.cacheResponse = cacheResponse;

   if (cacheResponse != null) {
     this.sentRequestMillis = cacheResponse.sentRequestAtMillis();
     this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis();
     Headers headers = cacheResponse.headers();
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
如果 cacheResponse 不为 null，即为步骤 1 的 YES，then

```
// CacheStrategy.Factory#get

public CacheStrategy get() {
  CacheStrategy candidate = getCandidate();

  if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
    // We're forbidden from using the network and the cache is insufficient.
    return new CacheStrategy(null, null);
  }

  return candidate;
}

// CacheStrategy.Factory#getCandidate

private CacheStrategy getCandidate() {
  // No cached response.
  if (cacheResponse == null) {
  // 步骤 1 NO 的情况
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
  // 步骤 2 的 NO 的情况
    return new CacheStrategy(request, null);
  }

  // 这也是步骤 2 的 NO 的情况，但是是根据当前 request 的 headers 里面的 'Cache-Control:no-cache' 
  // 此外，或者的条件是 headers 里面 'If-Modified-Since' 或者 'If-None-Match' 的存在
  // 服务端处理优先级上 'If-None-Match' > 'If-Modified-Since'
  // 'If-Modified-Since'，检查自一个时间后数据是否有改变
  // 'If-None-Match'，配合 etag，检查一个版本是否匹配
  // 目的是忽略了本地新鲜度的判断直接进行服务端的新鲜度验证，属于步骤 3 NO 的情况
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

// 到这，进行步骤 3 的 NO，和之前 request 的 headers 使用的字段一样，使用 'If-None-Match'，'If-Modified-Since' 从服务端检测缓存的新鲜度

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

回到 `CacheInterceptor#intercept`

```
Request networkRequest = strategy.networkRequest;
Response cacheResponse = strategy.cacheResponse;

// networkRequest 的存在就说明还要向服务端请求
// 这个请求可能是因为缓存不存在，不需要缓存，或者是新鲜度检查

... 省略了 504 的代码

// If we don't need the network, we're done.
if (networkRequest == null) {
  return cacheResponse.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .build();
}
// 不需要网络请求，直接用缓存，对应着步骤 3 的 YES

Response networkResponse = null;
try {
  networkResponse = chain.proceed(networkRequest);
} finally {
  // If we're crashing on I/O or otherwise, don't leak the cache body.
  if (networkResponse == null && cacheCandidate != null) {
    closeQuietly(cacheCandidate.body());
  }
}
// 继续通过责任链的下一个拦截器去进行网络请求，对应步骤 1/2/3 的 NO

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
    
// 对应步骤 4 的 NO 和步骤 6
if (cache != null) {
  if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
  // 对应步骤 7 的 YES 存入缓存，以及步骤 8
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

// 到了这里，对应的就是步骤 7 的 NO
return response;
```

