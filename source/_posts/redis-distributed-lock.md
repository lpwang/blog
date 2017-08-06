---
title: Redis-分布式锁的应用
date: 2017-08-06 17:01:19
categories: redis
tags: redis
---

> 在分布式架构情况下，例如支付系统，仓储物流系统，订单系统。各个服务协同工作完成客户的请求。在分布式架构情况下用户的一个请求会调用多个后台系统来完成。在这种情况下如果出现网络抖动，或者用户进行多次订单提交操作就会导致后台记录两条相同的订单数据。给用户代理很差的体验。借助redis的线程安全的特性进行分布式锁的设计。

### 介绍几个redis的方法。

1. setnx (set if not exist)  - 利用这条命令进行key的存储。
2. expire - 利用这条命令将key设置过期时间。

### 实现思路

1. 为请求来的线程添加请求的过期时间，当请求时间超过这个预设的阈值的时候 return false 。退出锁的竞争。
2. 使用setnx命令设置key和value。如果key没有存在请求线程设置成功 return 1。如果key存在return 0。表示已经有线程获得了锁，然后没有获得锁的线程再次进入循环竞争获取锁。
3. 如果获取锁的线程，将setnx的key设置过期时间，设置过期时间是为了防止获取锁的线程在执行请求时，如果服务down了，那么其他线程的请求再无法获取锁，所以使用命令 expire 设置 key 的过期时间确保在处理请求过程中，如果服务意外终止，设置的 key 依旧可以被删除，其他线程也会在过期时间后 key 被删除后，再次的获取锁。
4. 当获取锁的线程处理完请求后将 key 删除，但是有一个问题是，当获取锁的线程（线程一）处理请求的时间很长长到超出了设置的key的过期时间，那么其他线程（线程二）就会获取锁并处理自己的请求，当线程二正在处理自己的请求的时候，线程一刚好处理完它的请求，这是就将key删除，但是线程一删除的是线程二的key，线程一的key早因过期时间已经到了而被删除了。所以需要对线程设置的key的value进行校验。

### 代码

1. RedisLock.java 类中定义了获取锁和释放锁的方法，实现思路已经说过。不在赘述。
```java
public class RedisLock {
	
	private SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
	private static final ThreadLocal<String> EXPIRETIMETHREADLOCAL = new ThreadLocal<String>();

	public boolean lock (String key,int timeOut,int expire) throws InterruptedException {

		Date date = new Date();
		Jedis redis = ConnectFactory.getNewRedisConnection();
		long timeOutMoment = System.currentTimeMillis() + (timeOut * 1000);
		while (timeOutMoment > System.currentTimeMillis()) {
			date.setTime(System.currentTimeMillis() + (expire * 1000));
			String expireTime = sdf.format(date);
			if (redis.setnx(key,expireTime) == 1) {
				// 设置key的过期时间
				EXPIRETIMETHREADLOCAL.set(expireTime);
				redis.expire(key, expire);
				System.out.println(Thread.currentThread().getName()+ "  " +"获取锁");
				return true;
			}

			//Thread.currentThread().sleep(3000);
		}

		return false;
	}
	
	public boolean unLock (String key) {
		Jedis redis = ConnectFactory.getRedisConnection();
		ThreadLocal<String> expireTimeThreadLocal = new ThreadLocal<String>();
		if (redis.get(key).equals(EXPIRETIMETHREADLOCAL.get())) { //
			return redis.del(key) > 0 ? true : false;
		}
		System.out.println("key存活时间已经超出预设时间。");
		return true;
	}
	
}
```

2. 调用RedisLock 获取锁的启动类。我模拟了50个线程进行并发请求获取锁。

```java
public class Start {
	
	public static void main(String[] args) throws InterruptedException {
		RedisLock redisLock = new RedisLock();
		String key = "001";
		int timeOut = 1 * 60 * 60;
		int expire = 120;
		for (int i=0;i<50;i++) {
			Thread thread = new Thread(new Runnable() {
				
				@Override
				public void run() {
					try {
						if (redisLock.lock(key, timeOut, expire)) {
							Thread.currentThread().sleep(5000);
							System.out.println(redisLock.unLock(key));
						}
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}
				
			});
			thread.setName("Thread - "+i);
			thread.start();
		}
		
		
	}
	
}
```

### 结束

到此使用 redis 实现分布式锁结束了。关于redis 的应用场景还有很多，以后我会慢慢的更新。
