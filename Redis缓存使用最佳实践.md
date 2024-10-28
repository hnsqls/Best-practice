# 缓存穿透问题解决-缓存空值

访问数据库不存在的数据，会一直请求到数据库，被别有用心的人使用，可能会一直请求数据库，导致数据库宕机。解决方法有两

一：缓存空数据，二，使用布隆过滤器进行校验。

> 缓存空数据

在数据库查询到不存在的数据时，对该数据进行缓存为空（可以设置稍短的3~5分钟的TTL），之后相同的请求，就会在缓存中查到，而不去请求数据库。

代码案列

```java
  /**
     * 查询商户信息
     * @param id
     * @return
     */
    @Override
    public Result queryById(Long id) {
        //查询缓存
        String string = stringRedisTemplate.opsForValue().get(CACHE_SHOP_KEY+id);
        //hutool 工具类 符合条件“adc" 不符合条件“”，null, "/t/n"
        if (StrUtil.isNotBlank(string)){
            Shop shop = JSONUtil.toBean(string, Shop.class);
            return Result.ok(shop);
        }
        //若是 " " 上面已经判断了不是“” 不是null ,
        if(string != null){
            return Result.fail("商户不存在");
        }

        // 缓存不存在 查数据库
        Shop shop = getById(id);
        if (shop ==null) {
            //将空值写入缓存
            stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY + id, "", CACHE_NULL_TTL, TimeUnit.MINUTES);
            return Result.fail("商户不存在");
        }

        //写入缓存
        stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY+ id, JSONUtil.toJsonStr(shop), CACHE_SHOP_TTL, TimeUnit.MINUTES);

        return Result.ok(shop);
    }
```

# 缓存击穿问题- 互斥锁

热点key失效，构造缓存复杂，在构造缓存的期间大量请求，只允许一个请求到数据库构造缓存。

具体流程

假设现在线程1过来访问，他查询缓存没有命中，但是此时他获得到了锁的资源，那么线程1就会一个人去执行逻辑，假设现在线程2过来，线程2在执行过程中，并没有获得到锁，那么线程2就重试获取缓存资源和锁（递归），直到线程1把锁释放后，线程2获得到锁或者缓存资源，可能线程二执行到获取缓存就获得到缓存就之间返回了，也可能没查到缓存，执行到获得了锁，这时候要再次校验一下是否获得了缓存。没有获得缓存在取构建缓存。



```java
 /**
     * 获取锁
     * @param key
     * @return
     */
    private boolean tryLock(String key){
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 20, TimeUnit.SECONDS);
        return BooleanUtil.isTrue(flag);

    }

    /**
     * 释放锁
     * @param key
     */
    private  void unlock(String key){
        stringRedisTemplate.delete(key);
    }
```



