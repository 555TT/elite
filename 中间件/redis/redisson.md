# redisson功能深度解读

## 分布式锁

...

## 限流器RedissonRateLimiter

### 实践用法

```Java
public class SlidingWindowRateLimiter{

    private RedissonClient redissonClient;

    private static final String LIMIT_KEY_PREFIX = "redisson:demo:limit:";

    public SlidingWindowRateLimiter(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }
    
    public Boolean tryAcquire(String key, int limit, int windowSize) {
        RRateLimiter rRateLimiter = redissonClient.getRateLimiter(LIMIT_KEY_PREFIX + key);

        if (!rRateLimiter.isExists()) {//如果限流器未创建过，设置限流器参数
            rRateLimiter.trySetRate(RateType.OVERALL, limit, windowSize, RateIntervalUnit.SECONDS);
        }

        return rRateLimiter.tryAcquire();
    }
}
//在需要用到的地方直接
Boolean access = slidingWindowRateLimiter.tryAcquire(telephone, 1, 60);//表示针对某个手机号限流，一分钟只能请求一次（防止重复刷取手机验证码）
```

### 限流原理解读

1.限流器初始化：

从demo代码rRateLimiter.trySetRate()一直点到org.redisson#trySetRateAsync(RateType type, long rate, long rateInterval, RateIntervalUnit unit):

```Java
public RFuture<Boolean> trySetRateAsync(RateType type, long rate, long rateInterval, RateIntervalUnit unit) {
        return commandExecutor.evalWriteNoRetryAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                "redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);"
              + "redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);"
              + "return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);",
                Collections.singletonList(getRawName()), rate, unit.toMillis(rateInterval), type.ordinal());
    }
```

重点看lua脚本：

```lua
redis.call('hsetnx', KEYS[1], 'rate', ARGV[1]);
redis.call('hsetnx', KEYS[1], 'interval', ARGV[2]);
return redis.call('hsetnx', KEYS[1], 'type', ARGV[3]);
```

比较简单，就是往一个Hash结构中设置速率、间隔时间、模式（单机or集群）

2.获取令牌

org.redisson.RedissonRateLimiter#tryAcquireAsync(org.redisson.client.protocol.RedisCommand<T>, java.lang.Long)：

```lua

-- 速率
local rate = redis.call("hget", KEYS[1], "rate")
-- 时间区间(ms)
local interval = redis.call("hget", KEYS[1], "interval")
local type = redis.call("hget", KEYS[1], "type")
assert(rate ~= false and interval ~= false and type ~= false, "RateLimiter is not initialized")

-- {name}:value 分析后面的代码，这个key记录的是当前令牌桶中的令牌数
local valueName = KEYS[2]

-- {name}:permits 这个key是一个zset，记录了请求的令牌数，score则为请求的时间戳
local permitsName = KEYS[4]

-- 单机限流才会用到，集群模式不用关注
if type == "1" then
    valueName = KEYS[3]
    permitsName = KEYS[5]
end

-- 原版本有bug(https://github.com/redisson/redisson/issues/3197)，最新版将这行代码提前了
-- rate为1 arg1这里是 请求的令牌数量(默认是1)。rate必须比请求的令牌数大
assert(tonumber(rate) >= tonumber(ARGV[1]), "Requested permits amount could not exceed defined rate")

-- 第一次执行这里应该是null，会进到else分支
-- 第二次执行到这里由于else分支中已经放了valueName的值进去，所以第二次会进if分支
local currentValue = redis.call("get", valueName)
if currentValue ~= false then
    -- 从第一次设的zset中取数据，范围是0 ~ (第二次请求时间戳 - 令牌生产的时间)
    -- 可以看到，如果第二次请求时间距离第一次请求时间很短(小于令牌产生的时间)，那么这个差值将小于上一次请求的时间，取出来的将会是空列表。反之，能取出之前的请求信息
    -- 这里作者将这个取出来的数据命名为expiredValues，可认为指的是过期的数据
    local expiredValues = redis.call("zrangebyscore", permitsName, 0, tonumber(ARGV[2]) - interval)
    local released = 0
    -- lua迭代器，遍历expiredValues，如果有值，那么released等于之前所有请求的令牌数之和，表示应该释放多少令牌
    for i, v in ipairs(expiredValues) do
        local random, permits = struct.unpack("fI", v)
        released = released + permits
    end

    -- 没有过期请求的话，released还是0，这个if不会进，有过期请求才会进
    if released > 0 then
        -- 移除zset中所有元素，重置周期
        redis.call("zrem", permitsName, unpack(expiredValues))
        currentValue = tonumber(currentValue) + released
        redis.call("set", valueName, currentValue)
    end

    -- 这里简单分析下上面这段代码:
    -- 1. 只有超过了1个令牌生产周期后的请求，expiredValues才会有值。
    -- 2. 以rate为3举例，如果之前发生了两个请求那么现在released为2，currentValue为1 + 2 = 3
    -- 以此可以看到，redisson的令牌桶放令牌操作是通过请求时间窗来做的，如果距离上一个请求的时间已经超过了一个令牌生产周期时间，那么令牌桶中的令牌应该得到重置，表示生产rate数量的令牌。

    -- 如果当前令牌数 ＜ 请求的令牌数
    if tonumber(currentValue) < tonumber(ARGV[1]) then
        -- 从zset中找到距离当前时间最近的那个请求，也就是上一次放进去的请求信息
        local nearest = redis.call('zrangebyscore', permitsName, '(' .. (tonumber(ARGV[2]) - interval), tonumber(ARGV[2]), 'withscores', 'limit', 0, 1); 
        local random, permits = struct.unpack("fI", nearest[1])
        -- 返回 上一次请求的时间戳 - (当前时间戳 - 令牌生成的时间间隔) 这个值表示还需要多久才能生产出足够的令牌
        return tonumber(nearest[2]) - (tonumber(ARGV[2]) - interval)
    else
        -- 如果当前令牌数 ≥ 请求的令牌数，表示令牌够多，更新zset
        redis.call("zadd", permitsName, ARGV[2], struct.pack("fI", ARGV[3], ARGV[1]))
        -- valueName存的是当前总令牌数，-1表示取走一个
        redis.call("decrby", valueName, ARGV[1])
        return nil
    end
else
    -- set一个key-value数据 记录当前限流器的令牌数
    redis.call("set", valueName, rate)
    -- 建了一个以当前限流器名称相关的zset，并存入 以score为当前时间戳，以lua格式化字符串{当前时间戳为种子的随机数、请求的令牌数}为value的值。
    -- struct.pack第一个参数表示格式字符串，f是浮点数、I是长整数。所以这个格式字符串表示的是把一个浮点数和长整数拼起来的结构体。我的理解就是往zset里记录了最后一次请求的时间戳和请求的令牌数
    redis.call("zadd", permitsName, ARGV[2], struct.pack("fI", ARGV[3], ARGV[1]))
    -- 从总共的令牌数 减去 请求的令牌数。
    redis.call("decrby", valueName, ARGV[1])
    return nil
end
```

