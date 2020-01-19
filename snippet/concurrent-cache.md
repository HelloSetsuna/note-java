# 线程安全 带过期时间 数据缓存工具类 实现 



```java
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

