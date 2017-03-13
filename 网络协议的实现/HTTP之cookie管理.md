# cookie介绍

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1781505-bca68bda9b3993c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# cookie 规范

> 注: 主要参考[HTTP cookies 详解](http://blog.csdn.net/lijing198997/article/details/9378047),笔记做成思维导图

[RFC2965-HTTP State Management Mechanism](https://tools.ietf.org/html/rfc2965)

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1781505-a0a052abc1084221.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1781505-b63b06f5cde8d53e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# okhttp源码中的实现

## Cookie类的构造
> 根据构造可以看出,基本是按照cookie规范来的

```
private Cookie(String name, String value, long expiresAt, String domain, String path,
      boolean secure, boolean httpOnly, boolean hostOnly, boolean persistent) {
    this.name = name;
    this.value = value;
    this.expiresAt = expiresAt;//过期时间
    this.domain = domain;//域
    this.path = path;//子路径匹配
    this.secure = secure;//是否仅应用于https
    this.httpOnly = httpOnly;//是否禁止javascript操作cookie
    this.hostOnly = hostOnly;//是否针对特定主机
    this.persistent = persistent;//是否持久化
  }

  Cookie(Builder builder) {
    if (builder.name == null) throw new NullPointerException("builder.name == null");
    if (builder.value == null) throw new NullPointerException("builder.value == null");
    if (builder.domain == null) throw new NullPointerException("builder.domain == null");

    this.name = builder.name;
    this.value = builder.value;
    this.expiresAt = builder.expiresAt;
    this.domain = builder.domain;
    this.path = builder.path;
    this.secure = builder.secure;
    this.httpOnly = builder.httpOnly;
    this.persistent = builder.persistent;
    this.hostOnly = builder.hostOnly;
  }

```

## cookie交互流程的客户端实现
> 三,二,一,走

#### [BridgeInterceptor](https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/http/BridgeInterceptor.java)的intercept方法中
```
  List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
//给request设置cookie,List<Cookie> cookies由cookieJar.loadForRequest方法返回,该方式由开发者实现.
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
//从response中解析出cookies,字符串变成List<Cookie>,交给cookiejar接口处理

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);
```


### 跟随cookie流程,先看从response中解析的过程
> header中格式类似下面
> Set-Cookie:name1=value1;name2=value2 ;expires=date;domain=domain ;path=path ;secure;httpOnly 

1. 从response中拿到headers

2. 从headers中拿到"Set-Cookie"对应的内容(String)

3. 将拿到的String根据cookie字符串特点解析,并构建Cookie类

###  HttpHeaders.receiveHeaders
```
public static void receiveHeaders(CookieJar cookieJar, HttpUrl url, Headers headers) {
    if (cookieJar == CookieJar.NO_COOKIES) return;

    List<Cookie> cookies = Cookie.parseAll(url, headers);//关键的解析方法在这里面
    if (cookies.isEmpty()) return;

    cookieJar.saveFromResponse(url, cookies);//解析出的cookies交给cookieJar,让它去决定怎么保存
  }

```

###  Cookie.parseAll
```

public static List<Cookie> parseAll(HttpUrl url, Headers headers) {
    List<String> cookieStrings = headers.values("Set-Cookie");//可以解析headers中多个Set-Cookie字段
    List<Cookie> cookies = null;

    for (int i = 0, size = cookieStrings.size(); i < size; i++) {
      Cookie cookie = Cookie.parse(url, cookieStrings.get(i));//Set-Cookie:后方的cookie内容的核心解析方法:
      if (cookie == null) continue;
      if (cookies == null) cookies = new ArrayList<>();
      cookies.add(cookie);
    }

    return cookies != null
        ? Collections.unmodifiableList(cookies)
        : Collections.<Cookie>emptyList();
  }

```

### Cookie.parse: 核心方法
> 对照着cookie规范来看

```
static Cookie parse(long currentTimeMillis, HttpUrl url, String setCookie) {
    int pos = 0;
    int limit = setCookie.length();
    int cookiePairEnd = delimiterOffset(setCookie, pos, limit, ';');

    int pairEqualsSign = delimiterOffset(setCookie, pos, cookiePairEnd, '=');
    if (pairEqualsSign == cookiePairEnd) return null;

    String cookieName = trimSubstring(setCookie, pos, pairEqualsSign);//拿到name
    if (cookieName.isEmpty() || indexOfControlOrNonAscii(cookieName) != -1) return null;

    String cookieValue = trimSubstring(setCookie, pairEqualsSign + 1, cookiePairEnd);//拿到value
    if (indexOfControlOrNonAscii(cookieValue) != -1) return null;

    long expiresAt = HttpDate.MAX_DATE;//默认有效期很长
    long deltaSeconds = -1L;
    String domain = null;
    String path = null;
    boolean secureOnly = false;
    boolean httpOnly = false;
    boolean hostOnly = true;
    boolean persistent = false;//默认为会话cookie

//解析name=value;后面的几个属性
    pos = cookiePairEnd + 1;
    while (pos < limit) {
      int attributePairEnd = delimiterOffset(setCookie, pos, limit, ';');//拿到后一个分号的

      int attributeEqualsSign = delimiterOffset(setCookie, pos, attributePairEnd, '=');
      String attributeName = trimSubstring(setCookie, pos, attributeEqualsSign);//拿到属性名
//拿到属性值
      String attributeValue = attributeEqualsSign < attributePairEnd
          ? trimSubstring(setCookie, attributeEqualsSign + 1, attributePairEnd)
          : "";

//一个个匹配并赋值
      if (attributeName.equalsIgnoreCase("expires")) {
        try {
          expiresAt = parseExpires(attributeValue, 0, attributeValue.length());
          persistent = true;
        } catch (IllegalArgumentException e) {
          // Ignore this attribute, it isn't recognizable as a date.
        }
      } else if (attributeName.equalsIgnoreCase("max-age")) {
        try {
          deltaSeconds = parseMaxAge(attributeValue);
          persistent = true;
        } catch (NumberFormatException e) {
          // Ignore this attribute, it isn't recognizable as a max age.
        }
      } else if (attributeName.equalsIgnoreCase("domain")) {
        try {
          domain = parseDomain(attributeValue);
          hostOnly = false;
        } catch (IllegalArgumentException e) {
          // Ignore this attribute, it isn't recognizable as a domain.
        }
      } else if (attributeName.equalsIgnoreCase("path")) {
        path = attributeValue;
      } else if (attributeName.equalsIgnoreCase("secure")) {
        secureOnly = true;
      } else if (attributeName.equalsIgnoreCase("httponly")) {
        httpOnly = true;
      }

      pos = attributePairEnd + 1;
    }

//拿过期时间
    // If 'Max-Age' is present, it takes precedence over 'Expires', regardless of the order the two
    // attributes are declared in the cookie string.
    if (deltaSeconds == Long.MIN_VALUE) {
      expiresAt = Long.MIN_VALUE;
    } else if (deltaSeconds != -1L) {
      long deltaMilliseconds = deltaSeconds <= (Long.MAX_VALUE / 1000)
          ? deltaSeconds * 1000
          : Long.MAX_VALUE;
      expiresAt = currentTimeMillis + deltaMilliseconds;
      if (expiresAt < currentTimeMillis || expiresAt > HttpDate.MAX_DATE) {
        expiresAt = HttpDate.MAX_DATE; // Handle overflow & limit the date range.
      }
    }

    // If the domain is present, it must domain match. Otherwise we have a host-only cookie.
    if (domain == null) {
      domain = url.host();//有规定domain,就按domain的规则,没有,就严格匹配host
    } else if (!domainMatch(url, domain)) {//域不匹配,set cookie无效,弃之
      return null; // No domain match? This is either incompetence or malice!
    }

    // If the path is absent or didn't start with '/', use the default path. It's a string like
    // '/foo/bar' for a URL like 'http://example.com/foo/bar/baz'. It always starts with '/'.
    if (path == null || !path.startsWith("/")) {
      String encodedPath = url.encodedPath();
      int lastSlash = encodedPath.lastIndexOf('/');
      path = lastSlash != 0 ? encodedPath.substring(0, lastSlash) : "/";
    }

    return new Cookie(cookieName, cookieValue, expiresAt, domain, path, secureOnly, httpOnly,
        hostOnly, persistent);
  }

```

### 从response中解析出的List<Cookie>的保存
```
cookieJar.saveFromResponse(url, cookies);
```
追踪源码可以看到,cookieJar是由OkHttpClient传递过来的,是由OkHttpClient.Builder.cookieJar(cookieJar)传进来,具体由用户实现.OkHttpClient.Builder提供了默认实现:
```
cookieJar = CookieJar.NO_COOKIES;

```

其实现为:

```
public interface CookieJar {
  /** A cookie jar that never accepts any cookies. */
  CookieJar NO_COOKIES = new CookieJar() {
    @Override public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
    }

    @Override public List<Cookie> loadForRequest(HttpUrl url) {
      return Collections.emptyList();
    }
  };


  void saveFromResponse(HttpUrl url, List<Cookie> cookies);


  List<Cookie> loadForRequest(HttpUrl url);
}
```

### 再看request中添加cookie的方法
> 非常简单了,就是CookieJar接口的loadForRequest方法,默认是返回null,如果管理,需要用户自己实现

[BridgeInterceptor](https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/internal/http/BridgeInterceptor.java)的intercept方法中
```
  List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
//给request设置cookie,List<Cookie> cookies由cookieJar.loadForRequest方法返回,该方式由开发者实现.
    }

```

# 自定义cookie的管理策略
> 从上面的源码分析可以看出,cookie管理中,cookie解析和cookie设置两个部分都不用我们操心,框架帮我们做了,我们要做的是实现cookieJar来定制cookie的保存.

## 客户端的cookie策略
> 我们很容易就可以知道,有三种情况

1. 不要cookie
2. 会话cookie: cookie只存在内存中,客户端/浏览器一关闭,cookie就没了
3. 持久cookie: cookie写到硬盘上,下次客户端再打开时可以从上面读取,然后添加到request中

## 分别的实现

### 不要cookie
okhttp已经内置了,就是CookieJar.NO_COOKIES

### 会话cookie:
```
new CookieJar() {  
    private final HashMap<String, List<Cookie>> cookieStore = new HashMap<String, List<Cookie>>();  
  
    @Override  
    public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {  
        cookieStore.put(url.host(), cookies);  
    }  
  
    @Override  
    public List<Cookie> loadForRequest(HttpUrl url) {  
        List<Cookie> cookies = cookieStore.get(url.host());  
        return cookies != null ? cookies : new ArrayList<Cookie>();  
    }  
})

```

### 持久化cookie


```

private class CookiesManager implements CookieJar {
        private final PersistentCookieStore cookieStore = new PersistentCookieStore(getApplicationContext());

        @Override
        public void saveFromResponse(HttpUrl url, List<Cookie> cookies) {
            if (cookies != null && cookies.size() > 0) {
                for (Cookie item : cookies) {
                    cookieStore.add(url, item);
                }
            }
        }

        @Override
        public List<Cookie> loadForRequest(HttpUrl url) {
            List<Cookie> cookies = cookieStore.get(url);
            return cookies;
        }
    }

```

### PersistentCookieStore 的实现
> 用SharedPreferences来保存
> 注: 参考[OkHttp3实现Cookies管理及持久化](http://www.open-open.com/lib/view/open1453422314105.html)

```
public class PersistentCookieStore {
    private static final String LOG_TAG = "PersistentCookieStore";
    private static final String COOKIE_PREFS = "Cookies_Prefs";

    private final Map<String, ConcurrentHashMap<String, Cookie>> cookies;
    private final SharedPreferences cookiePrefs;


    public PersistentCookieStore(Context context) {
        cookiePrefs = context.getSharedPreferences(COOKIE_PREFS, 0);
        cookies = new HashMap<>();

        //将持久化的cookies缓存到内存中 即map cookies
        Map<String, ?> prefsMap = cookiePrefs.getAll();
        for (Map.Entry<String, ?> entry : prefsMap.entrySet()) {
            String[] cookieNames = TextUtils.split((String) entry.getValue(), ",");
            for (String name : cookieNames) {
                String encodedCookie = cookiePrefs.getString(name, null);
                if (encodedCookie != null) {
                    Cookie decodedCookie = decodeCookie(encodedCookie);
                    if (decodedCookie != null) {
                        if (!cookies.containsKey(entry.getKey())) {
                            cookies.put(entry.getKey(), new ConcurrentHashMap<String, Cookie>());
                        }
                        cookies.get(entry.getKey()).put(name, decodedCookie);
                    }
                }
            }
        }
    }

    protected String getCookieToken(Cookie cookie) {
        return cookie.name() + "@" + cookie.domain();
    }

    public void add(HttpUrl url, Cookie cookie) {
        String name = getCookieToken(cookie);

        //将cookies缓存到内存中 如果缓存过期 就重置此cookie
        if (!cookie.persistent()) {
            if (!cookies.containsKey(url.host())) {
                cookies.put(url.host(), new ConcurrentHashMap<String, Cookie>());
            }
            cookies.get(url.host()).put(name, cookie);
        } else {
            if (cookies.containsKey(url.host())) {
                cookies.get(url.host()).remove(name);
            }
        }

        //讲cookies持久化到本地
        SharedPreferences.Editor prefsWriter = cookiePrefs.edit();
        prefsWriter.putString(url.host(), TextUtils.join(",", cookies.get(url.host()).keySet()));
        prefsWriter.putString(name, encodeCookie(new SerializableOkHttpCookies(cookie)));
        prefsWriter.apply();
    }

    public List<Cookie> get(HttpUrl url) {
        ArrayList<Cookie> ret = new ArrayList<>();
        if (cookies.containsKey(url.host()))
            ret.addAll(cookies.get(url.host()).values());
        return ret;
    }

    public boolean removeAll() {
        SharedPreferences.Editor prefsWriter = cookiePrefs.edit();
        prefsWriter.clear();
        prefsWriter.apply();
        cookies.clear();
        return true;
    }

    public boolean remove(HttpUrl url, Cookie cookie) {
        String name = getCookieToken(cookie);

        if (cookies.containsKey(url.host()) && cookies.get(url.host()).containsKey(name)) {
            cookies.get(url.host()).remove(name);

            SharedPreferences.Editor prefsWriter = cookiePrefs.edit();
            if (cookiePrefs.contains(name)) {
                prefsWriter.remove(name);
            }
            prefsWriter.putString(url.host(), TextUtils.join(",", cookies.get(url.host()).keySet()));
            prefsWriter.apply();

            return true;
        } else {
            return false;
        }
    }

    public List<Cookie> getCookies() {
        ArrayList<Cookie> ret = new ArrayList<>();
        for (String key : cookies.keySet())
            ret.addAll(cookies.get(key).values());

        return ret;
    }

    /**
     * cookies 序列化成 string
     *
     * @param cookie 要序列化的cookie
     * @return 序列化之后的string
     */
    protected String encodeCookie(SerializableOkHttpCookies cookie) {
        if (cookie == null)
            return null;
        ByteArrayOutputStream os = new ByteArrayOutputStream();
        try {
            ObjectOutputStream outputStream = new ObjectOutputStream(os);
            outputStream.writeObject(cookie);
        } catch (IOException e) {
            Log.d(LOG_TAG, "IOException in encodeCookie", e);
            return null;
        }

        return byteArrayToHexString(os.toByteArray());
    }

    /**
     * 将字符串反序列化成cookies
     *
     * @param cookieString cookies string
     * @return cookie object
     */
    protected Cookie decodeCookie(String cookieString) {
        byte[] bytes = hexStringToByteArray(cookieString);
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
        Cookie cookie = null;
        try {
            ObjectInputStream objectInputStream = new ObjectInputStream(byteArrayInputStream);
            cookie = ((SerializableOkHttpCookies) objectInputStream.readObject()).getCookies();
        } catch (IOException e) {
            Log.d(LOG_TAG, "IOException in decodeCookie", e);
        } catch (ClassNotFoundException e) {
            Log.d(LOG_TAG, "ClassNotFoundException in decodeCookie", e);
        }

        return cookie;
    }

    /**
     * 二进制数组转十六进制字符串
     *
     * @param bytes byte array to be converted
     * @return string containing hex values
     */
    protected String byteArrayToHexString(byte[] bytes) {
        StringBuilder sb = new StringBuilder(bytes.length * 2);
        for (byte element : bytes) {
            int v = element & 0xff;
            if (v < 16) {
                sb.append('0');
            }
            sb.append(Integer.toHexString(v));
        }
        return sb.toString().toUpperCase(Locale.US);
    }

    /**
     * 十六进制字符串转二进制数组
     *
     * @param hexString string of hex-encoded values
     * @return decoded byte array
     */
    protected byte[] hexStringToByteArray(String hexString) {
        int len = hexString.length();
        byte[] data = new byte[len / 2];
        for (int i = 0; i < len; i += 2) {
            data[i / 2] = (byte) ((Character.digit(hexString.charAt(i), 16) << 4) + Character.digit(hexString.charAt(i + 1), 16));
        }
        return data;
    }
  }
```

## 更任性一点
> 既然List<Cookie> 是我返回给框架的,那么我可以改动,或者手动添加一个全新的cookie,这个就不展开了.

# 总结
 okhttp对cookie的实现中,做了两件事:
按照规范把response header中的"set-cookie"字段解析成List<Cookie>
把cookieJar返回的List<Cookie>设置到请求头中.

需要用户做的事是定义cookie的保存方式:
是不保存,还是保存在内存,还是持久化到硬盘,方式是通过实现CookieJar接口



# 封装到极致的网络库:
[HttpUtilForAndroid](https://github.com/hss01248/HttpUtilForAndroid)