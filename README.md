# NetVolley
# Volley 的主要特点
* (1). 扩展性强。Volley 中大多是基于接口的设计，可配置性强。
* (2). 一定程度符合 Http 规范，包括返回 ResponseCode(2xx、3xx、4xx、5xx）的处理，请求头的处理，缓存机制的支持等。并支持重试及优先级定义。
* (3). 默认 Android2.3 及以上基于 HttpURLConnection，2.3 以下基于 HttpClient 实现，这两者的区别及优劣在4.2.1 Volley中具体介绍。
* (4). 提供简便的图片加载工具。
## 2.2 Volley 中的概念
* Volley：Volley 对外暴露的 API，通过 newRequestQueue(…) 函数新建并启动一个请求队列RequestQueue。
* Request：表示一个请求的抽象类。StringRequest、JsonRequest、ImageRequest 都是它的子类，表示某种类型的请求。
* RequestQueue：表示请求队列，里面包含一个CacheDispatcher(用于处理走缓存请求的调度线程)、NetworkDispatcher数组(用于处理走网络请求的调度线程)，一个ResponseDelivery(返回结果分发接口)，通过 start() 函数启动时会启动CacheDispatcher和NetworkDispatchers。
* CacheDispatcher：一个线程，用于调度处理走缓存的请求。启动后会不断从缓存请求队列中取请求处理，队列为空则等待，请求处理结束则将结果传递给ResponseDelivery去执行后续处理。当结果未缓存过、缓存失效或缓存需要刷新的情况下，该请求都需要重新进入NetworkDispatcher去调度处理。
* NetworkDispatcher：一个线程，用于调度处理走网络的请求。启动后会不断从网络请求队列中取请求处理，队列为空则等待，请求处理结束则将结果传递给ResponseDelivery去执行后续处理，并判断结果是否要进行缓存。
* ResponseDelivery：返回结果分发接口，目前只有基于ExecutorDelivery的在入参 handler 对应线程内进行分发。
* HttpStack：处理 Http 请求，返回请求结果。目前 Volley 中有基于 HttpURLConnection 的HurlStack和 基于 Apache HttpClient 的HttpClientStack。
* Network：调用HttpStack处理请求，并将结果转换为可被ResponseDelivery处理的NetworkResponse。
* Cache：缓存请求结果，Volley 默认使用的是基于 sdcard 的DiskBasedCache。NetworkDispatcher得到请求结果后判断是否需要存储在 Cache，CacheDispatcher会从 Cache 中取缓存结果。
## 4.2 核心类功能介绍
### 4.2.1 Volley.java
* 这个和 Volley 框架同名的类，其实是个工具类，作用是构建一个可用于添加网络请求的RequestQueue对象。
* (1). 主要函数
* Volley.java 有两个重载的静态方法。
* public static RequestQueue newRequestQueue(Context context)

* public static RequestQueue newRequestQueue(Context context, HttpStack stack)
* 第一个方法的实现调用了第二个方法，传 HttpStack 参数为 null。
* 第二个方法中，如果 HttpStatck 参数为 null，则如果系统在 Gingerbread 及之后(即 API Level >= 9)，采用基于 HttpURLConnection 的 HurlStack，如果小于 9，采用基于 HttpClient 的 HttpClientStack。
* if (stack == null) {
  *   if (Build.VERSION.SDK_INT >= 9) {
  *       stack = new HurlStack();
  *   } else {
  *       stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
  *  }
* }
* 得到了 HttpStack,然后通过它构造一个代表网络（Network）的具体实现BasicNetwork。
* 接着构造一个代表缓存（Cache）的基于 Disk 的具体实现DiskBasedCache。
* 最后将网络（Network）对象和缓存（Cache）对象传入构建一个 RequestQueue，启动这个 RequestQueue，并返回。
* Network network = new BasicNetwork(stack);
* RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
* queue.start();
* return queue;
* 我们平时大多采用Volly.newRequestQueue(context)的默认实现，构建 RequestQueue。
* 通过源码可以看出，我们可以抛开 Volley 工具类构建自定义的 RequestQueue，采用自定义的HttpStatck，采用自定义的Network实现，采用自定义的 Cache 实现等来构建RequestQueue。
* (2). HttpURLConnection 和 AndroidHttpClient(HttpClient 的封装)如何选择及原因：
* 在 Froyo(2.2) 之前，HttpURLConnection 有个重大 Bug，调用 close() 函数会影响连接池，导致连接复用失效，所以在 Froyo 之前使用 HttpURLConnection 需要关闭 keepAlive。
另外在 Gingerbread(2.3) HttpURLConnection 默认开启了 gzip 压缩，提高了 HTTPS 的性能，Ice Cream Sandwich(4.0) HttpURLConnection 支持了请求结果缓存。
再加上 HttpURLConnection 本身 API 相对简单，所以对 Android 来说，在 2.3 之后建议使用 HttpURLConnection，之前建议使用 AndroidHttpClient。
* (3). 关于 User Agent
* 通过代码我们发现如果是使用 AndroidHttpClient，Volley 还会将请求头中的 User-Agent 字段设置为 App 的 ${packageName}/${versionCode}，
* 如果异常则使用 "volley/0"，不过这个获取 User-Agent 的操作应该放到 if else 内部更合适。而对于 HttpURLConnection 却没有任何操作，为什么呢？
* 如果用 Fiddler 或 Charles 对数据抓包我们会发现，我们会发现 HttpURLConnection 默认是有 User-Agent 的，类似：
* Dalvik/1.6.0 (Linux; U; Android 4.1.1; Google Nexus 4 - 4.1.1 - API 16 - 768x1280_1 Build/JRO03S)
* 经常用 WebView 的同学会也许会发现似曾相识，是的，WebView 默认的 User-Agent 也是这个。实际在请求发出之前，会检测 User-Agent 是否为空，如果不为空，则加上系统默认 User-Agent。在 Android 2.1 之后，我们可以通过
* String userAgent = System.getProperty("http.agent");
* 得到系统默认的 User-Agent，Volley 如果希望自定义 User-Agent，可在自定义 Request 中重写 getHeaders() 函数
* @Override
* public Map<String, String> getHeaders() throws AuthFailureError {
  *   // self-defined user agent
   *  Map<String, String> headerMap = new HashMap<String, String>();
   *  headerMap.put("User-Agent", "android-open-project-analysis/1.0");
   *  return headerMap;
* }
### 4.2.2 Request.java
* 代表一个网络请求的抽象类。我们通过构建一个Request类的非抽象子类(StringRequest、JsonRequest、ImageRequest 或自定义)对象，并将其加入到·RequestQueue·中来完成一次网络请求操作。
* Volley 支持 8 种 Http 请求方式 GET, POST, PUT, DELETE, HEAD, OPTIONS, TRACE, PATCH
* Request 类中包含了请求 url，请求请求方式，请求 Header，请求 Body，请求的优先级等信息。
* 因为是抽象类，子类必须重写的两个方法。
* abstract protected Response<T> parseNetworkResponse(NetworkResponse response);
* 子类重写此方法，将网络返回的原生字节内容，转换成合适的类型。此方法会在工作线程中被调用。
* abstract protected void deliverResponse(T response);
* 子类重写此方法，将解析成合适类型的内容传递给它们的监听回调。
* 以下两个方法也经常会被重写
* public byte[] getBody()
* 重写此方法，可以构建用于 POST、PUT、PATCH 请求方式的 Body 内容。
* protected Map<String, String> getParams()
* 在上面getBody函数没有被重写情况下，此方法的返回值会被 key、value 分别编码后拼装起来转换为字节码作为 Body 内容。




