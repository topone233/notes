## Redis支持的数据结构

| 类型                 | 简介                                                   | 特性                                                         | 应用场景                                                     |
| -------------------- | ------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| String(字符串)       | 二进制安全的字符出汗                                   | 可以包含任何数据,比如jpg图片或者序列化的对象,一个键最大能存储512M |                                                              |
| Hash(字典)           | 键值对集合                                             | 适合存储对象,并且可以像数据库中update一个属性一样只修改某一项属性值(Memcached中需要取出整个字符串反序列化成对象修改完再序列化存回去) | 存储、读取、修改用户属性；图片缓存                           |
| List(列表)           | 链表(双向链表)                                         | 增删快,提供了操作某一段元素的API                             | 1,最新消息排行等功能(比如朋友圈的时间线)； 2,消息队列        |
| Set(集合)            | 哈希表实现,元素不重复                                  | 1、添加、删除,查找的复杂度都是O(1) ； 2、为集合提供了求交集、并集、差集等操作 | 1、共同好友； 2、利用唯一性,统计访问网站的所有独立ip ；3、好友推荐时,根据tag求交集,大于某个阈值就可以推荐 |
| Sorted Set(有序集合) | 将Set中的元素增加一个权重参数score,元素按score有序排列 | 数据插入集合时,已经进行天然排序                              | 1、排行榜； 2、带权重的消息队列                              |

## StringRedisTemplate

### StringRedisTemplate与RedisTemplate

- StringRedisTemplate继承RedisTemplate。
- 其实他们两者之间的区别主要在于他们使用的序列化类:
  - RedisTemplate默认使用的是JdkSerializationRedisSerializer  存入数据会将数据先序列化成字节数组然后存入Redis数据库。 
  - StringRedisTemplate使用的是StringRedisSerializer
- 使用时注意事项：
  - 当你的redis数据库里面本来存的是字符串数据或者你要存取的数据就是字符串类型数据的时候，那么你就使用StringRedisTemplate即可。
  - 如果你的数据是复杂的对象类型，而取出的时候又不想做任何的数据转换，直接从Redis里面取出一个对象，那么使用RedisTemplate是更好的选择。
- RedisTemplate使用时常见问题：
  - redisTemplate 中存取数据都是字节数组。当redis中存入的数据是可读形式而非字节数组时，使用redisTemplate取值的时候会无法获取导出数据，获得的值为null。可以使用 StringRedisTemplate 试试。

### 通用方法

####  keys(pattern) => keys

```java
// 获取匹配表达式的所有key
// 注意，keys方法会阻塞redis，可以考虑用scan或sets
Set<K> keys(K pattern)
```

![image-20220330095919448](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330095919448.png)

####  expire() => expire、pExpire

```java
    public Boolean expire(K key, long timeout, TimeUnit unit) {
        byte[] rawKey = this.rawKey(key);
        long rawTimeout = TimeoutUtils.toMillis(timeout, unit);
        return (Boolean)this.execute((connection) -> {
            try {
                // 设置key的剩余存活时间（毫秒）
                return connection.pExpire(rawKey, rawTimeout);
            } catch (Exception var8) {
                // 设置key的剩余存活时间（秒）
                return connection.expire(rawKey, TimeoutUtils.toSeconds(timeout, unit));
            }
        }, true);
    }
```

#### getExpire() => ttl、pTtl

```java
    public Long getExpire(K key, TimeUnit timeUnit) {
        byte[] rawKey = this.rawKey(key);
        return (Long)this.execute((connection) -> {
            try {
                // 获取key的剩余存活时间（毫秒），默认是-1无限期，-2表示key不存在
                return connection.pTtl(rawKey, timeUnit);
            } catch (Exception var4) {
                // 获取key的剩余存活时间（秒），默认是-1无限期，-2表示key不存在
                return connection.ttl(rawKey, timeUnit);
            }
        }, true);
    }
```

![image-20220330100309559](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330100309559.png)

#### hasKey() => exists

```java
    public Boolean hasKey(K key) {
        byte[] rawKey = this.rawKey(key);
        return (Boolean)this.execute((connection) -> {
            // 判断key是否存在，1存在，0不存在
            return connection.exists(rawKey);
        }, true);
    }
```

![image-20220330093506752](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330093506752.png)

#### delete() => del

![image-20220330100629817](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330100629817.png)

这里能看到一个要注意的点，redis区分大小写。

### opsForValue()

#### set() => set、setEx、pSetEx

