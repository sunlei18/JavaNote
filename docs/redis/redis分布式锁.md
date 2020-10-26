
分布式锁的实现一般有三种：

1.数据库乐观锁

2.基于redis的分布式锁

3.基于zk的分布式锁

保证分布式锁高可用必须满足以下条件：

1.互斥性。任何情况下只有一个客户端能持有锁。

2.不会发生死锁。即使有一个客户端在持有锁的时间发生崩溃而没有主动解锁，也能保证后续其他客户端能加锁。

3.具有容错性。只要redis节点正常运行，客户端就可以加锁和解锁。

4.加锁和解锁必须是同一个客户端，客户端自己不能把其他客户端的锁解了。

## 一.业务使用分布式锁

 使用分布式锁的场景一般发生在多线程去执行段业务逻辑，锁的细粒度越细越好。主要逻辑为：设置一个全局唯一key，尝试加锁，加锁成功执行业务逻辑，加锁失败直接返回，无论加锁成功还是失败都执行解锁操作。加锁成功的线程执行解锁操作时，能正确解锁；加锁失败的线程执行解锁操作时，是没有锁可以解的，这是设置的每个线程加锁的value值不同保证的。
 ```java
public class Demo {
    public void  redisHandle(){
        //处理多台机器重复执行的问题
        String lockKey = DateFormatUtils.format(new Date(), "yyyyMMddHHmm");
        try{
            if(!redisLock.lock(lockKey,0)){
                log.info("can not get a lockKey:"+lockKey);
                return;
            }
            //业务逻辑
 
        }catch(Exception e){
            e.printStackTrace();
        }finally {
            redisLock.unlock();
        }
    }
}
 ```

## 二.旧版本代码

1.代码逻辑

（1）通过setNX()方法尝试加锁，如果写入成功，则返回加锁成功，它这里是将过期时间作为了value写入redis；

（2）如果写入失败，则获取锁的过期时间，和当前时间进行比较，如果锁已经过期，则设置新的过期时间，返回加锁成功；

（3）如果锁没有过期，则sleep一个delay时间，再进行重试加锁。

2.代码问题

（1）过期时间是客户端自己来保证的，则对客户端的时间同步性要求比较高，如果不能满足时间同步性，则不能保证分布式锁的公平性，时间跑的快的永远能拿到锁，时间跑的慢的永远拿到不锁；

（2）当锁过期的时候，多个客户端同时执行getSet()方法，虽然最终只有一个客户端可以加锁，但是这个客户端的锁的过期时间可能被其他客户端覆盖

（3）锁不具备拥有者标识，即任何客户端都可以解锁；

（4）解锁sleep耗费性能。

