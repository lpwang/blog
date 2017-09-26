---
title: 使用jmeter压测Netty通讯服务
categories: jmeter
date: 2017-09-26 17:07:20
tags: jmeter netty
---

## 使用jmeter压测Netty通讯服务

> 项目中使用`netty`框架开发底层通讯服务器模块。设备与通讯服务器采用`tcp长连接`的通讯方式。建立链接过程使用`ssl`对连接通道进行加密。建立连接后定时向服务器发送心跳数据。

### 测试目标

1. 单台 `netty server` 维持长连接数量。
2. 长连接数量增长下`tps`指数。
3. 长连接数量增长下`jvm的堆栈`内存变化。

### 测试工具

1. jmeter
2. jvisualvm

### 数据准备

编写脚本准备`100W`终端信息。直接上代码。

```java
public class ProduceMAC {
	
	private static final ConcurrentLinkedQueue<Jedis> JEDISQUEUE = new ConcurrentLinkedQueue<Jedis>();
	private static BufferedWriter WRITER = null;
	private static AtomicInteger MAC_NUM = new AtomicInteger(0);
	
	static {
		// init JEDISQUEUE
		for (int i=0;i<1000;i++) {
			Jedis redis = new Jedis("192.168.79.83", 6379);
			redis.select(1);
			JEDISQUEUE.add(redis);
		}
		
		// init sql file
		Path path = Paths.get("D:", "device.sql");
		File devic_sql_file = new File(path.toString());
		try {
			Files.createParentDirs(devic_sql_file);
			WRITER = new BufferedWriter(new FileWriter( devic_sql_file ));
		}catch (Exception e) {
			throw new RuntimeException(e.getMessage());
		}
	}
	
	public static void main(String[] args) {
		
		
		
		HashSet<String> MacSet = new HashSet<String>();
		String linChar = ":";
		
		for (int i=0;i<1000000;i++) {
		
			String mac_1 = UUID.randomUUID().toString().replaceAll("-", "").toUpperCase().substring(0, 2);
			String mac_2 = UUID.randomUUID().toString().replaceAll("-", "").toUpperCase().substring(2, 4);
			String mac_3 = UUID.randomUUID().toString().replaceAll("-", "").toUpperCase().substring(4, 6);
			String mac_4 = UUID.randomUUID().toString().replaceAll("-", "").toUpperCase().substring(6, 8);
			String mac_5 = UUID.randomUUID().toString().replaceAll("-", "").toUpperCase().substring(8, 10);
			String mac_6 = UUID.randomUUID().toString().replaceAll("-", "").toUpperCase().substring(10, 12);
			
			StringBuilder mac = new StringBuilder();
			mac.append(mac_1).append(linChar).append(mac_2).append(linChar).append(mac_3)
			.append(linChar).append(mac_4).append(linChar).append(mac_5).append(linChar)
			.append(mac_6);
			
			MacSet.add(mac.toString());
			
		}
		
		
		if (1000000 == MacSet.size()) {
			PersistenceMAC(MacSet);
			PersistenceSQL(MacSet);
		}
		
	}
	
	
	private static void PersistenceMAC (HashSet<String> macSet) {
		System.out.println("> redis");
		ExecutorService es = Executors.newFixedThreadPool(100);
		for (String mac : macSet) {
			es.execute(new Runnable() {
				
				@Override
				public void run() {
					Jedis redis;
					try {
						redis = JEDISQUEUE.poll();
						int c = ProduceMAC.MAC_NUM.incrementAndGet();
						redis.hset("dev_mac",String.valueOf(c), mac);
						JEDISQUEUE.add(redis);
					} catch (Exception e) {
						e.printStackTrace();
					}
				}
			});
		}
	}
	
	
	
	private static void PersistenceSQL (HashSet<String> macSet) {
		
		if (null == WRITER) {
			throw new RuntimeException("no writer");
		}
		
		StringBuilder SQLSB = new StringBuilder();
		SQLSB = initSQL(SQLSB);
		System.out.println("> sql");
		
		int count = 1;
		int segment = 5000;
		
		try {
			
			for (String mac : macSet) {
				
				if (0 == count % segment) {
					SQLSB.append("('"+ UUID.randomUUID().toString() +"', '0208-4680-4892-1600', 'cb8b3ab7580c46528ba4e4aef59deb47', 'ylcs-"+count+"', '11', '"+mac+"', 'Android1', 'xtgly', now(), 'xtgly', now())");
					SQLSB.append(",");
					try {
						String writeSQL = SQLSB.toString().substring(0, SQLSB.length()-1)+";";
						System.out.println("writeSQL > "+count);
						WRITER.write(writeSQL);
						WRITER.newLine();
						WRITER.flush();
					} catch (Exception e) {
						throw new RuntimeException(e.getMessage());
					}
					SQLSB = initSQL(SQLSB);
				}else {
					SQLSB.append("('"+ UUID.randomUUID().toString() +"', '0208-4680-4892-1600', 'cb8b3ab7580c46528ba4e4aef59deb47', 'ylcs-"+count+"', '11', '"+mac+"', 'Android1', 'xtgly', now(), 'xtgly', now())");
					SQLSB.append(",");
				}
				
				System.out.println(count);
				count++;
			}
			
		}catch (Exception e) {
			throw new RuntimeException(e.getMessage());
		}finally {
			try {
				WRITER.close();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
		
		System.out.println("> sql done");
		
	}
	
	private static StringBuilder initSQL (StringBuilder sql) {
		sql.delete(0, sql.length());
		sql.append("INSERT INTO iot_basic.iot_device ")
		.append("(device_id, product_id, active_code, device_name, device_type, device_mac, device_os, adder, add_time, updater, update_time) ")
		.append("VALUES ");
		return sql;
	}
	
}
```

