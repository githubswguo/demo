##SpringBoot几种定时任务的实现方式

定时任务实现的几种方式：

- Timer：这是java自带的java.util.Timer类，这个类允许你调度一个java.util.TimerTask任务。使用这种方式可以让你的程序按照某一个频度执行，但不能在指定时间运行。一般用的较少。
- ScheduledExecutorService：也jdk自带的一个类；是基于线程池设计的定时任务类,每个调度任务都会分配到线程池中的一个线程去执行,也就是说,任务是并发执行,互不影响。
- Spring Task：Spring3.0以后自带的task，可以将它看成一个轻量级的Quartz，而且使用起来比Quartz简单许多。
- Quartz：这是一个功能比较强大的的调度器，可以让你的程序在指定时间执行，也可以按照某一个频度执行，配置起来稍显复杂。

##使用Timer

	public class TestTimer {
	    public static void main(String[] args) {
	        TimerTask timerTask = new TimerTask() {
	            @Override
	            public void run() {
	                System.out.println("task  run:"+ new Date());
	            }
	        };
	        Timer timer = new Timer();
	        //安排指定的任务在指定的时间开始进行重复的固定延迟执行。这里是每3秒执行一次
	        timer.schedule(timerTask,10,3000);
	    }
	}

##使用ScheduledExecutorService该方法跟Timer类似，直接看demo：
	
	public class TestScheduledExecutorService {
	    public static void main(String[] args) {
	        ScheduledExecutorService service = Executors.newSingleThreadScheduledExecutor();
	        // 参数：1、任务体 2、首次执行的延时时间
	        //      3、任务执行间隔 4、间隔时间单位
	        service.scheduleAtFixedRate(()->System.out.println("task ScheduledExecutorService "+new Date()),
				 0, 3, TimeUnit.SECONDS);
	    }
	}

##使用Spring Task--简单的定时任务

在SpringBoot项目中，我们可以很优雅的使用注解来实现定时任务，首先创建项目，导入依赖：


	<dependencies>
	  <dependency>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-web</artifactId>
	  </dependency>
	  <dependency>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter</artifactId>
	  </dependency>
	  <dependency>
	    <groupId>org.projectlombok</groupId>
	    <artifactId>lombok</artifactId>
	    <optional>true</optional>
	  </dependency>
	  <dependency>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-test</artifactId>
	    <scope>test</scope>
	  </dependency>
	</dependencies>

创建任务类：

	@Slf4j
	@Component
	public class ScheduledService {
	    @Scheduled(cron = "0/5 * * * * *")
	    public void scheduled(){
	        log.info("=====>>>>>使用cron  {}",System.currentTimeMillis());
	    }
	    @Scheduled(fixedRate = 5000)
	    public void scheduled1() {
	        log.info("=====>>>>>使用fixedRate{}", System.currentTimeMillis());
	    }
	    @Scheduled(fixedDelay = 5000)
	    public void scheduled2() {
	        log.info("=====>>>>>fixedDelay{}",System.currentTimeMillis());
	    }
	}

在主类上使用@EnableScheduling注解开启对定时任务的支持，然后启动项目

	2018-09-28 15:40:34.216  INFO 9452 --- [ taskExecutor-4] com.swguo.utils.ScheduledService         : ====>>>>使用fixedRete1538120434216
	2018-09-28 15:40:34.247  INFO 9452 --- [ taskExecutor-5] com.swguo.utils.ScheduledService         : ====>>>fixedDelay1538120434247
	2018-09-28 15:40:35.002  INFO 9452 --- [ taskExecutor-6] com.swguo.utils.ScheduledService         : ======>>> 使用cron 1538120435002
	2018-09-28 15:40:39.218  INFO 9452 --- [ taskExecutor-7] com.swguo.utils.ScheduledService         : ====>>>>使用fixedRete1538120439218
	2018-09-28 15:40:39.249  INFO 9452 --- [ taskExecutor-8] com.swguo.utils.ScheduledService         : ====>>>fixedDelay1538120439249

可以看到三个定时任务都已经执行，并且使同一个线程中串行执行，如果只有一个定时任务，这样做肯定没问题，当定时任务增多，如果一个任务卡死，会导致其他任务也无法执行。