```java
public class RedisLock {
 
    private static Logger logger = LoggerFactory.getLogger(RedisLock.class);
 
    @Autowired
    @Qualifier("securityRedisCache")
    private IRedisCache<String,String> redisCache;
 
    private static final int DEFAULT_ACQUIRY_RESOLUTION_MILLIS = 100;
 
    private String lockKey;
 
    private int expireMsecs = 60 * 1000;
 
    private int timeoutMsecs = 10 * 1000;
 
    private volatile boolean locked = false;
 
    public RedisLock(){
 
    }
 
    public RedisLock(String lockKey) {
        this.lockKey = lockKey;
    }
 
    public RedisLock(String lockKey, int timeoutMsecs) {
        this(lockKey);
        this.timeoutMsecs = timeoutMsecs;
    }
 
    public RedisLock(String lockKey, int timeoutMsecs, int expireMsecs) {
        this(lockKey, timeoutMsecs);
        this.expireMsecs = expireMsecs;
    }
 
    public String getLockKey() {
        return lockKey;
    }
 
    private String get(final String key) {
        Object obj = null;
        try {
            obj = redisCache.getRedisTemplate().execute(new RedisCallback<Object>() {
                @Override
                public Object doInRedis(RedisConnection connection) throws DataAccessException {
                    StringRedisSerializer serializer = new StringRedisSerializer();
                    byte[] data = connection.get(serializer.serialize(key));
                    connection.close();
                    if (data == null) {
                        return null;
                    }
                    return serializer.deserialize(data);
                }
            });
        } catch (Exception e) {
            logger.error("get redis error, key : {}", key);
        }
        return obj != null ? obj.toString() : null;
    }
 
    private boolean setNX(final String key, final String value) {
        Object obj = null;
        try {
            obj = redisCache.getRedisTemplate().execute(new RedisCallback<Object>() {
                @Override
                public Object doInRedis(RedisConnection connection) throws DataAccessException {
                    StringRedisSerializer serializer = new StringRedisSerializer();
                    Boolean success = connection.setNX(serializer.serialize(key), serializer.serialize(value));
                    connection.close();
                    return success;
                }
            });
        } catch (Exception e) {
            logger.error("setNX redis error, key : {}", key);
        }
        return obj != null ? (Boolean) obj : false;
    }
 
    private String getSet(final String key, final String value) {
        Object obj = null;
        try {
            obj = redisCache.getRedisTemplate().execute(new RedisCallback<Object>() {
                @Override
                public Object doInRedis(RedisConnection connection) throws DataAccessException {
                    StringRedisSerializer serializer = new StringRedisSerializer();
                    byte[] ret = connection.getSet(serializer.serialize(key), serializer.serialize(value));
                    connection.close();
                    return serializer.deserialize(ret);
                }
            });
        } catch (Exception e) {
            logger.error("setNX redis error, key : {}", key);
        }
        return obj != null ? (String) obj : null;
    }
 
    public synchronized boolean lock(String lockKey,int timeoutMsecs) throws InterruptedException {
        this.lockKey = lockKey;
        this.timeoutMsecs = timeoutMsecs;
        int timeout = timeoutMsecs;
        while (timeout >= 0) {
            long expires = System.currentTimeMillis() + expireMsecs + 1;
            String expiresStr = String.valueOf(expires);
            if (this.setNX(lockKey, expiresStr)) {
                locked = true;
                return true;
            }
 
            String currentValueStr = this.get(lockKey); //redis里的时间
 
 
            // 判断是否为空, redis旧锁是否已经过期, 如果被其他线程设置了值, 则第二个条件判断是过不去的
            if (currentValueStr != null && Long.parseLong(currentValueStr) < System.currentTimeMillis()) {
 
                String oldValueStr = this.getSet(lockKey, expiresStr);
 
                // 获取上一个锁到期时间, 并设置现在的锁到期时间
                // 如果这个时候, 多个线程恰好都到了这里
                // 只有一个线程拿到的过期时间是小于当前时间的，后续的线程set进去过期时间但拿到的过期时间会大于当前时间
                // 只有一个线程的设置值和当前值相同, 那么它才有权利获取锁，其余线程继续等待
                if (oldValueStr != null && oldValueStr.equals(currentValueStr)) {
                    locked = true;
                    return true;
                }
            }
            timeout -= DEFAULT_ACQUIRY_RESOLUTION_MILLIS;
 
            Thread.sleep(DEFAULT_ACQUIRY_RESOLUTION_MILLIS);
 
        }
        return false;
    }
 
 
    public synchronized void unlock() {
        try {
            if (locked) {
                Thread.sleep(2000);
                redisCache.getRedisTemplate().delete(lockKey);
                locked = false;
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
     
}
```

这里的locked被volatile关键字修饰，保证变量的可见性，被volatile关键字修饰的变量，如果值发生了变更，其他线程立马可见，避免出现脏读的现象。

## 三.新版本代码
Lua脚本保证了加锁和解锁的原子性。加锁的时候，加锁和设置锁过期时间两步操作原子性，解锁的时候，判断锁标识和解锁两步操作原子性。

保证加锁原子性：

（1）setnx方法

（2）lua脚本

（3）redis事务

问题：没有实现锁可重入性。