```java
/**
     * 查询商户信息 缓存击穿互斥锁
     * @param id
     * @return
     */
    public Shop queryWithMutex(Long id){
        String shopKey = CACHE_SHOP_KEY+ id;

        // 1. 从redis中查询店铺缓存
        String shopJson = stringRedisTemplate.opsForValue().get(shopKey);

        //2.判断是否命中缓存  isnotblank false: "" or "/t/n" or "null"
        if(StrUtil.isNotBlank(shopJson)){
            // 3.若命中则返回信息
            Shop shop = JSONUtil.toBean(shopJson, Shop.class);
            return shop;
        }
        //数据穿透判空   不是null 就是空串 ""
        if (shopJson != null){
            //返回错误信息
//            return  Result.fail("没有该商户信息（缓存）");
            return null;
        }
        //4.没有命中缓存，查数据库
        //todo :解决缓存击穿  不能直接查数据库。 利用互斥锁解决

        /**
         * 实现缓存重建
         * 1. 获取互斥锁
         * 2. 判断是否成功
         * 3. 失败就休眠重试
         * 4.成功 查数据库
         * 5 数据库存在该数据写入缓存
         * 6 不存在返回错误信息并写入缓存“”
         * 7 释放锁
         *
         */

        //获取互斥锁 失败  休眠重试
        String lockKey = "lock:shop" + id;
        Shop shop=null;

        try {
            boolean isLock = tryLock(lockKey);
            //获取锁失败
            if (!isLock) {

                System.out.println("获取锁失败，重试");
                Thread.sleep(50);
                return queryWithMutex(id);//递归 重试
            }

            // 获取锁成功，再次检测缓存是否存在，存在就无需构建缓存，因为可能有的线程刚构建好缓存并释放锁，其他线程获取了锁
            //检测缓存是否存在  存在
            shopJson = stringRedisTemplate.opsForValue().get(shopKey);
            if (StrUtil.isNotBlank(shopJson)) {
                return JSONUtil.toBean(shopJson, Shop.class);
            }
            if (shopJson !=null){
                return null;
            }
            // 缓存不存在
            // 查数据库
             shop = super.getById(id);
            Thread.sleep(200);//模拟你测试环境 热点key失效模拟重建延迟
            if (shop == null){
                //没有该商户信息
                stringRedisTemplate.opsForValue().set(shopKey,"",CACHE_NULL_TTL,TimeUnit.SECONDS);
                return null;
            }
            //有该商户信息
            stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY+ id, JSONUtil.toJsonStr(shop),CACHE_SHOP_TTL, TimeUnit.MINUTES);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            unlock(lockKey);
        }
        return shop;

    }
```

# 缓存击穿问题- 逻辑过期时间

需要添加逻辑过期时间字段，直接在shop类中添加不太友好改了源代码

可以新建一个类

```java
/**
 * 逻辑过期类
 */
@Data
public class RedisData {
    private LocalDateTime expireTime;
    private Object data;
}
```

数据预热

```java
 /**
     * 添加逻辑过期时间
     * @param id
     * @param expireSeconds
     */
    public void savaShop2Redis(Long id ,Long expireSeconds){

        // 查询店铺数据
        Shop shop = getById(id);

        //封装逻辑过期时间
        RedisData redisData = new RedisData();
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(expireSeconds));
        redisData.setData(shop);
        //写入redis
        stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY+id,JSONUtil.toJsonStr(redisData));
    }
```

执行测试方法即可加入到redis。

正式代码

```java
private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);
public Shop queryWithLogicalExpire( Long id ) {
    String key = CACHE_SHOP_KEY + id;
    // 1.从redis查询商铺缓存
    String json = stringRedisTemplate.opsForValue().get(key);
    // 2.判断是否存在
    if (StrUtil.isBlank(json)) {
        // 3.存在，直接返回
        return null;
    }
    // 4.命中，需要先把json反序列化为对象
    RedisData redisData = JSONUtil.toBean(json, RedisData.class);
    Shop shop = JSONUtil.toBean((JSONObject) redisData.getData(), Shop.class);
    LocalDateTime expireTime = redisData.getExpireTime();
    // 5.判断是否过期
    if(expireTime.isAfter(LocalDateTime.now())) {
        // 5.1.未过期，直接返回店铺信息
        return shop;
    }
    // 5.2.已过期，需要缓存重建
    // 6.缓存重建
    // 6.1.获取互斥锁
    String lockKey = LOCK_SHOP_KEY + id;
    boolean isLock = tryLock(lockKey);
    // 6.2.判断是否获取锁成功
    if (isLock){
        CACHE_REBUILD_EXECUTOR.submit( ()->{

            try{
                //重建缓存
                this.saveShop2Redis(id,20L);
            }catch (Exception e){
                throw new RuntimeException(e);
            }finally {
                unlock(lockKey);
            }
        });
    }
    // 6.4.返回过期的商铺信息
    return shop;
}
```



# 缓存工具类

