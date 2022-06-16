---
layout: post
title: Spring-Security-Oauth2.0
date: 2021-12-13
tags: 计算机基础
---

### 文档链接
- [spring-security 中文文档](https://www.springcloud.cc/spring-security-zhcn.html)
- [spring-security-oauth2 官方文档](https://projects.spring.io/spring-security-oauth/docs/oauth2.html)
- [spring-security-oauth2 github wiki](https://github.com/spring-projects/spring-security/wiki/OAuth-2.0-Migration-Guide)

### 一、OAuth2 介绍
[阮一峰：OAuth 2.0 的四种方式](http://www.ruanyifeng.com/blog/2019/04/oauth-grant-types.html)

### 二、源码解析：授权码模式
- 说明：这一章节所有出现的回调地址统一使用`www.baidu.com`举例。

#### 0. 概念解释
- user认证信息：登录用户认证信息，接口入参authentication能够描述的一种数据。
- client认证信息：注册在服务器上的客户端。<br/>
　　　　　　client配置在AuthorizationServerConfigurerAdapter中的ClientDetailsServiceConfigurer。<br/>
　　　　　　client也是接口入参authentication能够描述的一种数据。
- code：client从oauth服务器获得的授权码叫code。授权码只能使用一次(用户每次授权，即便是同一个client，也都是不同的授权码)。
- accessToken：验权token。oauth服务器能根据这个token识别client被user授予了什么权限。
- refreshToken：刷新token。用来刷新accessToken。
- AuthorizationServerConfigurerAdapter：AuthorizationServer配置的适配类，OAuth2的配置类继承于这个类。提供访问安全类配置、访问端点类配置、client配置。
- AuthorizationServerSecurityConfigurer：访问安全配置。AuthorizationServerConfigurerAdapter的3个configure之一。提供访问权限、client加密方式、token过滤链等配置。
- AuthorizationServerEndpointsConfigurer：访问端点配置。能够装载TokenEndpoint和AuthorizationEndpoint的属性，包括AuthorizationServer、TokenServices、TokenStore、ClientDetailsService、UserDetailsService和redirectResolver、自定义回调页面等。
- ClientDetailsServiceConfigurer：客户端配置。配置客户端数据，包含内存型、jdbc型、ClientDetailsService自定义型三种。
- AuthorizationCodeService 授权码grant_code存储库，存储授权码和登录认证信息OAuth2Authentication的关系，包含内存型，jdbc型，random型(base类，可自行扩展成redis等)
- TokenStore：accessToken存储库。包含内存型、jdbc型、jwt型、jwk型、redis型5种，也可自行扩展。
- AuthorizationEndpoint：client申请权限及用户授权的端点。
- TokenEndpoint：用户授权后创建accessToken，及刷新accessToken的端点。
- RedirectResolver：重定向处理器。


#### 1. 获得授权码
- GET：http://localhost:8081/oauth/authorize?client_id=myClient&response_type=code&redirectUri=?&scope=pay%20account<br/>
  参数: client_id(必填)、response_type(必填)、redirectUri(必填)、scope(可选；默认取所有；多个值用空格隔开)
- 重定向：https://www.baidu.com/?code=VYMph2 ( https://www.baidu.com是Oauth2Config配置的redirectUri,code是返回的授权码)
- 重定向(错误信息):https://www.baidu.com/?error=invalid_scope&error_description=Invalid scope: &scope=all Account pay


**源码解析**：

- 入口：`@RequestMapping(value = "/oauth/authorize")`路径请求进入下面这个接口　<br/>
  `org.springframework.security.oauth2.provider.endpoint.AuthorizationEndpoint#authorize`
- 接口校验参数`response_type`必须包含“token”或“code”
- 接口校验参数`clientId`必须非空
- 读取client配置（config配置于AuthorizationServerConfigurerAdapter的实现类）
- 解析回调地址：`String resolvedRedirect = redirectResolver.resolveRedirect(redirectUriParameter, client);`
1. 校验client配置了授权码模式：client配置的`authorizedGrantTypes`非空，并且包含"implicit"或"authorization_code"（仅隐藏模式和授权码模式拥有回调地址）
2. 校验请求中的回调地址在client配置的回调地址集合中：`DefaultRedirectResolver#redirectMatches` 校验请求uri和注册uri的schema、userInfo、host、port、path、queryParams
```
// 示例：http://www.sina.com/aabc/def?num=1
{
    shcema: "http",
    userInfo: null,
    host: "www.sina.com",
    port: null,
    path: "/aabc/def",
    queryParams: [{num   : 1}] //数组
}
```
3. 重定向到`forward:/oauth/confirm_access`
        - 默认实现`WhitelabelApprovalEndpoint#getAccessConfirmation`，html文本由java代码字符串拼接。
        - 可以通过Oauth2配置类（AuthorizationServerConfigurerAdapter实现类）的`endpoints.pathMapping("/oauth/confirm_assess","/my/confirm_assess")`自定义授权表单提交页面路径。
```
//附：spring-security-oauth项目源码demo的配置
//路径：demo.Application.OAuth2Config#configure
endpoints.authenticationManager(authenticationManager)
    .pathMapping("/oauth/confirm_access", confirmPath)
    .pathMapping("/oauth/token", tokenPath)
    .pathMapping("/oauth/check_token", checkTokenPath)
    .pathMapping("/oauth/token_key", tokenKeyPath)
    .pathMapping("/oauth/authorize", authorizePath);
```

---
#### 2. 用户授权
POST `/oauth/authorize`返回给用户的授权页面提供一个授权表单，用户提交发送下面这个请求：
- Request URL: http://localhost:8081/oauth/authorize
- Authorization: *********
- Referer: http://localhost:8081/oauth/authorize?client_id=myClient&response_type=code&redirect_uri=http://www.baidu.com
- FormData:
```
{
    user_oauth_approval: true, //必传
    scope.all: true,    //scope+点+权限范围名称
    scope.account: false,
    scope.pay: false,
    authorize: Authorize    //必传
} //
```


**样例回调地址**
- uri：`https://www.baidu.com/?code=rf2dHP`，参数code是返回授权码。
- response header：`Location: http://www.baidu.com?code=rf2dHP`

**源码解析**
- 入口：`AuthorizationEndpoint#approveOrDeny`
1. 校验登录状态
2. 校验scope，设置authorizationRequest.scope和authorizationRequest.approved （`ApprovalStoreUserApprovalHandler#updateAfterApproval`）
```
// authorizationRequest.scope只保留同意授权的scope(且必须是client配置中拥有的scope)。
// 若存在同意授权的scope，设置authorizationRequest.approved = true；若全不同意授权，则为false。
authorizationRequest = userApprovalHandler.updateAfterApproval(authorizationRequest,
        (Authentication) principal);
//以下两句是冗余代码，authorizationRequest.approved在上面那句已经设置好了。
boolean approved = userApprovalHandler.isApproved(authorizationRequest, (Authentication) principal);
    authorizationRequest.setApproved(approved);
```
3. 若authorizationRequest.approved为true，返回成功回调；若authorizationRequest.approved为false，返回失败回调。<br/>
　　成功回调样例：`https://www.baidu.com/?code=qix4K5`<br/>
　　失败回调样例：`https://www.baidu.com/?error=access_denied&error_description=User%20denied%20access`


#### 3. 获取token
- POST：http://localhost:8081/oauth/token?code=VYMph2&grant_type=authorization_code&redirect_uri=http://www.baidu.com&scope=account
- response:
```
{
  "access_token": "f85b6742-8df5-4a8f-b9a8-9846fe944de1",
  "token_type": "bearer",
  "refresh_token": "1f69687b-ed31-4b13-9031-a6f5aa4c93c8",
  "expires_in": 43199,
  "scope": "all"
}
```

**源码解析**
- 入口：`TokenEndpoint#postAccessToken`
1. 入参principal描述的是client而非user，从中获取client信息。
2. 校验入参scope都包含于client注册的scopes中。
```
oAuth2RequestValidator.validateScope(tokenRequest, authenticatedClient);
```
3. 获取授权用户信息的方式：
核心代码调用链：
- 由`getTokenGranter().grant`进入
- 由这个实现类`AbstractTokenGranter`进入`getAccessToken`
- 调用`AuthorizationCodeTokenGranter`实现类的`getOAuth2Authentication`方法
- 方法中核心代码`authorizationCodeServices.consumeAuthorizationCode(authorizationCode);`通过授权码确定授权的用户。
- authorizationCodeServices默认使用内存实现，可以自定义配置在AuthorizationServerConfigurerAdapter中。（我的activiti-learning项目demo中自定义了redis_mock实现类）

4.  `OAuth2Authentication#getOAuth2Request` 这里重置了scope，用缓存的scope替换`了/oauth/token`请求中的scope，避免攻击。
     取对象的这个方法(属性)`authentication.getOAuth2Request() // authentication.storedRequest`

5. 创建token的过程。
 1. `DefaultTokenServices#tokenStore` 缓存了`accessToken。缓存未过期，同一用户同一授权，多次用不同授权码获取的accessToken相同
 2. `refreshToken`是accessToken的属性，若accessToken未过期，同样复用同一个refreshToken

```
public OAuth2AccessToken createAccessToken(OAuth2Authentication authentication) throws AuthenticationException {
    
    //从tokenStore查token是否已存在，对于key即authentication的判断唯一条件是client_id + response_type + redirect_uri + 授权scope
    OAuth2AccessToken existingAccessToken = tokenStore.getAccessToken(authentication);
    OAuth2RefreshToken refreshToken = null;
    if (existingAccessToken != null) {
        if (existingAccessToken.isExpired()) {
            //如果existingAccessToken过期了，复用它的refreshToken，因为客户端仍可能用这个refreshToken刷新新的accessToken
            if (existingAccessToken.getRefreshToken() != null) {
                refreshToken = existingAccessToken.getRefreshToken();
                // The token store could remove the refresh token when the
                // access token is removed, but we want to
                // be sure...
                tokenStore.removeRefreshToken(refreshToken);
            }
            tokenStore.removeAccessToken(existingAccessToken);
        }
        else {
            //如果authentication发生变化，创建新token
            // Re-store the access token in case the authentication has changed
            tokenStore.storeAccessToken(existingAccessToken, authentication);
            return existingAccessToken;
        }
    }
    
    /* === 以下代码是 accessToken不存在或者过期 的情况=== */
    
    // Only create a new refresh token if there wasn't an existing one
    // associated with an expired access token.
    // Clients might be holding existing refresh tokens, so we re-use it in
    // the case that the old access token
    // expired.
    //refreshToken不存在，创建新refreshToken。
    //在不考虑数据丢失的情况下，只有在accessToken不存在时会创建新refreshToken，因为accessToken过期时，我们会优先复用旧的refreshToken，以保证支持客户端用原refreshToken刷新token。
    if (refreshToken == null) {
        refreshToken = createRefreshToken(authentication);
    }
    // But the refresh token itself might need to be re-issued if it has
    // expired.
    //但如果token是过期的，虽然支持客户端用旧refreshToken刷新token，但需要更新refreshToken。
    else if (refreshToken instanceof ExpiringOAuth2RefreshToken) {
        ExpiringOAuth2RefreshToken expiring = (ExpiringOAuth2RefreshToken) refreshToken;
        if (System.currentTimeMillis() > expiring.getExpiration().getTime()) {
            refreshToken = createRefreshToken(authentication);
        }
    }
    
    //因为accessToken不存在或者过期，创建新token
    OAuth2AccessToken accessToken = createAccessToken(authentication, refreshToken);
    tokenStore.storeAccessToken(accessToken, authentication);
    // In case it was modified
    refreshToken = accessToken.getRefreshToken();
    if (refreshToken != null) {
        tokenStore.storeRefreshToken(refreshToken, authentication);
    }
    return accessToken;

}
```


### 三、OAUTH2实践操作流程：授权码模式

1. 配置`WebSecurityConfigurerAdapter`、`AuthorizationServerConfigurerAdapter`、`ResourceServerConfigurerAdapter`
2. 应用向授权服务端申请授权码，接口：`GET /oauth/authorize`。
3. 用户在跳转页面授权，接口： `POST /oauth/authorize`。
4. 应用在回调页面上拿到`grant_code`, 然后向授权服务端请求`access_token`，接口：`POST /oauth/token`。
5. 应用使用`access_token`作为`bearer token`，向资源服务器发送请求。
6. 应用使用申请access_token时返回的`refresh_token`，向授权服务端请求刷新token，接口`POST /oauth/token`。



### 四、OAUTH2实践问题记录：授权码模式

1. `/oauth/authorize`接口没有跳转登录页面，显示堆栈报错

```
{"@type":"java.lang.RuntimeException","localizedMessage":"请求访问：/error，认证失败，无法访问系统资源","message":"请求访问：/error，认证失败，无法访问系统资源","stackTrace":
```
原因：默认流程是进入授权页面，发现未登录（认证失败），跳转登录页面。我自定义配置了认证失败处理器`.exceptionHandling().authenticationEntryPoint(unauthorizedHandler)`，处理器会打印错误信息，于是失去了跳转登录页的效果。
 
2. `/login`登录成功后仍然没有登录状态，需要无限循环登录。
原因：`.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)`取消了session缓存认证信息，又没有实现token认证，导致服务器无法记住用户认证状态。

3. 使用accessToken获取资源报Unauthorized错误（拥有scope）

 ```
   {
       "timestamp": "2021-12-23T02:45:39.440+0000",
       "status": 401,
       "error": "Unauthorized",
       "message": "Unauthorized",
       "path": "/OAuth2Demo/oauthTest/account"
   }
```
原因：没有配置resourceServerConfigAdapt，即没有设置资源服务配置。
 
4. 使用accessToken获取资源报无resource_id错误（拥有scope）
{
    "error": "access_denied",
    "error_description": "Invalid token does not contain resource id (oauth2-resource)"
}
 原因：授权端配置了client的resourceId时，资源服务端没有配置resourceId。
 解法：(1) 可以在token端配置client时去掉resourceId。(2) 可以在resourceServer配置加上resourceId。


5.  使用accessToken获取资源无scope权限

```
{
    "error": "insufficient_scope",
    "error_description": "Insufficient scope for this resource",
    "scope": "pay"
}
```

