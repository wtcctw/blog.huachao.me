title:  "Redis基础"
date: 2015-06-10
updated: 2015-12-26
categories:
- 后端
tags:
- redis
- db
---

> 凡是用memcached的场景，麻烦都用Redis代替

## Redis String/List/Set/Sorted-Set/Hash/

- Redis相比较memcached，能支持更多的数据结构，比如List，Set等
- Redis的Key在命名的时候，用`:`分割
- Redis同样支持二进制的存储Object `byte[]`
- Redis有一个Jedis的Java Client，其中Jedis Pool是类似于c3p0的东西，使用了commons-pool
- Jedis支持pipeline的操作，一般对多个Key操作，会使用pipeline

## 一个Java Redis客户端，基于Jedis
``` java
public class MyRedisClient {
    private static Logger logger = LoggerFactory.getLogger(MyRedisClient.class);
    private static final String CAT_NAME = "RedisCall";

    @Resource
    private JedisPool jedisPool;

    //************************** String ********************************//
    public List<Long> incr(final String... keys) {
        List<Long> emptyResult = new ArrayList<Long>();
        if (keys == null || keys.length == 0) {
            return emptyResult;
        }

        try {
            List<Long> results = JedisHelper.doJedisOperation(new JedisCallback<List<Long>>() {
                @Override
                public List<Long> doWithJedis(Jedis jedis) {
                    Pipeline pipelined = jedis.pipelined();
                    for (String key : keys) {
                        pipelined.incr(key);
                    }
                    List<Object> resultObjects = pipelined.syncAndReturnAll();
                    List<Long> resultList = new ArrayList<Long>();
                    for (Object resultObject : resultObjects) {
                        try {
                            resultList.add(Long.valueOf(resultObject.toString()));
                        } catch (Exception e) {
                            resultList.add(0L);
                            logger.error("redis incr result:convert object to long error", e);
                        }
                    }
                    return resultList;
                }
            }, this.jedisPool);
            return results;
        } catch (Exception e) {
            logger.error("call incr error : ", e);
        }
        return emptyResult;
    }

    public List<Long> decr(final String... keys) {
        List<Long> emptyResult = new ArrayList<Long>();
        if (keys == null || keys.length == 0) {
            return emptyResult;
        }

        try {
            List<Long> results = JedisHelper.doJedisOperation(new JedisCallback<List<Long>>() {
                @Override
                public List<Long> doWithJedis(Jedis jedis) {
                    Pipeline pipelined = jedis.pipelined();
                    for (String key : keys) {
                        pipelined.decr(key);
                    }
                    List<Object> resultObjects = pipelined.syncAndReturnAll();
                    List<Long> resultList = new ArrayList<Long>();
                    for (Object resultObject : resultObjects) {
                        try {
                            resultList.add(Long.valueOf(resultObject.toString()));
                        } catch (Exception e) {
                            resultList.add(0L);
                            logger.error("redis decr result:convert object to long error", e);
                        }
                    }
                    return resultList;
                }
            }, this.jedisPool);
            return results;
        } catch (Exception e) {
            logger.error("call decr error : ", e);
        }
        return emptyResult;
    }

    public Long expire(final String key, final int seconds) {
        if (StringUtils.isEmpty(key) || seconds <= 0) {
            return -1L;
        }

        try {
            Long result = JedisHelper.doJedisOperation(new JedisCallback<Long>() {
                @Override
                public Long doWithJedis(Jedis jedis) {
                    return jedis.expire(key, seconds);
                }
            }, this.jedisPool);
            return result;
        } catch (Exception e) {
            logger.error("call expire error : ", e);
        }
        return -1L;
    }

    public List<Long> expire(final List<RedisExpireDTO> expires) {
        List<Long> emptyResult = new ArrayList<Long>();
        if (CollectionUtils.isEmpty(expires)) {
            return emptyResult;
        }

        try {
            List<Long> result = JedisHelper.doJedisOperation(new JedisCallback<List<Long>>() {
                @Override
                public List<Long> doWithJedis(Jedis jedis) {
                    Pipeline pipelined = jedis.pipelined();
                    for (RedisExpireDTO expire : expires) {
                        pipelined.expire(expire.getKey(), expire.getSeconds());
                    }
                    List<Object> resultObjects = pipelined.syncAndReturnAll();
                    List<Long> resultList = new ArrayList<Long>();
                    if (CollectionUtils.isNotEmpty(resultObjects)) {
                        for (Object resultObject : resultObjects) {
                            try {
                                resultList.add(Long.valueOf(resultObject.toString()));
                            } catch (Exception e) {
                                resultList.add(-1L);
                                logger.error("redis expire result:convert object to long error", e);
                            }
                        }
                    }
                    return resultList;
                }
            }, this.jedisPool);
            return result;
        } catch (Exception e) {
            logger.error("call expires error : ", e);
        }
        return emptyResult;
    }

    public List<String> get(final String... keys) {
        List<String> emptyResult = new ArrayList<String>();
        if (keys == null || keys.length == 0) {
            return emptyResult;
        }

        try {
            List<String> results = JedisHelper.doJedisOperation(new JedisCallback<List<String>>() {
                @Override
                public List<String> doWithJedis(Jedis jedis) {
                    return jedis.mget(keys);
                }
            }, this.jedisPool);
            return results;
        } catch (Exception e) {
            logger.error("call get error : ", e);
        }
        return emptyResult;
    }

    public List<byte[]> get(final byte[]... keys) {
        List<byte[]> emptyResult = new ArrayList<byte[]>();
        if (keys == null || keys.length == 0) {
            return emptyResult;
        }

        try {
            List<byte[]> results = JedisHelper.doJedisOperation(new JedisCallback<List<byte[]>>() {
                @Override
                public List<byte[]> doWithJedis(Jedis jedis) {
                    return jedis.mget(keys);
                }
            }, this.jedisPool);
            return results;
        } catch (Exception e) {
            logger.error("call get error : ", e);
        }
        return emptyResult;
    }

    //************************** List ********************************//

    //************************** Set ********************************//
    public Long sAdd(final String key, final String... members) {
        if (StringUtils.isEmpty(key) || members == null || members.length == 0) {
            return -1L;
        }

        try {
            Long result = JedisHelper.doJedisOperation(new JedisCallback<Long>() {
                @Override
                public Long doWithJedis(Jedis jedis) {
                    return jedis.sadd(key, members);
                }
            }, this.jedisPool);
            return result;
        } catch (Exception e) {
            logger.error("call sadd error : ", e);
        }
        return -1L;
    }

    public boolean sIsMember(final String key, final String member) {
        if (StringUtils.isEmpty(key) || StringUtils.isEmpty(member)) {
            return false;
        }
        try {
            Boolean isMember = JedisHelper.doJedisOperation(new JedisCallback<Boolean>() {
                @Override
                public Boolean doWithJedis(Jedis jedis) {
                    return jedis.sismember(key, member);
                }
            }, this.jedisPool);
            return isMember;
        } catch (Exception e) {
            logger.error("call sIsMember error : ", e);
        }
        return false;
    }

    public Set<String> sMembers(final String key) {
        Set<String> ms = null;
        if (StringUtils.isEmpty(key)) {
            return ms;
        }
        try {
            ms = JedisHelper.doJedisOperation(new JedisCallback<Set<String>>() {
                @Override
                public Set<String> doWithJedis(Jedis jedis) {
                    return jedis.smembers(key);
                }
            }, this.jedisPool);
        } catch (Exception e) {
            logger.error("call smemebers error : ", e);
        }
        return ms;
    }

    //************************** Sorted Set ********************************//

    public List<Object> zAdd(final String key, final List<RedisTupleDTO> members) {
        List<Object> emptyResult = new LinkedList<Object>();
        if (StringUtils.isEmpty(key)) {
            return emptyResult;
        }
        if (CollectionUtils.isEmpty(members)) {
            return emptyResult;
        }

        String op = "zadd:pipeline:(RedisKey)";
        try {
            List<Object> results = JedisHelper.doJedisOperation(new JedisCallback<List<Object>>() {
                @Override
                public List<Object> doWithJedis(Jedis jedis) {
                    Pipeline pipelined = jedis.pipelined();
                    for (RedisTupleDTO member : members) {
                        pipelined.zadd(key, member.getScore(), member.getElement());
                    }
                    return pipelined.syncAndReturnAll();
                }
            }, this.jedisPool);
            return results;
        } catch (Exception e) {
            logger.error("call zadd error : ", e);
        }

        return emptyResult;
    }

    public Double zIncrBy(final String key, final double score, final String member) {
        if (StringUtils.isEmpty(key) || StringUtils.isEmpty(member) ) {
            return null;
        }

        try {
            Double resultScore = JedisHelper.doJedisOperation(new JedisCallback<Double>() {
                @Override
                public Double doWithJedis(Jedis jedis) {
                    return jedis.zincrby(key, score, member);
                }
            }, this.jedisPool);
            return resultScore;
        } catch (Exception e) {
            logger.error("call zIncrBy error : ", e);
        }
        return null;
    }

    public Double zScore(final String key,final String member) {
        if (StringUtils.isEmpty(key) || StringUtils.isEmpty(member)) {
            return null;
        }
        try {
            Double score = JedisHelper.doJedisOperation(new JedisCallback<Double>() {
                @Override
                public Double doWithJedis(Jedis jedis) {
                    return jedis.zscore(key, member);
                }
            }, this.jedisPool);
            return score;
        } catch (Exception e) {
            logger.error("call zScore error : ", e);
        }
        return null;
    }

    public Set<RedisTupleDTO> zRevRangeWithScores(final String key, final long start, final long end) {
        Set<RedisTupleDTO> tupleResultSet = new LinkedHashSet<RedisTupleDTO>();
        if (StringUtils.isEmpty(key)) {
            return tupleResultSet;
        }

        try {
            Set<Tuple> tuples = JedisHelper.doJedisOperation(new JedisCallback<Set<Tuple>>() {
                @Override
                public Set<Tuple> doWithJedis(Jedis jedis) {
                    return jedis.zrevrangeWithScores(key, start, end);
                }
            }, this.jedisPool);

            for (Tuple tuple : tuples) {
                tupleResultSet.add(new RedisTupleDTO(tuple.getElement(),tuple.getScore()));
            }
        } catch (Exception e) {
            logger.error("call zRevRangeWithScores error : ", e);
        }
        return tupleResultSet;
    }

    public String hget(final String key, final String field) {
        String value = null;

        try {
            value = JedisHelper.doJedisOperation(new JedisCallback<String>() {
                @Override
                public String doWithJedis(Jedis jedis) {
                    return jedis.hget(key, field);
                }
            }, this.jedisPool);

        } catch (Exception e) {
            logger.error("call hget error : ", e);
        }

        return value;
    }

    public Long hset(final String key, final String field, final String value) {
        Long ret = null;

        try {
            ret = JedisHelper.doJedisOperation(new JedisCallback<Long>() {
                @Override
                public Long doWithJedis(Jedis jedis) {
                    return jedis.hset(key, field, value);
                }
            }, this.jedisPool);

        } catch (Exception e) {
            logger.error("call hset error : ", e);
        }

        return ret;
    }

    public Map<String, String> hgetall(final String key) {
        Map<String,String> ret = null;

        try {
            ret = JedisHelper.doJedisOperation(new JedisCallback<Map<String,String>>() {
                @Override
                public Map<String,String> doWithJedis(Jedis jedis) {
                    return jedis.hgetAll(key);
                }
            }, this.jedisPool);

        } catch (Exception e) {
            logger.error("call hgetall error : ", e);
        }

        return ret;
    }

    public String set(final String key, final String value) {
        String ret = null;
        try {
            ret = JedisHelper.doJedisOperation(new JedisCallback<String>() {
                @Override
                public String doWithJedis(Jedis jedis) {
                    return jedis.set(key, value);
                }
            }, this.jedisPool);

        } catch (Exception e) {
            logger.error("call set error : ", e);
            throw new RuntimeException(e.getMessage());
        }

        return ret;
    }


    public String setex(final String key, final int seconds, final String value) {
        String ret = null;
        try {
            ret = JedisHelper.doJedisOperation(new JedisCallback<String>() {
                @Override
                public String doWithJedis(Jedis jedis) {
                    return jedis.setex(key, seconds, value);
                }
            }, this.jedisPool);

        } catch (Exception e) {
            logger.error("call setex error : ", e);
            throw new RuntimeException(e.getMessage());
        }

        return ret;

    }

    public String set(final byte[] key, final byte[] value) {
        String ret = null;
        try {
            ret = JedisHelper.doJedisOperation(new JedisCallback<String>() {
                @Override
                public String doWithJedis(Jedis jedis) {
                    return jedis.set(key, value);
                }
            }, this.jedisPool);

        } catch (Exception e) {
            logger.error("call set error : ", e);
            throw new RuntimeException(e.getMessage());
        }

        return ret;
    }

    public String setex(final byte[] key, final int seconds, final byte[] value) {
        String ret = null;
        try {
            ret = JedisHelper.doJedisOperation(new JedisCallback<String>() {
                @Override
                public String doWithJedis(Jedis jedis) {
                    return jedis.setex(key, seconds, value);
                }
            }, this.jedisPool);

        } catch (Exception e) {
            logger.error("call setex error : ", e);
            throw new RuntimeException(e.getMessage());
        }

        return ret;
    }

    public Long del(final String... keys){
        Long ret = null;
        try {
            ret = JedisHelper.doJedisOperation(new JedisCallback<Long>() {
                @Override
                public Long doWithJedis(Jedis jedis) {
                    return jedis.del(keys);
                }
            }, jedisPool);
        }catch (Exception e){
            logger.error("call del error : ", e);
        }
        return ret;
    }
}
```
*注意，其中用到的JedisHelper和JedisCallback*

## 关于Jedis的架构
- JedisPool使用了commons-pool，记得去研究一下
- BinaryJedisCommands, JedisCommands是两个interface，其中定义了二进制和String两种类型的Redis所支持的命令
- Jedis, BinaryJedis实现了上面两个interface
- Pipeline可以通过jedis.pipelined来获得，然后反复调用pipeline上的方法，最后pipelined.syncAndReturnAll();

## 关于二进制存储
- 可以用commons-lang包里面的SerializationUtils的serialize和deserialize方法进行序列化和反序列化

## Code
[MySampleCode@Github](https://github.com/wtcctw/me.huachao/tree/master/redis-basic)