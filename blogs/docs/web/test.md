[实现原理不同](#实现原理不同)  
[使用范围不同](#使用范围不同)  
[触发时机不同](#触发时机不同)  
[拦截的请求范围不同](#拦截的请求范围不同)
### 实现原理不同
(1)过滤器是基于函数回调的；  
(2)拦截器则是基于Java的反射机制(动态代理)实现的。

这里重点说下过滤器: 过滤器
在我们自定义的过滤器中都会实现一个doFilter()方法，这个方法有一个FilterChain参数。而实际上FilterChain是一个回调接口，ApplicationFilterChain是它的实现类，这个实现类内部也有一个doFilter()方法，就是回调方法。
```java
public interface FilterChain {
    void doFilter(ServletRequest var1, ServletResponse var2) throws IOException, ServletException;
}
```
### 使用范围不同
### 触发时机不同
### 拦截的请求范围不同