```java
    public void set(K key, V value) {
        final byte[] rawValue = this.rawValue(value);
        this.execute(new AbstractOperations<K, V>.ValueDeserializingRedisCallback(key) {
            protected byte[] inRedis(byte[] rawKey, RedisConnection connection) {
                // 插入键值对，默认过期时间是永不过期
                connection.set(rawKey, rawValue);
                return null;
            }
        }, true);
    }

    public void set(K key, V value, final long timeout, final TimeUnit unit) {
        final byte[] rawKey = this.rawKey(key);
        final byte[] rawValue = this.rawValue(value);
        this.execute(new RedisCallback<Object>() {
            public Object doInRedis(RedisConnection connection) throws DataAccessException {
                this.potentiallyUsePsetEx(connection);
                return null;
            }
            public void potentiallyUsePsetEx(RedisConnection connection) {
                if (!TimeUnit.MILLISECONDS.equals(unit) || !this.failsafeInvokePsetEx(connection)) {
                    // 插入键值对，并设置过期时间（秒）
                    connection.setEx(rawKey, TimeoutUtils.toSeconds(timeout, unit), rawValue);
                }
            }

            private boolean failsafeInvokePsetEx(RedisConnection connection) {
                boolean failed = false;
                try {
                    // 插入键值对，并设置过期时间（毫秒）
                    connection.pSetEx(rawKey, timeout, rawValue);
                } catch (UnsupportedOperationException var4) {
                    failed = true;
                }
                return !failed;
            }
        }, true);
    }
```

set可以跟参数

- ex 设置键key的过期时间，单位时秒
- px 设置键key的过期时间，单位时毫秒
- nx 只有键key不存在的时候才会设置key的值
- xx 只有键key存在的时候才会设置key的值
- get 返回 key 存储的值，如果 key 不存在返回空

![image-20220330101250372](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330101250372.png)

#### get() =>get

#### setIfAbsent()  => setNX

参考文档：http://www.redis.cn/commands/setnx.html

```java
	/**
     * 先get
     * 如果key不存在，则set
     * 如果存在，不进行任何操作
     */
	@Nullable
    public Boolean setIfAbsent(K key, V value) {
        byte[] rawKey = this.rawKey(key);
        byte[] rawValue = this.rawValue(value);
        return (Boolean)this.execute((connection) -> {
            // 调用Redis的setNX方法，SET if Not eXists 
            return connection.setNX(rawKey, rawValue);
        }, true);
    }

    public Boolean setIfAbsent(K key, V value, long timeout, TimeUnit unit) {
        byte[] rawKey = this.rawKey(key);
        byte[] rawValue = this.rawValue(value);
        Expiration expiration = Expiration.from(timeout, unit);
        return (Boolean)this.execute((connection) -> {
            return connection.set(rawKey, rawValue, expiration, SetOption.ifAbsent());
        }, true);
    }
```

#### getAndSet() => getSet

http://www.redis.cn/commands/getset.html

```java
	/**
     * 将key的值设为value，并返回旧的value（ 如果key存在但是对应的value不是字符串，就返回错误）
     * 与直接set修改value的区别在于，直接set不会返回旧的value。而且此命令是原子性操作
     */
	@Nullable
    public V getAndSet(K key, V newValue) {
        final byte[] rawValue = this.rawValue(newValue);
        return this.execute(new AbstractOperations<K, V>.ValueDeserializingRedisCallback(key) {
            protected byte[] inRedis(byte[] rawKey, RedisConnection connection) {
                // Redis的getSet方法
                return connection.getSet(rawKey, rawValue);
            }
        }, true);
    }
```

用处：RedisLock.lock

![image-20220330101742083](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330101742083.png)

#### increment() => incr、incrBy

```java
    public Long increment(K key) {
        byte[] rawKey = this.rawKey(key);
        return (Long)this.execute((connection) -> {
            // 对指定key的value自增
            // 如果key不存在，value初始化为0 再自增
            return connection.incr(rawKey);
        }, true);
    }

    public Long increment(K key, long delta) {
        byte[] rawKey = this.rawKey(key);
        return (Long)this.execute((connection) -> {
            // 对指定key的value增加指定值
            // 如果key不存在，value初始化为0 再增加
            return connection.incrBy(rawKey, delta);
        }, true);
    }

    public Double increment(K key, double delta) {
        byte[] rawKey = this.rawKey(key);
        return (Double)this.execute((connection) -> {
            return connection.incrBy(rawKey, delta);
        }, true);
    }
```

![image-20220330102239944](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330102239944.png)

#### decrement() => decr、decrBy

