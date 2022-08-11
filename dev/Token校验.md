# Token校验流程

流程上大致分为：

- 设置拦截器，拦截器拦截请求
- 获取请求中的请求头，根据前后端约定key获取token
- 尝试解析token，token不存在或者解析失败则返回403
- 解析成功后，读取payload，payload中存储着用户的mail和nickname
- 根据用户的基本信息去reids缓存中查找token

```java
 public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws IOException {

        try {
            //获取请求头中的token
            String token = request.getHeader(JwtUtil.JWT_KEY);

            //非空检验
            if (Objects.isNull(token))
                return false;

            //进行token校验
            if (!authenticator.authenticate(token))
                return false;

        }catch (Exception e){
            response.sendError(HttpStatus.FORBIDDEN.value());
            return false;
        }

        return true;
    }
```

```java
public Boolean authenticate(String token){

        try {
            //读取payload
            String payload = JwtUtil.parseJWT(token).getSubject();

            //JSON转换
            User user = JSON.parseObject(payload,User.class);

            //读取Redis缓存中的token
            if (Objects.isNull(redisCache.getCacheObject(user.getMail())))
                return false;

        }catch (Exception e){
            e.printStackTrace();
            return false;
        }

        return true;
    }
```