###多线程执行

在传统的Spring项目中，我们可以在xml配置文件添加task的配置，而在SpringBoot项目中一般使用config配置类的方式添加配置，所以新建一个AsyncConfig类

	@Configuration
	@EnableAsync
	public class AsyncConfig {
	     /*
	    此处成员变量应该使用@Value从配置中读取
	     */
	    private int corePoolSize = 10;
	    private int maxPoolSize = 200;
	    private int queueCapacity = 10;
	    @Bean
	    public Executor taskExecutor() {
	        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
	        executor.setCorePoolSize(corePoolSize);
	        executor.setMaxPoolSize(maxPoolSize);
	        executor.setQueueCapacity(queueCapacity);
	        executor.initialize();
	        return executor;
	    }
	}
 
@Configuration：表明该类是一个配置类  
@EnableAsync：开启异步事件的支持

然后在定时任务的类或者方法上添加@Async 。最后重启项目，每一个任务都是在不同的线程中

	2018-09-28 15:40:34.216  INFO 9452 --- [ taskExecutor-4] com.swguo.utils.ScheduledService         : ====>>>>使用fixedRete1538120434216
	2018-09-28 15:40:34.247  INFO 9452 --- [ taskExecutor-5] com.swguo.utils.ScheduledService         : ====>>>fixedDelay1538120434247
	2018-09-28 15:40:35.002  INFO 9452 --- [ taskExecutor-6] com.swguo.utils.ScheduledService         : ======>>> 使用cron 1538120435002
	2018-09-28 15:40:39.218  INFO 9452 --- [ taskExecutor-7] com.swguo.utils.ScheduledService         : ====>>>>使用fixedRete1538120439218
	2018-09-28 15:40:39.249  INFO 9452 --- [ taskExecutor-8] com.swguo.utils.ScheduledService         : ====>>>fixedDelay1538120439249

###执行时间的配置

在上面的定时任务中，我们在方法上使用@Scheduled注解来设置任务的执行时间，并且使用三种属性配置方式：

- fixedRate：定义一个按一定频率执行的定时任务
- fixedDelay：定义一个按一定频率执行的定时任务，与上面不同的是，改属性可以配合initialDelay， 定义该任务延迟执行时间。
- cron：通过表达式来配置任务执行时间

###cron表达式详解

一个cron表达式有至少6个（可能有7个）有空格分隔的时间元素。按顺序依次为：

- 秒（0~59）
- 分钟（0~59）
- 小时（0~23）
- 天（0~31）
- 月（0~11）
- 星期（1~7,1=SUN,SUN,MON,TUE,WED,THU,FRI,SAT）
- 年份(1970~2099)

其中每个元素可以是一个值(如6)，一个连续区间（9-12），一个时间间隔（8-18/4）（/表示每隔四个小时），一个列表（1,3,5）通配符。由于月份中日期和星期中的日期这两个元素互斥，必须对其中一个进行设置。



- 每隔5秒执行一次：/5 * ?
- 每隔1分钟执行一次：0 /1 ?
- 0 0 10,14,16 ? 每天上午10点，下午2点，4点
- 0 0/30 9-17 ? 朝九晚五工作时间内每半小时
- 0 0 12 ? * WED 表示每个星期三中午12点
- 0 0 12 ? 每天中午12点触发
- 0 15 10 ?  每天上午10:15触发
- “0 15 10 ? 2005” 2005年的每天上午10:15触发
- “0 14 * ?” 在每天下午2点到下午2:59期间的每1分钟触发
- “0 0/5 14 ?” 在每天下午2点到下午2:55期间的每5分钟触发
- “0 0/5 14,18 ?” 在每天下午2点到2:55期间和下午6点到6:55期间的每5分钟触发
- “0 0-5 14 ?” 在每天下午2点到下午2:05期间的每1分钟触发
- “0 10,44 14 ? 3 WED” 每年三月的星期三的下午2:10和2:44触发
- “0 15 10 ? * MON-FRI” 周一至周五的上午10:15触发
- “0 15 10 15 * ?” 每月15日上午10:15触发
- 