1. 使用`UUID`生成`100W mac`地址
2. 再使用生成的`mac`地址分别生成`sql`脚本，添加到`redis`中
3. 将`mac`信息使用`redis`的哈希表数据结构进行存储 `redis.hset("dev_mac",String.valueOf(c), mac); ` `dev_mac` 作为`key` 循环`mac`集合的`index` 作为 `hash_key`, `mac` 作为`hash_value`。这样处理方便多机分布式压测。
4. 循环mac地址集合生成`sql`脚本，为了进行批处理操作，每`5000`条`mac`数据作为一个`insert into` 语句
5. 测试脚本运行结束后，将生成的`sql`脚本 `source` 到数据库中。

### 压力测试脚本

直接上代码

```java
public class NettyClient extends AbstractJavaSamplerClient {
	
	private SampleResult results;
	
	private static Log log = LogFactory.getLog(NettyClient.class);
	
	private Jedis redis = null;
	
	private static void start (ConnectionInfoDomain connectionInfoDomain,DeviceRegisterDomain deviceRegisterDomain) throws InterruptedException {
		
		
		String host = connectionInfoDomain.getHost();
		int port = connectionInfoDomain.getPort();
		
		log.info("准备建立连接: host - "+host+", post - "+port);
		
		
		StringBuilder cerFilePath = new StringBuilder();
		cerFilePath.append(DeviceConstant.DEV_KEY_FILE_BASE_PATH).append(File.separatorChar).append(deviceRegisterDomain.getBusiness().getProductId())
		.append(File.separatorChar).append(deviceRegisterDomain.getBusiness().getDevMac().replaceAll(":", "")).append(File.separatorChar)
		.append(DeviceConstant.DEV_CER_FILE_NAME);
		
		StringBuilder privateKeyFilePath = new StringBuilder();
		privateKeyFilePath.append(DeviceConstant.DEV_KEY_FILE_BASE_PATH).append(File.separatorChar).append(deviceRegisterDomain.getBusiness().getProductId()).append(File.separatorChar)
		.append(deviceRegisterDomain.getBusiness().getDevMac().replaceAll(":", "")).append(File.separatorChar).append(DeviceConstant.PRIVATE_KEY_FILE_NAME);
		
		if (!new File(cerFilePath.toString()).exists() || !new File(privateKeyFilePath.toString()).exists()) {
			try {
				ActiveRespEntry activeRespEntry = SslFactory.getCertificate("192.168.79.85", 8060, "/api/iot-basic/device/activate",deviceRegisterDomain);
				SSLUtils.persistenceCER(activeRespEntry.getCertificate(),deviceRegisterDomain);
				SSLUtils.persistenceTerminalId(deviceRegisterDomain.getBusiness().getDevMac(), activeRespEntry.getDeviceId(), connectionInfoDomain);
			}catch (Exception e) {
				throw new RuntimeException(e.getMessage());
			}
		}
		
		EventLoopGroup group = new NioEventLoopGroup();
		try {
			Bootstrap b = new Bootstrap();
			b.group(group)
			.channel(NioSocketChannel.class)
			.option(ChannelOption.TCP_NODELAY, true) // 不使用 Tcp 缓存
			.handler(new ChannelInitializer<SocketChannel>() {

				@Override
				protected void initChannel(SocketChannel ch) throws Exception {
					ChannelPipeline channelPipeline = ch.pipeline();
					// channelPipeline.addFirst(new StringDecoder(Charset.forName("UTF-8"))); // 添加字符串解码处理器
					// channelPipeline.addLast(new DeviceRegisterCommandHandler(deviceRegisterDomain));
					// channelPipeline.addLast(new AdCommandHandler());
					// channelPipeline.addLast(new AppUpdateHandler());
					// channelPipeline.addLast(new HeartBeatHandler());
					
					
					SSLEngine sslEngine = SslContextFactory2.getClientContext(deviceRegisterDomain).newEngine(ch.alloc());
					sslEngine.setUseClientMode(true);
					channelPipeline.addLast(new SslHandler(sslEngine));
					channelPipeline.addLast(new HeartBeatHandler( SSLUtils.getTerminalId(deviceRegisterDomain.getBusiness().getDevMac(), connectionInfoDomain) ));
				}

			});
			
			ChannelFuture f = b.connect(host, port).sync();
			f.channel().closeFuture().sync();
		} finally {
			group.shutdownGracefully();
		}
		
		
	}
	
	private static <T> T loadConfigFile (String configFilePath,Class<T> parseToClass) throws IOException, NoSuchFieldException, SecurityException {
		if (!StringUtils.isNotEmpty(configFilePath))
			log.error("设备信息文件路径不能为空！",new RuntimeException("设备信息文件路径不能为空！"));
		
		File configFile = new File(configFilePath);
		if(configFile.exists()) {
			// 读取设备配置信息
			return new Gson().fromJson(new BufferedReader(new FileReader(configFile)), parseToClass);
		}else {
			log.error("设备信息文件路径不能到达！配置的错误路径为： "+configFile.getPath());
			throw new RuntimeException("设备信息文件路径不能为空！");
		}
	}
	
	
	
	@Override
	public void setupTest(JavaSamplerContext context) {
		results = new SampleResult();  
        results.setSamplerData(toString());  
        results.setDataType("text");  
        results.setContentType("text/plain");  
        results.setDataEncoding("UTF-8");  
  
        results.setSuccessful(true);  
        results.setResponseMessageOK();  
        results.setResponseCodeOK();  
          
	}
	
	@Override
	public SampleResult runTest(JavaSamplerContext arg0)  {
		try {
			
			results.sampleStart();
			
			Path configFilePath =  Paths.get("/home","wanglp","communication-pressure-test","device-info", "config.json"); //"‪E:/Other/device_client/config.json";
			//Path deviceFilePath =  Paths.get("E:", "Other", "device_client", "deviceInfo.json"); //"‪E:/Other/device_client/deviceInfo.json";
			
			ConnectionInfoDomain connectionInfoDomain = loadConfigFile(configFilePath.toString(), ConnectionInfoDomain.class);
			//DeviceRegisterDomain deviceRegisterDomain = loadConfigFile(deviceFilePath.toString(), DeviceRegisterDomain.class);
			
			log.info("解析连接配置文件获取的实体为："+connectionInfoDomain);
			//log.debug("解析终端激活配置文件获取的实体为："+deviceRegisterDomain);
			
			//RedisFactory.setJedis(connectionInfoDomain.getRedisHost(),connectionInfoDomain.getRedisPort());
			Jedis redis = RedisFactory.getNewConnection(connectionInfoDomain.getRedisHost(),connectionInfoDomain.getRedisPort());
			redis.select(1);
			String threadNum = Thread.currentThread().getName().split("-")[1].trim();
			int device_num = connectionInfoDomain.getBeginIndex() + Integer.parseInt(threadNum);
			log.info("Current device num > "+device_num);
			
			String mac = redis.hget("dev_mac", String.valueOf(device_num));
			log.info("Current Mac > "+mac);
			redis.close();
			
			DeviceRegisterDomain deviceRegisterDomain = new DeviceRegisterDomain();
			DeviceRegisterDomain.Business bussiness = new DeviceRegisterDomain.Business(); 
			bussiness.setProductId("0208-4680-4892-1600");
			bussiness.setAuthorizationCode("CB8B3AB7580C46528BA4E4AEF59DEB47");
			bussiness.setDevMac(mac);
			deviceRegisterDomain.setBusiness(bussiness);
			
			NettyClient.start(connectionInfoDomain, deviceRegisterDomain);
			
			results.sampleEnd();
			return results;
		} catch (Exception e) {
			log.info("xxxxx"+e.getMessage());
			throw new RuntimeException(e.getMessage());
		}
		
	}
}
```

