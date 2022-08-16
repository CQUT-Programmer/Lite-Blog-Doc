# Token校验流程

参考:[一文教你搞定前后端所有鉴权方案，让你不再迷惘 - 掘金 (juejin.cn)](https://juejin.cn/post/7129298214959710244)

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac396824cc4b4a88b5f57d84c1f856cc~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)



**此图十分简单明了，不过实现起来却不简单，全过程花了我整整4天，个人实现的是access+refresh+redis，即双token验证且存储在redis中。**

------

**对于不同的情况：**

- 如果 access-token 过期，后台将会返回401
- 如果 access-token 非法，后台将会返回403
- 如果 refesh-token 过期或者非法，后台均会返回403



## 大致原理

双token验证，一个是access-token，一个是refresh-token。access-token是用来验证大部分请求的，因为请求比较频繁，而且涉及面比较广，对安全性要求比较高，所以时效性很短，通常在2小时左右。但是每两个小时就要重新登陆获取token的话，用户体验极差，这就衍生出了refresh-token，该token时效性比较长，请求次数比较少，用来刷新access-token。当access-token过期时，携带refresh-token请求刷新，后台就会返回一个新的access-token。



很多人认为jwt是无状态的，存在服务端违背了使用的初衷，不过我使用redis理由有以下两点：

- 因为jwt是无状态的，只要签发出去了就一直是有效的，服务端在jwt过期之前都无法进行任何干涉，redis可以设置key的过期时间，使用redis可以有效解决此问题，就可以通过服务端直接使某一个jwt过期。
- 如果只有一个服务器，一个服务端，这种简单的架构下不需要redis完全没有什么大问题，但如果是分布式，多地单点登陆，就必须使用到redis。



## 用户登陆

用户登陆成功后，通过调用authenticator处理access-token与refresh-token。

过程中会将这两个token存储在redis中，其中，redis中的键名我采用的是 uuid-jwtKey-mail, 除非有恶意脚本重复登陆，否则在大多数情况下该键名不会出现重复。处理完成后返回这两个token给前端。

返回的数据格式如下

```json
{
    "data": {
        "access": {
            "jwtKey": "5624d023ce7c5ce2081b64a1157c85dc773d3c9d",
            "token": "eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiJlNmRlNTVkMjRmNTI0NmYzYmE1MDM0Y2NlNzU1ZGJmNyIsInN1YiI6IntcImdlbmRlclwiOlwi55S3XCIsXCJsb2dpblRpbWVcIjpcIjIwMjItMDgtMTYgMTM6MTE6MTRcIixcIm1haWxcIjpcIjI2M0BxcS5jb21cIixcIm5pY2tOYW1lXCI6XCJ3eWhcIixcInV1aWRcIjpcIjhkMDM1NGY0MGI2MTRmOGM4MDFhYmY3ZDA5NmY1NDIxXCJ9IiwiaXNzIjoiZjQwZDI0NGFmZWIxZmQ3ZTI0NTVlOTdkMGRiYTg2MjQ1NTU5MWJkZCIsImlhdCI6MTY2MDYyNjY3NCwiZXhwIjoxNjYwNjMzODc0fQ.uT1w1QQYcuSePLZxxazAawb3iDqDB8Wg_SNTTZfEBoI"
        },
        "refresh": {
            "jwtKey": "506f5771e24330ae206c099e96d9bbb7c010b291",
            "token": "eyJhbGciOiJIUzI1NiJ9.eyJqdGkiOiI2ZjI3NTAzODczZmM0ZjgxODhmYjE0NGRhNDU1MmM1MSIsInN1YiI6IntcImdlbmRlclwiOlwi55S3XCIsXCJsb2dpblRpbWVcIjpcIjIwMjItMDgtMTYgMTM6MTE6MTRcIixcIm1haWxcIjpcIjI2M0BxcS5jb21cIixcIm5pY2tOYW1lXCI6XCJ3eWhcIixcInV1aWRcIjpcIjhkMDM1NGY0MGI2MTRmOGM4MDFhYmY3ZDA5NmY1NDIxXCJ9IiwiaXNzIjoiZjQwZDI0NGFmZWIxZmQ3ZTI0NTVlOTdkMGRiYTg2MjQ1NTU5MWJkZCIsImlhdCI6MTY2MDYyNjY3NSwiZXhwIjoxNjYxMjMxNDc1fQ.kss5ia7uGOsyKSXI1Y8hok80PhzTMFm2M2oZhA99OPQ"
        }
    },
    "code": 200,
    "msg": "登陆成功"
}
```