总结一下，redisson用了`zset`来记录请求的信息，这样可以非常巧妙的通过比较score，也就是请求的时间戳，来判断当前请求距离上一个请求有没有超过一个令牌生产周期。如果超过了，则说明令牌桶中的令牌需要生产，之前用掉了多少个就生产多少个，而之前用掉了多少个令牌的信息也在zset中保存了。

然后比较当前令牌桶中令牌的数量，如果足够多就返回了，如果不够多则返回到下一个令牌生产还需要多少时间。这个返回值特别重要。

接下来就是回到java代码，各个API入口点进去，最后都会调到`org.redisson.RedissonRateLimiter#tryAcquireAsync(long, org.redisson.misc.RPromise<java.lang.Boolean>, long)`这个方法

```java
private void tryAcquireAsync(long permits, RPromise<Boolean> promise, long timeoutInMillis) {
    long s = System.currentTimeMillis();
    RFuture<Long> future = tryAcquireAsync(RedisCommands.EVAL_LONG, permits);
    future.onComplete((delay, e) -> {
        if (e != null) {
            promise.tryFailure(e);
            return;
        }
        
        if (delay == null) {
            //delay就是lua返回的 还需要多久才会有令牌
            promise.trySuccess(true);
            return;
        }
        
        //没有手动设置超时时间的逻辑
        if (timeoutInMillis == -1) {
            //延迟delay时间后重新执行一次拿令牌的动作
            commandExecutor.getConnectionManager().getGroup().schedule(() -> {
                tryAcquireAsync(permits, promise, timeoutInMillis);
            }, delay, TimeUnit.MILLISECONDS);
            return;
        }
        
        //el 请求redis拿令牌的耗时
        long el = System.currentTimeMillis() - s;
        //如果设置了超时时间，那么应该减去拿令牌的耗时
        long remains = timeoutInMillis - el;
        if (remains <= 0) {
            //如果那令牌的时间比设置的超时时间还要大的话直接就false了
            promise.trySuccess(false);
            return;
        }
        //比如设置的的超时时间为1s，delay为1500ms，那么1s后告知失败
        if (remains < delay) {
            commandExecutor.getConnectionManager().getGroup().schedule(() -> {
                promise.trySuccess(false);
            }, remains, TimeUnit.MILLISECONDS);
        } else {
            long start = System.currentTimeMillis();
            commandExecutor.getConnectionManager().getGroup().schedule(() -> {
                //因为这里是异步的，所以真正再次拿令牌之前再检查一下过去了多久时间。如果过去的时间比设置的超时时间大的话，直接false
                long elapsed = System.currentTimeMillis() - start;
                if (remains <= elapsed) {
                    promise.trySuccess(false);
                    return;
                }
                //再次拿令牌
                tryAcquireAsync(permits, promise, remains - elapsed);
            }, delay, TimeUnit.MILLISECONDS);
        }
    });
}
```

Java客户端拿到redis返回的`下一个令牌生产完成还需要多少时间`，也就是`delay`字段。如果这个delay为null，则表示成功获得令牌，如果没拿到，则过delay时间后通过异步线程再次发起拿令牌的动作。这里也可以看到，redisson的RateLimiter是非公平的，多个线程同时拿不到令牌的话并不保证先请求的会先拿到令牌。