```java
    public Long decrement(K key) {
        byte[] rawKey = this.rawKey(key);
        return (Long)this.execute((connection) -> {
            // 对指定key的value自减
            // 如果key不存在，value初始化为0 再自减
            return connection.decr(rawKey);
        }, true);
    }

    public Long decrement(K key, long delta) {
        byte[] rawKey = this.rawKey(key);
        return (Long)this.execute((connection) -> {
            // 对指定key的value减去指定值
            // 如果key不存在，value初始化为0 在执行减法
            return connection.decrBy(rawKey, delta);
        }, true);
    }
```

### opsForHash()

#### put() => hSet

```java
    public void put(K key, HK hashKey, HV value) {
        byte[] rawKey = this.rawKey(key);
        byte[] rawHashKey = this.rawHashKey(hashKey);
        byte[] rawHashValue = this.rawHashValue(value);
        this.execute((connection) -> {
            connection.hSet(rawKey, rawHashKey, rawHashValue);
            return null;
        }, true);
    }
```

- 第一个参数是hash散列值，通过这个值定位到的是一个具体的HashMap

- 第二个参数就是HashMap的key

- 第三个参数就是对应的value

#### get() => hGet

```java
	@Nullable
    public HV get(K key, Object hashKey) {
        byte[] rawKey = this.rawKey(key);
        byte[] rawHashKey = this.rawHashKey(hashKey);
        byte[] rawHashValue = (byte[])this.execute((connection) -> {
            return connection.hGet(rawKey, rawHashKey);
        }, true);
        return rawHashValue != null ? this.deserializeHashValue(rawHashValue) : null;
    }
```

获取此hash位置中key所对应的value（允许为空）

![image-20220330105440567](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330105440567.png)

#### putAll() => hMSet

#### entries() => hGetAll

```java
    public Map<HK, HV> entries(K key) {
        byte[] rawKey = this.rawKey(key);
        Map<byte[], byte[]> entries = (Map)this.execute((connection) -> {
            // 获取hash所对应的键值对Map
            return connection.hGetAll(rawKey);
        }, true);
       
```

![image-20220330105354337](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330105354337.png)

#### size() => hLen

#### hasKey() => hExists

#### increment() => hIncrBy

```java
    public Long increment(K key, HK hashKey, long delta) {
        byte[] rawKey = this.rawKey(key);
        byte[] rawHashKey = this.rawHashKey(hashKey);
        return (Long)this.execute((connection) -> {
            return connection.hIncrBy(rawKey, rawHashKey, delta);
        }, true);
    }

    public Double increment(K key, HK hashKey, double delta) {
        byte[] rawKey = this.rawKey(key);
        byte[] rawHashKey = this.rawHashKey(hashKey);
        return (Double)this.execute((connection) -> {
            return connection.hIncrBy(rawKey, rawHashKey, delta);
        }, true);
    }
```

#### delete() => hDel

![image-20220330105522408](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330105522408.png)

### opsForList()

#### leftPush() => lPush

```java
    public Long leftPush(K key, V value) {
        byte[] rawKey = this.rawKey(key);
        byte[] rawValue = this.rawValue(value);
        return (Long)this.execute((connection) -> {
            // 插入到队列最左边（头部）
            return connection.lPush(rawKey, new byte[][]{rawValue});
        }, true);
    }
```

#### rightPush => rPush

```java
    public Long rightPush(K key, V value) {
        byte[] rawKey = this.rawKey(key);
        byte[] rawValue = this.rawValue(value);
        return (Long)this.execute((connection) -> {
            // 插入到队列最右边（尾部）
            return connection.rPush(rawKey, new byte[][]{rawValue});
        }, true);
    }
```

#### rightPushAll => rPush

```java
    public Long rightPushAll(K key, V... values) {
        byte[] rawKey = this.rawKey(key);
        byte[][] rawValues = this.rawValues(values);
        return (Long)this.execute((connection) -> {
            return connection.rPush(rawKey, rawValues);
        }, true);
    }

    public Long rightPushAll(K key, Collection<V> values) {
        byte[] rawKey = this.rawKey(key);
        byte[][] rawValues = this.rawValues(values);
        return (Long)this.execute((connection) -> {
            return connection.rPush(rawKey, rawValues);
        }, true);
    }
```

#### remove() => lRem

#### range() => lRange

```java
    public List<V> range(K key, long start, long end) {
        byte[] rawKey = this.rawKey(key);
        return (List)this.execute((connection) -> {
            // 返回指定下标范围内的元素，全闭区间
            // 0,-1可以查询全部。-1表示倒数第一个，-2倒数第二个，以此类推
            return this.deserializeValues(connection.lRange(rawKey, start, end));
        }, true);
    }
```

