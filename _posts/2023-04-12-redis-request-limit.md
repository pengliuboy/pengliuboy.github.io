---
layout: post
custom_js: mouse_coords
---
# 使用Redis限流
### 原理
滑动窗口方式，限制窗口时间内的请求数量；使用Redis的zset数据结构，按照每次请求时间戳作为score存储到zset中

### 使用
1、移除窗口时间外的数据，按照请求时间core作为范围条件

```java
// ts 为当时时间戳，periodMillis为窗口时间，如30000毫秒
pipe.zRemRangeByScore(key, 0, ts - periodMillis);
```
2、获取当前set数量

```java
// 获取当前请求数量
Long count = pipe.zCard(key);
if (Objects.isNull(count)) {
   count = 0L;
}
```
3、添加请求到set

```java
// 没有超过限制，添加当前请求
if (count < limit) {
    pipe.zAdd(key, ts, String.valueOf(ts).getBytes(StandardCharsets.UTF_8));
}
```
4、为key设置有效时间

```java
pipe.expire(key, periodMillis / 1000);
```
### 代码
1、注解

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;
import java.util.concurrent.TimeUnit;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    String key() default "";
    TimeUnit timeUnit() default TimeUnit.SECONDS;
    int limit() default 0;
    String msg() default "请求过多";
}
```
2、切面

```java
import com.alibaba.fastjson.JSONObject;
import com.thc.platform.ai.annotation.RateLimit;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.data.redis.connection.RedisConnection;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.lang.reflect.Method;
import java.nio.charset.StandardCharsets;
import java.util.List;
import java.util.Objects;
import java.util.concurrent.TimeUnit;

/**
 * @ClassName RateLimitAspect
 * @Description 限流切面
 * @Version 1.0
 **/
@Slf4j
@Component
@Aspect
public class RateLimitAspect {

    @Resource
    private RedisTemplate<Object, Object> redisTemplate;

    @Around("@annotation(com.thc.platform.ai.annotation.RateLimit)")
    public Object handle(ProceedingJoinPoint joinPoint) {
        try {
            MethodSignature signature = (MethodSignature)joinPoint.getSignature();
            Method method = signature.getMethod();
            RateLimit rateLimit = method.getAnnotation(RateLimit.class);
            byte[] key = ("chat-gpt:" + rateLimit.key()).getBytes(StandardCharsets.UTF_8);
            TimeUnit timeUnit = rateLimit.timeUnit();
            long periodMillis = getPeriodMillis(timeUnit);

            int limit = rateLimit.limit();

            List<Object> result = redisTemplate.executePipelined((RedisConnection pipe) -> {
                long ts = System.currentTimeMillis();
                pipe.multi();
                // 移除当前窗口外的数据
                pipe.zRemRangeByScore(key, 0, ts - periodMillis);    //执行结果0
                // 获取当前请求数量
                Long count = pipe.zCard(key);    //执行结果1
                if (Objects.isNull(count)) {
                    count = 0L;
                }
                // 没有超过限制，添加当前请求
                if (count < limit) {
                    pipe.zAdd(key, ts, String.valueOf(ts).getBytes(StandardCharsets.UTF_8));    //执行结果2
                }
                pipe.expire(key, periodMillis / 1000);    //执行结果3
                pipe.exec();

                return null;
            });
            // 获取pipe的数据
            List<Object> objList = (List<Object>)result.get(0);
            if (objList.size() == 4) {
                long count = 0;
                // 获取执行结果1
                Object ctnObj = objList.get(1);
                if (Objects.nonNull(ctnObj) && ctnObj instanceof Long) {
                    count = (Long) ctnObj;
                }
                if (count > limit) {
                    return JSONObject.parseObject("{\"code\":400, \"msg\": \"" + rateLimit.msg() + "\"}");
                }
            }
            return joinPoint.proceed();
        } catch (Throwable e) {
            log.error("处理RateLimit注解失败，{}", e.getMessage());
        }

        return null;
    }

    private long getPeriodMillis(TimeUnit timeUnit) {

        if (timeUnit == TimeUnit.DAYS) {
            return 24 * 60 * 60 * 1000L;
        }

        if (timeUnit == TimeUnit.HOURS) {
            return 60 * 60 * 1000L;
        }

        if (timeUnit == TimeUnit.MINUTES) {
            return 60 * 1000;
        }

        if (timeUnit == TimeUnit.SECONDS) {
            return 1000;
        }

        return 1;
    }

}
```
3、业务使用

```java
// 限定30分钟内访问30次，超过后提示msg
@RateLimit(key = "chat", timeUnit = TimeUnit.MINUTES, limit = 30, msg = "访问受限")
@PostMapping("/gpt/v1/chat/completions")
public Object chat(@RequestBody JSONObject req) {
    return response(req, "chat");
}
```
