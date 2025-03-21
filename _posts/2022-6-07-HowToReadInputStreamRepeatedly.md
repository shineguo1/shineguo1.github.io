---
layout: post
title: 重复读httpServletRequest.getInputStream()的问题与解决
date: 2022-06-07
tags: java实践
---

### 问题描述
有一个场景需要过滤http请求参数，如果包含某个字段，就需要做一些处理。简简单单写个servlet的Filter，然后检查一下parameters和inputStream。然后发现只要读过inputStream，就无法mapping到对应的controller方法，会报400错。

### 原因
ServletInputStream不可重复读，如果在filter里读了inputStream，那么springMvc读到的inputStream就为空，于是就不能映射到对应的控制器方法。
一开始想法很简单，InputStream有mark和reset方法，可以重置stream状态，就可以重新读了，但是实际操作发现ServletInputStream没有实现这两个方法，也就是说从设计上就不支持重复读。所以核心思路是去重写这两个方法就能解决了。

### 预想方案
##### 1. MyFilter
1. 第一步是过滤器，如果是HttpServletRequest，那么就是我们监听的对象。
2. 使用MarkSupportedHttpServletRequestWrapper把servletRequest转成inpuStream支持mark和reset的包装类。
3. 在取json内容前后分别mark和reset复原状态即可。
```java
@Component
public class MyFilter extends HttpFilter {

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        if (servletRequest instanceof HttpServletRequest) {
            internalDoFilter((HttpServletRequest) servletRequest, servletResponse, filterChain);
        } else {
            filterChain.doFilter(servletRequest, servletResponse);
        }
    }

    private void internalDoFilter(HttpServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        MarkSupportedHttpServletRequestWrapper servletRequestWrapper = new MarkSupportedHttpServletRequestWrapper(servletRequest);
        JSONObject jsonParam = getJsonParam(servletRequestWrapper);
        //... do something
        filterChain.doFilter(servletRequestWrapper, servletResponse);

    }

    private JSONObject getJsonParam(MarkSupportedHttpServletRequestWrapper servletRequest) throws IOException {
        ServletInputStream inputStream = servletRequest.getInputStream();
        try {
            inputStream.mark(inputStream.available() + 1);
            BufferedReader streamReader = new BufferedReader(new InputStreamReader(inputStream, "UTF-8"));
            StringBuilder responseStrBuilder = new StringBuilder();
            String inputStr;
            while ((inputStr = streamReader.readLine()) != null) {
                responseStrBuilder.append(inputStr);
            }
            return JSONObject.parseObject(responseStrBuilder.toString());
        } catch (Exception e) {
            throw e;
        } finally {
            inputStream.reset();
        }
    }
}
```

##### 2. MarkSupportedHttpServletRequestWrapper
1. javax.servlet提供了HttpServletRequest的包装类，我只需要用自己的支持mark的servletInputStream替换掉原始的servletInputStream即可。
2. tomcat的ServletInputStream实现类是CoyoteInputStream，所以分成CoyoteInputStream的增强类和缺省的增强类两种情况，重写getInputStream()方法，返回我们的增强类。

```java

public class MarkSupportedHttpServletRequestWrapper extends HttpServletRequestWrapper {

    private ServletInputStream servletInputStream;

    /**
     * Constructs a request object wrapping the given request.
     *
     * @param request The request to wrap
     * @throws IllegalArgumentException if the request is null
     */
    public MarkSupportedHttpServletRequestWrapper(HttpServletRequest request) throws IOException {
        super(request);
        if (request.getInputStream() instanceof CoyoteInputStream) {
            servletInputStream = new MarkSupportedCoyoteInputStream((CoyoteInputStream) request.getInputStream());
        } else {
            servletInputStream = new DefaultMarkSupportedServletInputStream(request.getInputStream());
        }
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {
        return this.servletInputStream;
    }
}

```

##### 3. DefaultMarkSupportedServletInputStream
1. 对一般的ServletInputStream通用的解决方法，我们用支持mark和reset的ByteArrayInputStream作为底层数据结构，把输入流的值转存到字节输入流。
2. 实现ServletInputStream的所有接口。但发现isFinished、isReady、setReadListener方法无从下手（比如tomcat的readListener是挂在Request上的）。虽然解决了问题，但功能不完整，有些缺憾。


```java

public class DefaultMarkSupportedServletInputStream extends ServletInputStream {

    @Getter
    private byte[] buffer;
    private ByteArrayInputStream inputStream;

    public DefaultMarkSupportedServletInputStream(ServletInputStream servletInputStream) throws IOException {
        buffer = StreamUtils.copyToByteArray(servletInputStream);
        inputStream = new ByteArrayInputStream(buffer);
    }


    @Override
    public boolean isFinished() {
        return false;
    }

    @Override
    public boolean isReady() {
        return false;
    }

    @Override
    public void setReadListener(ReadListener listener) {
    }

    @Override
    public int read() throws IOException {
        return inputStream.read();
    }

    @Override
    public synchronized void mark(int readlimit) {
        inputStream.mark(readlimit);
    }

    @Override
    public void reset() throws IOException {
        inputStream.reset();
    }

    @Override
    public boolean markSupported() {
        return true;
    }
}
```

##### 4. MarkSupportedCoyoteInputStream
1. 因为DefaultMarkSupportedServletInputStream的实现并不完美，所以想给实际上的使用的实现类CoyoteInputStream写个特殊的增强类。
2. 因为CoyoteInputStream的数据是存在ib里的，ib本身提供了mark和reset方法，所以我想只要包装一下CoyoteInputStream，把ib的的mark和reset方法暴露出来就可以了。
3. InputBuffer的reset方法不会复原状态字段`state`,recycle方法会恢复成初始状态。两个都写上试一下。

```java

public class MarkSupportedCoyoteInputStream extends CoyoteInputStream {

    private InputBuffer ib;
    private CoyoteInputStream inputStream;

    public MarkSupportedCoyoteInputStream(CoyoteInputStream coyoteInputStream) {
        super(null);
        this.inputStream = coyoteInputStream;
        //ib在CoyoteInputStream中的作用域是protected，这个方法是自己写的工具类，通过反射取到protected作用域的变量值。
        this.ib = (InputBuffer) ReflectionUtils.getFieldValue(coyoteInputStream, "ib");
    }


    @Override
    public synchronized void mark(int readlimit) {
        try {
            ib.mark(readlimit);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void reset() throws IOException {
        //reset？recycle？
        ib.reset();
        ib.recycle();

    }

    @Override
    public boolean markSupported() {
        return true;
    }

    //... other override method

```


### 再次遭遇问题
1. 使用DefaultMarkSupportedServletInputStream包装ServletInputStream很顺利，像预想的一样能跳转到控制器方法。
2. 使用MarkSupportedCoyoteInputStream仍有问题，无论是reset和recycle，都不能使inputStream真正的复原.读取复原后，再读取，reader.readLine()返回是空值。综上所述，对于CoyoteInputStream的改造是失败的。


### 最后选择
　　要完美的解决，需要深入研究BufferedReader是如何读取CoyoteInputStream的，是哪个变量起到了控制作用，为什么reset和recycle都没有复原。但由于时间上不允许，DefaultMarkSupportedServletInputStream已经能解决问题，它的不优雅的小缺点在项目里没有实际的用途，未来也用不上，所以最后决定把MarkSupportedCoyoteInputStream去掉，直接使用DefaultMarkSupportedServletInputStream就好。