![image-20220330130944496](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330130944496.png)

### opsForSet()

#### add() => sAdd

>添加一个或多个指定的member元素到集合的 key中.
>
>如果集合key 不存在，则新建集合key,并添加member元素到集合key中.
>
>指定的一个或者多个元素member 如果已经在集合key中存在则忽略.

```java
    public Long add(K key, V... values) {
        byte[] rawKey = this.rawKey(key);
        byte[][] rawValues = this.rawValues((Object[])values);
        return (Long)this.execute((connection) -> {
            // 向set中添加一个或多个元素，已存在的元素将被忽略
            // 假如集合 key 不存在，则创建一个只包含被添加的元素作为成员的集合
            return connection.sAdd(rawKey, rawValues);
        }, true);
    }
```

#### members() => sMembers

```java
    public Set<V> members(K key) {
        byte[] rawKey = this.rawKey(key);
        Set<byte[]> rawValues = (Set)this.execute((connection) -> {
            // 返回set中的所有元素
            return connection.sMembers(rawKey);
        }, true);
        return this.deserializeValues(rawValues);
    }
```

![image-20220330103201276](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330103201276.png)

#### remove() => sRem

![image-20220330103436452](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330103436452.png)

### opsForHyperLogLog()

Redis HyperLogLog 是用来做基数统计的算法

- 基数：比如数据集 {1, 3, 5, 7, 5, 7, 8}， 那么这个数据集的基数集为 {1, 3, 5 ,7, 8}, 基数(不重复元素)为5
- 优点：在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。
- 缺点：只会根据输入元素来计算基数，而不会储存输入元素本身，不能像集合那样，返回输入的各个元素

#### add() => pfAdd

#### size() =>  pfCount

#### delete() => del

![image-20220330105802972](C:\Users\yuki\AppData\Roaming\Typora\typora-user-images\image-20220330105802972.png)



## 分布式锁

参考文章：

https://blog.csdn.net/asd051377305/article/details/108384490

https://blog.csdn.net/xiaolegeaizy/article/details/109803806

### 意义

分布式架构下，控制多个进程对同一资源的**互斥**访问，**保证数据的最终一致性**

### 代码实现

```java
    /**
     * 加锁
     * 1.setIfAbsent如果锁不存在，则加锁，获取锁成功
     * 2.如果锁已经存在，判断是否超时未释放
     *   1）getAndSet同时只能被一个线程执行，第一个执行的线程可以获得锁
     * @param key  唯一标志
     * @param value  当前时间+超时时间 也就是时间戳
     * @return boolean
     */
    public boolean lock(String key,String value){
        if(redisTemplate.opsForValue().setIfAbsent(key,value)){
            //可以成功设置,也就是key不存在
            return true;
        }
        //判断锁超时 - 防止原来的操作异常，没有运行解锁操作  防止死锁
        String currentValue = redisTemplate.opsForValue().get(key);
        //如果锁过期，currentValue不为空且小于当前时间
        if(!StringUtils.isEmpty(currentValue) && Long.parseLong(currentValue) < System.currentTimeMillis()){
            //获取上一个锁的时间value
            // getAndSet同时只能被一个线程执行，第一个执行的线程可以获得锁
            String oldValue =redisTemplate.opsForValue().getAndSet(key,value);

            //假设两个线程同时进来这里，因为key被占用了，而且锁过期了。获取的值currentValue=A(get取的旧的值肯定是一样的),两个线程的value都是B,key都是K.锁时间已经过期了。
            //而这里面的getAndSet一次只会一个执行，也就是一个执行之后，上一个的value已经变成了B。只有一个线程获取的上一个值会是A，另一个线程拿到的值是B。
            //oldValue不为空且oldValue等于currentValue，也就是校验是不是上个对应的商品时间戳，也是防止并发
            return !StringUtils.isEmpty(oldValue) && oldValue.equals(currentValue);
        }
        return false;
    }

    /**
     * 解锁
     * @param key key
     * @param value value
     */
    public void unlock(String key,String value){
        try {
            String currentValue = redisTemplate.opsForValue().get(key);
            if(!StringUtils.isEmpty(currentValue) && currentValue.equals(value) ){
                //删除key
                redisTemplate.opsForValue().getOperations().delete(key);
            }
        } catch (Exception e) {
            log.error("[Redis分布式锁] 解锁出现异常了，{}",e);
        }
    }
```

### 应用场景

- 新增话题
- 马甲根据热度执行指令消息发送

## 应用场景

### 利用Redis的Geo功能实现查找附近的位置

https://www.cnblogs.com/felordcn/p/13162188.html