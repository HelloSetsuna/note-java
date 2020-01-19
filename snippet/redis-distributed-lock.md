# 基于 单节点 Redis 实现分布式锁

这个属于比较烂大街的内容了, 这里我也是基于 SpringBoot2 比较新的Redis客户端 Lettuce 重新实现了下这个分布式锁, 作为一个SpringBean 在项目中使用会较方便些, 这里只使用单节点的 Redis服务,如果是多节点部署的Redis就会出问题了, 属于我另一个项目`emaxil` 里抽出的代码吧

## 代码实现
```  java
package com.veda.emaxil.util;

import cn.hutool.core.util.IdUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Component;

import java.util.Arrays;
import java.util.Objects;
import java.util.concurrent.TimeUnit;

/**
 * 基于 单节点 Redis 实现分布式锁
 * @author derick.jin 2019-12-30 18:58:00
 * @version 1.0
 **/
@Slf4j
@Component
public class RedisDistributedLock {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @Autowired
    private DefaultRedisScript<Long> redisScript;

    /**
     * 加锁
     * @param lockName 锁名称
     * @param timeoutMillis 锁超时时间 毫秒
     * @return 锁内容
     */
    public String doLock(String lockName, long timeoutMillis) {
        String lockValue = IdUtil.fastSimpleUUID();
        while (true) {
            Boolean isAbsent = stringRedisTemplate.opsForValue().setIfAbsent(lockName, lockValue, timeoutMillis, TimeUnit.MILLISECONDS);
            if (Objects.nonNull(isAbsent) && isAbsent) {
                log.info("do lock:{} success", lockName);
                return lockValue;
            }
            log.info("do lock:{} retry", lockName);
        }
    }

    /**
     * 解锁
     * @param lockName 锁名称
     * @param lockValue 锁内容
     * @return 是否解锁成功
     */
    public boolean unlock(String lockName, String lockValue) {
        // 使用 Lua 脚本 保证获取 和 删除是一组原子操作
        Long result = stringRedisTemplate.execute(redisScript, Arrays.asList(lockName, lockValue));
        log.info("un lock:{} finish:{}", lockName, result);
        return !Objects.isNull(result) && result == 1L;
    }

    @Bean
    public DefaultRedisScript<Long> defaultRedisScript() {
        DefaultRedisScript<Long> defaultRedisScript = new DefaultRedisScript<>();
        defaultRedisScript.setResultType(Long.class);
        defaultRedisScript.setScriptText("if redis.call('get', KEYS[1]) == KEYS[2] then return redis.call('del', KEYS[1]) else return 0 end");
        return defaultRedisScript;
    }
}

```

## 测试代码
```java
package com.veda.emaxil;

import com.veda.emaxil.util.RedisDistributedLock;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @author derick.jin 2019-12-30 20:07:00
 * @version 1.0
 **/
@SpringBootTest
public class RedisDistributedLuckTest {

    @Autowired
    private RedisDistributedLock redisDistributedLock;

    @Test
    public void test(){
        ExecutorService executorService = Executors.newFixedThreadPool(20);
        for (int i = 0; i < 10; i++) {
            executorService.execute(() -> {
                String lockValue = redisDistributedLock.doLock("lock", 1000);
                System.out.println("lalal~");
                redisDistributedLock.unlock("lock", lockValue);
            });
        }
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

```