---
title: '分布式限流: redis+令牌桶'
date: 2019-12-28 21:34:45
tags: redis
categories: 
- redis

---

限流一般有漏桶和令牌桶两种实现，详情网上很多资料。
一般都会选用令牌桶算法，比较灵活，可以支持预热、容忍一定突发流量、预支一部分流量的弹性。


# 单进程限流
单个jvm限流有现成的库，谷歌的guava库提供了`RateLimiter`,底层实现上有两个，一个是能容忍突发的实现，一个是能提供预热功能的实现；通过create时提供不同的参数来获得。
## 实现原理
### 朴素解法
如果按照令牌桶算法朴素的定义来实现的话，一个很自然的思路就是增加一个定时线程、进程，然后不断生成新的token; 
#### 朴素解法的缺点
重度依赖这个定时线程，如果定时程序挂了，所有工作线程就被卡住了，风险较大。
如果给定时线程加监控，又会需要监控的监控，那就变成俄罗斯套娃了。

### guava库中的解法
需要访问者多输入一个参数: 当前时钟。
`lazy eval`: 每次取token的时候才计算当前"应该"有多少token。
基于时钟来计算: 当前时钟下, 过去了多久没有生成token，应该生成多少新的token。
核心代码:
```java
/**
   * Updates {@code storedPermits} and {@code nextFreeTicketMicros} based on the current time.
   */
  void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
      storedPermits = min(maxPermits,
          storedPermits
            + (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros());
      nextFreeTicketMicros = nowMicros;
    }
  }
```
### guava库中解法的优点
开销更低: 去掉了定时线程的开销;
健壮性更高: 每个线程都可以承担生成新token的任务;

### guava库中解法的缺点
每个线程的时钟得对齐。但这一点在单进程场景下很好保证。

## 使用
```java
import com.google.common.util.concurrent.RateLimiter;
private static void run1() {
        RateLimiter limiter = RateLimiter.create(5);// 令牌桶
        while (true) {
            System.out.println("get 1 tokens: " + limiter.acquire() + "s"
            );

        }
    }

    /**
     * 3秒预热期: 预热期内限频较严格: 1.3s , 0.9s , 0.6s // 线性提速
     * 预热期后: 正式达到1秒2个的额定速度.
     */
    private static void run2() {// 预热测试
        RateLimiter r = RateLimiter.create(2, 3, TimeUnit.SECONDS);
        while (true) {
            System.out.println("get 1 tokens: " + r.acquire(1) + "s");
            System.out.println("get 1 tokens: " + r.acquire(1) + "s");
            System.out.println("get 1 tokens: " + r.acquire(1) + "s");
            System.out.println("get 1 tokens: " + r.acquire(1) + "s");
            System.out.println("end");
        }
    }

    private static void run3() {// 突发测试
        RateLimiter r = RateLimiter.create(5);
        while (true) {
            System.out.println("get 5 tokens: " + r.acquire(5) + "s");
            System.out.println("get 1 tokens: " + r.acquire(1) + "s");// 每次都负责还债
            System.out.println("get 1 tokens: " + r.acquire(1) + "s");
            System.out.println("get 1 tokens: " + r.acquire(1) + "s");
            System.out.println("end");
        }
    }
```



# 分布式限流
如果是多实例的情况下,下游处理能力有上限时(例如物料有限),需要对整体有一个限流。
可以参考guava的实现,实现一个基于redis的，有一定弹性的令牌桶实现:
(由于是借鉴guava的实现，所以有相同的依赖: 所有进程、机器的时钟对齐)
```java
public void tokenLimitDemo() {
        DefaultRedisScript<Long> lua = new DefaultRedisScript<>();
        lua.setResultType(Long.class);
        lua.setScriptSource(new ResourceScriptSource(new
                ClassPathResource("tokenLimiter.lua")));
        List<String> keys = new ArrayList<>();
        keys.add("test_ip");
        for (int i = 0; i < 100; i++) {
            Long res = (Long) redisTemplate.execute(lua, keys
                    , "1"
                    , String.valueOf(System.currentTimeMillis())
            );
            if (res != 1) {
                try {
                    System.out.println("waiting");
                    Thread.sleep(100);
                    i--;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } else {
                System.out.println(res);
            }
        }
}
```
相应的lua脚本: 
```lua
-- 令牌桶算法:
-- 1. 校验输入:
local need_token = tonumber(ARGV[1])
local req_time = tonumber(ARGV[2])
if type(need_token) ~= "number" or type(req_time) ~= "number" then
    return 0
end

-- 2. 校验redis:
local key = KEYS[1]
local info = redis.pcall("HMGET", key, "last_time", "cur_token_num", "max_token", "rate")
local last_time = tonumber(info[1])
local cur_token_num = tonumber(info[2])
local max_token = tonumber(info[3])
local rate = tonumber(info[4])

if type(last_time) ~= "number" then -- init
    last_time = req_time
    max_token = 2 -- 最大token弹性
    cur_token_num = max_token -- 假设已经预热完毕
    rate = 1
    -- 初始化redis:
    redis.pcall("HMSET", key, "last_time", req_time, "cur_token_num", cur_token_num)
    redis.pcall("HMSET", key, "max_token", max_token, "rate", rate)
end

-- 3. 处理请求:
local result = 0
if tonumber(need_token) > tonumber(max_token) then
    result = 0
end

local new_gen_token = math.floor((req_time - last_time) / 1000) * rate
local cur_token_num = math.min(cur_token_num + new_gen_token, max_token)

if tonumber(need_token) > tonumber(cur_token_num) then
    result = 0
else -- do consume token
    cur_token_num = cur_token_num - need_token
    result = 1
    -- 保存进度:
    redis.pcall("HMSET", key, "last_time", req_time, "cur_token_num", cur_token_num)
end

return result

```