```java
package com.hmdp.utils;

import cn.hutool.core.lang.func.Func;
import cn.hutool.core.util.BooleanUtil;
import cn.hutool.core.util.StrUtil;
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;
import com.hmdp.entity.Shop;
import javafx.beans.binding.ObjectExpression;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.function.Function;

import static com.hmdp.utils.RedisConstants.*;

/**
 * Redis 工具类
 * * 方法1：将任意Java对象序列化为json并存储在string类型的key中，并且可以设置TTL过期时间
 * * 方法2：将任意Java对象序列化为json并存储在string类型的key中，并且可以设置逻辑过期时间，用于处理缓存击穿问题
 *
 * * 方法3：根据指定的key查询缓存，并反序列化为指定类型，利用缓存空值的方式解决缓存穿透问题
 * * 方法4：根据指定的key查询缓存，并反序列化为指定类型，需要利用逻辑过期解决缓存击穿问题
 */
@Component
public class CacheClient {

    private  final StringRedisTemplate stringRedisTemplate;

    public CacheClient(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    //range

    /**
     * 方法1：将任意Java对象序列化为json并存储在string类型的key中，并且可以设置TTL过期时间
     * @param key
     * @param value
     * @param time
     * @param unit
     */
    public void set(String key , Object value, Long time, TimeUnit unit){
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(value), time,unit);
    }


    /**
     * 方法2：将任意Java对象序列化为json并存储在string类型的key中，并且可以设置逻辑过期时间，用于处理缓存击穿问题
     * @param key  redis的key
     * @param value
     * @param time
     * @param unit
     */
    public void setWithLogicalExpire(String key , Object value, Long time, TimeUnit unit){

        //RedisData 是自定义类
        RedisData redisData = new RedisData();
        redisData.setData(value);
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));
    }


    /**
     *  方法3：根据指定的key查询缓存，并反序列化为指定类型，利用缓存空值的方式解决缓存穿透问题
     * @param
     * @param id
     * @return
     */
    public <R,ID> R queryWithPassThrough(String keyPrefix, ID id, Class<R> type, Function<ID,R> dbFallback,Long time,TimeUnit unit){
        String key = keyPrefix+ id;

        // 1. 从redis中查询店铺缓存
        String json = stringRedisTemplate.opsForValue().get(key);

        //2.判断是否命中缓存  isnotblank false: "" or "/t/n" or "null"
        if(StrUtil.isNotBlank(json)){
            // 3.若命中则返回信息
            R r = JSONUtil.toBean(json, type);
            //            return Result.fail("没有该商户信息");
            return r;
        }
        //数据穿透判空   不是null 就是空串 ""
        if (json != null){
            return null;
        }
        //4.没有命中缓存，查数据库，因为不知道操作那个库，函数式编程，逻辑交给调用者完成
//       R r= getById(id); 交给调用者--》》函数式编程
        R r = dbFallback.apply(id);
        //5. 数据库为空，返回错误---》解决缓存穿透--》加入redis为空
        if (r == null){
            stringRedisTemplate.opsForValue().set(key,"",CACHE_NULL_TTL,TimeUnit.MINUTES);
//            return Result.fail("没有该商户信息");
            return null;
        }

        //6. 数据库不为空，返回查询的结果并加入缓存
        stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(r),time, unit);
        return r;
    }

    /**
     *  方法4：根据指定的key查询缓存，并反序列化为指定类型，需要利用逻辑过期解决缓存击穿问题
     * @param id
     * @return
     */
    public <R,ID> R queryWithLogicalExpire(String keyPrefix,ID id,Class<R> type,Function<ID,R>dbFallback,String lockPrefix,Long time,TimeUnit unit){
        String key = keyPrefix+ id;

        // 1. 从redis中查询店铺缓存
        String json = stringRedisTemplate.opsForValue().get(key);

        //2.判断数据是否存在（我们对于热点key设置永不过期）  isblank
        if(StrUtil.isBlank(json)){
            // 3.若未命中中则返回空
            return null;
        }

        //4.若命中缓存 判断是否过期
        RedisData redisData = JSONUtil.toBean(json, RedisData.class);
        JSONObject data = (JSONObject) redisData.getData();
        R r = JSONUtil.toBean(data, type);
        LocalDateTime expireTime = redisData.getExpireTime();

        //未过期 直接返回查询信息
        if (expireTime.isAfter(LocalDateTime.now())){
            return r;
        }
        //过期
        // 重建缓存
        // 获取锁
        String lockKey = lockPrefix + id;
        if (tryLock(lockKey)) {
            //再次校验缓存是否未过期（线程1刚写入缓存然后释放锁，线程2在线程1释放锁的同时，执行到获得锁）
            //  从redis中查询店铺缓存
            json = stringRedisTemplate.opsForValue().get(key);

            //2.判断数据是否存在（我们对于热点key设置永不过期）  isblank
            if(StrUtil.isBlank(json)){
                // 3.若未命中中则返回空
                return null;
            }

            //4.若命中缓存 判断是否过期
            redisData = JSONUtil.toBean(json, RedisData.class);
            data = (JSONObject) redisData.getData();
            r = JSONUtil.toBean(data, type);
            expireTime = redisData.getExpireTime();

            //未过期 直接返回查询信息
            if (expireTime.isAfter(LocalDateTime.now())){
                return r;
            }

            //二次校验过后还时过期的就新开线程重构缓存


            // 获得锁,开启新线程，重构缓存 ，老线程直接返回过期信息
            CACHE_REBUILD_EXECUTOR.submit( ()->{

                try{
                    //重建缓存
                    //先查数据库 封装逻辑过期时间 再写redis
                    R r1 = dbFallback.apply(id);

                    this.setWithLogicalExpire(key, r1, time, unit);


                }catch (Exception e){
                    throw new RuntimeException(e);
                }finally {
                    unlock(lockKey);
                }
            });

        }
        //未获得锁 直接返回无效信息
        return r;
    }

    /**缓存穿透互斥锁解
     *
     * @param keyPrefix
     * @param id
     * @param type
     * @param dbFallback
     * @param time
     * @param unit
     * @return
     */
    public <R,ID>  R queryMutex(String keyPrefix, ID id, Class<R> type, Function<ID,R>dbFallback,String lockPrefix, Long time, TimeUnit unit) {
        String key = keyPrefix + id;
        //1.从redis中查询店铺缓存
        String json = stringRedisTemplate.opsForValue().get(key);
        //2.判断数据是否存在缓存
        if (StrUtil.isNotBlank(json)) {
            //2.1存在缓存
            R r = JSONUtil.toBean(json, type);
            return r;
        }
        //  2.2 是否缓存“”
        //判断命中是否为空值  ""
        if (json != null) {
            return null;
        }
        // 2.3不存在缓存
        // 3 缓存重建
        // 3.1 获取互斥锁
        String lockKey = lockPrefix + id;
        R r = null;
        try {
        boolean isLock = tryLock(lockKey);
        // 成功获取锁 - 》查数据库缓存重建
        if (isLock) {
            //二次校验 缓存是否有值
            json = stringRedisTemplate.opsForValue().get(key);
            //判断缓存是否存在
            if (StrUtil.isNotBlank(json)) {
                //存在缓存
                r = JSONUtil.toBean(json, type);
                return r;
            }
            if (json != null) {
                //缓存为 ""
                return null;
            }
            // 缓存不存在--》 查询数据库


            //  查询数据库
            r = dbFallback.apply(id);
            if (r == null) {
                //缓存空值
                stringRedisTemplate.opsForValue().set(key, "", time, unit);
            }
            //缓存重建
            stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(r), time, unit);
            //返回数据
            return r;
        }
        // 3.2 获取锁失败 -》休眠重试
            //休眠
            Thread.sleep(50);
            // 递归重试
            return queryMutex(keyPrefix, id, type, dbFallback, lockPrefix, time, unit);
        }
        catch (InterruptedException e) {
            throw new RuntimeException(e);
        }finally {
            unlock(lockKey);
        }
    }



    //endrange
    /**
     * 线程池
     */
    private  static  final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);

    /**
     * 获取所
     * @param key
     * @return
     */
    private boolean tryLock(String key){
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, "1", 10, TimeUnit.SECONDS);
        return BooleanUtil.isTrue(flag);

    }

    /**
     * 释放锁
     * @param key
     */
    private  void unlock(String key){
        stringRedisTemplate.delete(key);
    }
}

```