1. 编写自定义压力测试脚本之前需要几部操作。
  - 找到`jmeter`安装目录 `/apache-jmeter-3.2/lib/ext` 将`ext`目录下的`ApacheJMeter_core.jar` 和 `ApacheJMeter_java.jar` 引入进测试脚本的工程中。
  - 将主类 `extends AbstractJavaSamplerClient` 覆盖其中的两个方法 `runTest` 和 `setupTest`。
  - 在`setupTest`方法中生成实例 `SampleResult` 对 `SampleResult` 进行设置 然后在`runTest` 方法中进行添加。
2. 将编写好的脚本打包成`jar`导入到`jmeter` 目录`/apache-jmeter-3.2/lib/ext` 中 ， 再将压测脚本所需要的依赖包也导入到 `/apache-jmeter-3.2/lib/ext` 中。
3. 打开`jmeter gui` 添加测试计划，编写测试计划，具体测试计划的线程组编辑下面再介绍。
4. 使用命令进行压力测试 ` sh ./jmeter.sh -n -t /home/wanglp/communication-pressure-test/device-info/csjh.jmx ` `-n` 表示 `no gui`  `-t` 表示编写的测试计划。

### 测试计划的线程组编写

1. 添加测试任务，再在测试计划下添加线程组，如图

![image](http://wx3.sinaimg.cn/large/74b07056ly1fhq4k9amh9j216u0ejwfj.jpg)

- 线程数: 表示启动的线程总数。
- Ramp-up period : 表示多少秒后将设置的线程数全部启动。
- 循环次数: 表示该线程组配置循环执行几次。

### Jvisualvm 的使用

`jvisualvm` 用来监控`java`程序的`堆栈内存`，`cpu`，`线程`，`类加载`数等信息的使用情况。`jvisualvm`程序包含在 `jdk` 家目录下 `bin` 目录下。为使得 `jvisualvm` 监控远程的 `java` 程序 需要为远程的`java`程序的启动脚本添加`jmx`参数。

```java
-Dcom.sun.management.jmxremote.port=<port>
-Dcom.sun.management.jmxremote.ssl=false 
-Dcom.sun.management.jmxremote.authenticate=false
```
其中 <port> 是 `jmx` 连接的端口。启动后远程程序后，再回到本机的 `jvisualvm` ，在远程一栏中添加远程主机，添加之后，再在远程主机下面添加 `jmx` 连接。

如图：
![image](http://wx1.sinaimg.cn/large/74b07056ly1fhqelo3fwtj20cc0j5wes.jpg)

监控 `jvisualvm` 参数 如图：
![image](http://wx1.sinaimg.cn/large/74b07056ly1fhqeln4z5dj21d60ptmyx.jpg)

### 测试报告

| 线程数  | 栈内存  | 堆内存  | 压测持续时间         | Server 状态 |
| ---- | ---- | ---- | -------------- | --------- |
| 3000 | 128M | 256M | 1 小时 18 分 34 秒 | down      |
|      |      |      |                |           |
|      |      |      |                |           |

### 测试发现的问题

1. 从微服务获取证书时间长，将线程启动频率调整至 1 thread / 3s 以内时，微服务堆内存溢出。
2. 随着压测时间的加长，`netty server` 的堆内存持续增加，直至堆内存溢出。
