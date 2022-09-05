# Springboot+Redis+Lua实现IP限流与过滤



## 引言

在做项目ip限流这一块时，有思考过用java语言去编写逻辑，后面发现redis自带lua解释器，可以直接执行lua脚本代码，lua本身就有轻便简洁的优点，

使用 lua 脚本的好处：

- 原子操作：lua脚本是作为一个整体执行的，所以中间不会被其他命令插入。
- 减少网络开销：可以将多个请求通过脚本的形式一次发送，减少网络时延。
- 复用性：lua脚本可以常驻在redis内存中，所以在使用的时候，可以直接拿来复用，也减少了代码量。

**注意：**lua脚本在redis中是单线程运行的，如果发生错误的话会阻塞线程，所以一定要调试好脚本

**需要有一定的lua脚本基础**:[Lua 数据类型 | 菜鸟教程 (runoob.com)](https://www.runoob.com/lua/lua-data-types.html)

**需要有一定的Redis基础**:[Redis 脚本 | 菜鸟教程 (runoob.com)](https://www.runoob.com/redis/redis-scripting.html)



## 配置

在原有的redis工具类中已经封装了redisTemplate，我们需要新增一个execute执行lua脚本的方法。

```java
/**
     * 在redis中执行一个lua脚本，因为redis配置使用的序列化器是fastjson,所以采用json格式来进行序列化
     *
     * @param script 脚本
     * @param keys   键值集合
     * @param args   参数
     * @return 执行的结果
     */

    public RedisEvalRes execute(RedisScript<JSONObject> script, List<String> keys, Object... args) {
        return new RedisEvalRes(
                Objects.requireNonNull(
                        (JSONObject) redisTemplate.execute(script, keys, args)));
    }
```

```java
package com.lite.common.entity;

import com.alibaba.fastjson2.JSONObject;
import lombok.Data;

/**
 * @author Stranger
 * @version 1.0
 * @description: TODO
 * @date 2022/9/5 19:17
 */
@Data
public class RedisEvalRes {

    private static final String RES_FLAG = "result";
    private JSONObject result;

    public <T> T getResult(Class<T> tClass){
        return result.getObject(RES_FLAG,tClass);
    }

    public RedisEvalRes(JSONObject result) {
        this.result = result;
    }
}

```

需要注意的是，由于项目采用的是fastjson做redis的序列化与反序列化，即采用json字符串做载体，需要进行一定的json处理



> 开发环境是Windows，Redis在Linux上部署，由于编码以及文件的换行符配置导致Windows下计算的SHA1，与Redis在Linux下缓存的文件SHA1不匹配，导致每次都无法命中缓存，此时可以通过IDEA的文件换行设置，调整脚本文件使用Unix换行符，可以解决不同系统匹配问题。

![image-20220905200833312](assets/Redis%E8%84%9A%E6%9C%AC/image-20220905200833312.png)

## 简单使用



###概念

`KEYS` ,`ARGV` 是两个默认的全局变量,

`KEYS`代表的是键值列表,`ARGV`代表的是参数列表

`cjson`是一个由c语言编写的json库

`redis`是默认的redis操作对象，可以用方法由`call()` ,`pcall()`，两者的区别在于，前者发生异常时会将异常抛给调用者，后者则将异常包装成table返回



### 示例脚本

先执行redis命令

```
set blog hellowrold
```

```lua

local payload = {}

local RES_FLAG = "result";

local key = KEYS[1]

local obj = ARGV[1]

local val = redis.call('get',key)

payload[RES_FLAG] = val;

return cjson.encode(payload)

```

```java
@Test
void scripTest() {
    Resource resource = new ClassPathResource("lua/ipLimit.lua");

    RedisEvalRes evalRes = redisCache.execute(
        RedisJsonScript.of(resource),
        Collections.singletonList("blog"),
        1,2);

    log.info(evalRes.getResult(String.class));

}
```

输出结果

```
helloworld
```



## IP限流



lua脚本

```lua
local payload = {}

local RES_FLAG = "result";

--获取传入的键值
local key = KEYS[1]

--获取时间限制
local limitTime = tonumber(ARGV[1])

--获取次数限制
local maxCount = tonumber(ARGV[2])

--让value自增1,如果不存在会自动创建
redis.call('INCRBY', key, 1)

--获取当前次数
local currentCount = tonumber(redis.call('GET', key))

--大于限制次数则返回-1
if currentCount > maxCount then
    payload[RES_FLAG] = -1
    --等于1说明是第一次访问,则设置key的过期时间
elseif currentCount == 1 then
    payload[RES_FLAG] = currentCount
    redis.call("EXPIRE", key, limitTime)
else
    payload[RES_FLAG] = currentCount
end

return cjson.encode(payload)

```



java代码

```java
package com.lite.auth.aspectJ;

import com.lite.auth.exception.SystemBusyException;
import com.lite.auth.utils.LiteBlogContextUtils;
import com.lite.common.entity.RedisEvalRes;
import com.lite.common.serializer.RedisCache;
import com.lite.common.serializer.RedisJsonScript;
import com.lite.system.annotation.RateLimit;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.ClassPathResource;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.Collections;

/**
 * @author Stranger
 * @version 1.0
 * @description: TODO
 * @date 2022/9/5 20:25
 */
@Aspect
@Component
public class RateLimitAspect {


    @Autowired
    RedisCache redisCache;
    @Autowired
    LiteBlogContextUtils contextUtils;


    @Before(value = "@annotation(com.lite.system.annotation.RateLimit)")
    public void rateLimitInterceptor(JoinPoint joinpoint) throws SystemBusyException {

        MethodSignature methodSignature = (MethodSignature) joinpoint.getSignature();

        Method method = methodSignature.getMethod();

        RateLimit rateLimit = method.getAnnotation(RateLimit.class);

        //获取当前用户token携带的uuid与方法名拼接成redis key
        StringBuilder keuBuilder = new StringBuilder()
                .append(contextUtils.getLocalUserInfo().getUuid())
                .append(".")
                .append(method.getDeclaringClass())
                .append(".")
                .append(method.getName())
                .append(".")
                .append(Arrays.toString(method.getParameters()))
                .append(".")
                .append(method.getReturnType());

        RedisEvalRes evalRes= redisCache.execute(
                RedisJsonScript.of(new ClassPathResource("lua/ipLimit.lua")),
                Collections.singletonList(keuBuilder.toString()),
                rateLimit.limitTime(),rateLimit.maxCount());

        Long resultCode = evalRes.getResult(Long.class);

        if (resultCode == -1)
            throw new SystemBusyException("系统繁忙,请稍后再试");
    }
}

```

