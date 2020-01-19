# 线程安全 带过期时间 的 数据缓存工具类 实现

> 虽然很多时候会使用一些分布式的缓存或第三方 的缓存 组件, 但是在业务代码中为了优化性能时需要的可能只是简单的 线程安全的缓存部分数据 ,  并不需要 太重量级的组件 , 以下我编写的这个缓存工具类其实是可以使用 SpringCache 来代替的,  但是这个配置起来会比较 麻烦些, 项目中 SpringCache配置了基于Redis的 分布式缓存 了,  为了 提高性能只在本地内存中维护 一个缓存, 设计完善了此缓存工具类 , 当然也可以再配置一个基于 JCache等 的 SpringCache实例 ,但是这些缓存组件都会对缓存的对象进行对象拷贝后再返回给调用 方 ,这在提供 给 别人用时是没有问题的 ,但是自己 内部使用就会有些浪费性能和内存了. 通过使用泛型的K 和 V 来分别表示缓存的key和对应的值的类型, CacheSupplier<K, V> 接口则负责提供获取数据的过程, 如果获取key对应的值找不到时就会调用CacheSupplier<K, V> 的 supply 方法获取数据并将数据缓存起来, 同时设置了缓存过期时间, supply加载完成后会标记这个key的过期时间戳, 获取数据时会验证时间戳. 

## 代码实现
``` java
import lombok.extern.slf4j.Slf4j;

import java.util.HashMap;
import java.util.Map;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;

/**
 * 线程安全的数据缓存的简单实现
 * 注意: 请不要修改返回值的缓存内容 为了更好的性能未对缓存结果进行对象拷贝
 * 由于很多时候在类内部, 需要编写线程安全的数据缓存代码, 因此在此次编写一个通用的泛型缓存类, 方便使用
 * @author derick.jin 2019-11-19 10:12:00
 * @version 1.0
 *
 * 对该缓存类添加了 过期时间特性 设置缓存的过期时间间隔, 不同 Key 加载后, 会在超过缓存时间后再次查询时过期 重新加载
 * @author derick.jin 2020-01-19 10:37:00
 * @version 1.1
 **/
@Slf4j
public class Cache<K, V> {

    private volatile Map<K, V> cacheData;
    /**
     * 缓存 Key 对应数据的过期时间, 超过该时间则应该重新加载数据
     */
    private volatile Map<K, Long> cacheExpiredAt = new ConcurrentHashMap<>();
    /**
     * 每次缓存加载数据后的过期时间 单位: 秒
     * 如果值为 0 表示数据只会初始加载一次 永不过期
     */
    private final long expiredSeconds;
    /**
     * 缓存提供商 返回结果会被缓存下来
     */
    private final CacheSupplier<K, V> cacheSupplier;

    public interface CacheSupplier<K, V> {
        V supply(K cacheKey);
    }

    public enum CacheType {
        // 读 仅同步更新
        READ_SYNCHRONOUS_UPDATE,
        // 读 可异步更新
        READ_CONCURRENT_UPDATE,
    }

    public Cache(CacheSupplier<K, V> supplier){
        this(CacheType.READ_SYNCHRONOUS_UPDATE, 0, supplier);
    }

    public Cache(long expiredSeconds, CacheSupplier<K, V> supplier){
        this(CacheType.READ_SYNCHRONOUS_UPDATE, expiredSeconds, supplier);
    }

    public Cache(CacheType cacheType, long expiredSeconds, CacheSupplier<K, V> cacheSupplier){
        this.expiredSeconds = expiredSeconds;
        this.cacheSupplier = cacheSupplier;
        switch (cacheType) {
            case READ_SYNCHRONOUS_UPDATE:
                cacheData = new HashMap<>();
                break;
            case READ_CONCURRENT_UPDATE:
                cacheData = new ConcurrentHashMap<>();
                break;
        }
    }

    /**
     * 使用此方法来获取缓存的值
     * 如果没有则调用 supplier 的 supply 方法并将其返回的值缓存 方便下次调用
     * 相比与 getSync 该方法的适合懒加载的情况, 并且懒加载的时候是线程安全的
     * 如多个线程同时执行该方法加载同一个 cacheKey, 只会有一个线程加载, 其它直接返回
     * @param cacheKey 缓存的键
     * @param supplier 临时的 缓存值提供者 默认会使用创建时设置的缓存提供者
     * @return 缓存的值
     */
    public V getOrCache(K cacheKey, CacheSupplier<K, V> supplier) {
        V data = cacheData.get(cacheKey);
        // 如果数据存在缓存值 且 设置了过期时间 则进行过期时间校验
        if (Objects.nonNull(data) && expiredSeconds != 0) {
            if (System.currentTimeMillis() > cacheExpiredAt.get(cacheKey)) {
                synchronized (this) {
                    if (System.currentTimeMillis() > cacheExpiredAt.get(cacheKey)) {
                        cacheData.remove(cacheKey);
                        data = null;
                    }
                }
            }
        }
        // 如果值为空则加载缓存数据
        if (Objects.isNull(data)) {
            synchronized (this) {
                if (Objects.isNull(data = cacheData.get(cacheKey))) {
                    data = supplier.supply(cacheKey);
                    cacheData.put(cacheKey, data);
                    // 如果设置了过期时间 则 设置过期时间戳
                    if (expiredSeconds != 0) {
                        cacheExpiredAt.put(cacheKey, System.currentTimeMillis() + expiredSeconds * 1000);
                    }
                }
            }
        }
        return data;
    }

    public V getOrCache(K cacheKey) {
        return getOrCache(cacheKey, cacheSupplier);
    }

    public synchronized V getSync(K cacheKey) {
        return cacheData.get(cacheKey);
    }

    public synchronized void setSync(K cacheKey, V data) {
        cacheData.put(cacheKey, data);
    }
}
```

## 测试代码
> 简单测试了下多个线程同时获取值 和 过期时间是否有效 
``` java
import com.cil.settle.CilSettleBootstrapApplication;
import com.cil.settle.entity.bo.common.Cache;
import lombok.extern.slf4j.Slf4j;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@Slf4j
@RunWith(SpringRunner.class)
@SpringBootTest(classes={CilSettleBootstrapApplication.class})
public class CacheTest {

    private final Cache<String, String> cache = new Cache<>(10, cacheKey -> {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("加载数据完成: {}", cacheKey);
        return "TEST_VALUE";
    });

    @Test
    public void doTest(){
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                log.info(cache.getOrCache("test"));
            }).start();
        }
        // 休息 12 秒
        try {
            Thread.sleep(12000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 再次尝试获取
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                log.info(cache.getOrCache("test"));
            }).start();
        }
        // 休息 10 秒
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```