Token中payload中sub携带的数据格式如下

```java
public class UserTokenVo {

    private String mail;

    private String nickName;

    private String avatar;

    private String gender;

    private Integer roleId;

    private String loginTime;

    private String uuid;
}

```

发送给前端的是经过json转换的字符串，需要自行进行base64解码

## AuthInterceptor拦截器

此拦截器主要拦截大部分url的access-token校验

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws IOException {

        try {

            //获取请求头中的token
            String accessToken = request.getHeader(JwtUtil.JWT_ACCESS_KEY);

            //断言
            Assert.notNull(accessToken,"AccessToken为空");

            if (!authenticator.authenticateAccessToken(accessToken))
                throw new AuthException();

        } catch (ExpiredJwtException e) {
            //token过期
            response.sendError(HttpStatus.UNAUTHORIZED.value(),HttpStatus.UNAUTHORIZED.toString());
            return false;
        } catch (Exception e){
            response.sendError(HttpStatus.FORBIDDEN.value(),HttpStatus.FORBIDDEN.toString());
            return false;
        }

        return true;
    }
```

```java
    public Boolean authenticateAccessToken(String token) {

        //读取payload
        String payload = JwtUtil.parseAccessJwt(token).getSubject();

        //读取载荷
        UserTokenVo userVo = JSON.parseObject(payload, UserTokenVo.class);

        //redis中获取不到则过期
        if (Objects.isNull(
                redisCache.getCacheObject(
                        JwtUtil.getRedisAccessKey(
                                userVo.getMail(),
                                userVo.getUuid())
                )))
            throw new ExpiredJwtException(null, null, null);

        return true;
    }
```

如果access-token过期，则会返回401状态码，前端接收到后会申请刷新token。

## RefreshInterceptor拦截器

此拦截器只拦截一个url，即/auth/refreshToken,请求此header时，需要同时携带已过期的access-token和refresh-token

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        try {
            //获取refresh token
            String refreshToken = request.getHeader(JwtUtil.JWT_REFRESH_KEY);

            //获取accessToken
            String accessToken = request.getHeader(JwtUtil.JWT_ACCESS_KEY);

            //断言
            Assert.notNull(refreshToken,"refreshToken不能为空");
            Assert.notNull(accessToken,"accessToken不能");

            //校验
            if (!authenticator.authenticateRefreshToken(refreshToken,accessToken))
                throw new AuthException();

            //因为获取的是refresh-token，此时再做时效判断没有意义，解析失败一律403，需要重新登陆获取refresh-token
        } catch (Exception e) {
            response.sendError(HttpStatus.FORBIDDEN.value(), HttpStatus.FORBIDDEN.toString());
            return false;
        }

        return true;
    }
```

```java
public Boolean authenticateRefreshToken(String refreshToken, String accessToken) throws AuthException {

        Date absolutelyExpireTime = null;

        String accessUuid = null;

        try {
            JwtUtil.parseAccessJwt(accessToken);
        } catch (ExpiredJwtException e) {

            //获取预设的过期时间
            Long expireTime = e.getClaims().getExpiration().getTime();

            //计算出绝对过期时间
            absolutelyExpireTime = new Date(JwtUtil.JWT_TRANSITION_TTL + expireTime);

            accessUuid = JSON.parseObject(e.getClaims().getSubject(),UserTokenVo.class).getUuid();

            //这个时候如果还有什么其他的异常直接返回false
        } catch (Exception e) {
            return false;
        }

        //允许过期时间已过
        if (Objects.isNull(absolutelyExpireTime) || absolutelyExpireTime.getTime() < System.currentTimeMillis())
            return false;


        //读取payload
        String payload = JwtUtil.parseAccessJwt(refreshToken).getSubject();

        //读取载荷
        UserTokenVo refreshPayload = JSON.parseObject(payload, UserTokenVo.class);

        //uuid必须相同
        if (Objects.isNull(accessUuid) || !accessUuid.equals(refreshPayload.getUuid()))
            return false;

        //redis中获取不到则过期
        if (Objects.isNull(
                redisCache.getCacheObject(
                        JwtUtil.getRedisRefreshKey(
                                refreshPayload.getMail(), refreshPayload.getUuid()
                        ))))
            throw new AuthException();

        return true;
    }

```

