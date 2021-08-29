### shiro的session

现象：请求立方体接口，每次都会生成新的sessionId。

调用链如下：

选择模型，点"确定"后，会触发过滤器：

AbstractShiroFilter：

```java
protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain) throws ServletException, IOException {
    Throwable t = null;

    try {
        final ServletRequest request = this.prepareServletRequest(servletRequest, servletResponse, chain);
        final ServletResponse response = this.prepareServletResponse(request, servletResponse, chain);
        Subject subject = this.createSubject(request, response);
```



```java
protected WebSubject createSubject(ServletRequest request, ServletResponse response) {
    return (new Builder(this.getSecurityManager(), request, response)).buildWebSubject();
}
```



WebSubject.class：

```java
public WebSubject buildWebSubject() {
    Subject subject = super.buildSubject();
    if (!(subject instanceof WebSubject)) {
        String msg = "Subject implementation returned from the SecurityManager was not a " + WebSubject.class.getName() + " implementation.  Please ensure a Web-enabled SecurityManager has been configured and made available to this builder.";
        throw new IllegalStateException(msg);
    } else {
        return (WebSubject)subject;
    }
}
```



Subject.class:

```java
public Subject buildSubject() {
    return this.securityManager.createSubject(this.subjectContext);
}
```



DefaultSecurityManager:

```java
public Subject createSubject(SubjectContext subjectContext) {
        SubjectContext context = this.copy(subjectContext);
        context = this.ensureSecurityManager(context);
        context = this.resolveSession(context);
        context = this.resolvePrincipals(context);
        Subject subject = this.doCreateSubject(context);
        this.save(subject);
        return subject;
    }
```



```

protected Subject doCreateSubject(SubjectContext context) {
        return this.getSubjectFactory().createSubject(context);
    }
```



DefaultWebSubjectFactory:

```java
createSubject:
     return new WebDelegatingSubject(principals, authenticated, host, session, sessionEnabled, request, response, securityManager);
     
     
```



我们自定义的拦截器：

SessionAuthcFilter：

```java
protected boolean preHandle(ServletRequest request, ServletResponse response)
      throws Exception {
   Subject us = SecurityUtils.getSubject(); 
   Session session = us.getSession();
```

