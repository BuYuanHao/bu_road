# Java定时任务技术方案总结

## 目录
1. [概述](#概述)
2. [原生定时任务](#原生定时任务)
   - [Timer和TimerTask](#timer和timertask)
   - [ScheduledExecutorService](#scheduledexecutorservice)
3. [Spring定时任务](#spring定时任务)
   - [@Scheduled注解](#scheduled注解)
   - [TaskScheduler](#taskscheduler)
4. [Quartz定时任务框架](#quartz定时任务框架)
5. [分布式定时任务](#分布式定时任务)
   - [XXL-JOB](#xxl-job)
   - [Elastic-Job](#elastic-job)
6. [技术选型对比](#技术选型对比)
7. [最佳实践](#最佳实践)

## 概述

定时任务是企业级应用中必不可少的功能，用于执行周期性的业务逻辑，如数据同步、报表生成、日志清理等。Java生态中提供了多种定时任务解决方案，每种都有其特点和适用场景。

## 原生定时任务

### Timer和TimerTask

#### 特点
- JDK原生支持，无需额外依赖
- 基于单线程执行，所有任务串行执行
- 如果某个任务执行时间过长，会影响后续任务的执行
- 如果某个任务抛出异常，会导致整个Timer线程停止

#### 适用场景
- 简单的单机定时任务
- 任务执行时间短且不会抛出异常
- 对精度要求不高的场景

#### 示例代码

```java
import java.util.Timer;
import java.util.TimerTask;
import java.util.Date;

public class TimerExample {
    public static void main(String[] args) {
        Timer timer = new Timer();
        
        // 延迟1秒执行，然后每3秒执行一次
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                System.out.println("定时任务执行时间：" + new Date());
                // 执行具体业务逻辑
                processBusinessLogic();
            }
        }, 1000, 3000);
        
        // 程序运行10秒后停止
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        timer.cancel();
    }
    
    private static void processBusinessLogic() {
        System.out.println("执行业务逻辑...");
    }
}
```

#### 优势
- 简单易用
- 无外部依赖
- 内存占用小

#### 劣势
- 单线程执行，性能有限
- 异常处理机制差
- 不支持复杂的调度策略

### ScheduledExecutorService

#### 特点
- JDK 1.5引入，基于线程池实现
- 支持多线程并发执行
- 异常不会影响其他任务的执行
- 提供更丰富的调度方法

#### 适用场景
- 需要并发执行多个定时任务
- 对任务执行的稳定性要求较高
- 中等复杂度的定时任务调度

#### 示例代码

```java
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class ScheduledExecutorExample {
    private static final ScheduledExecutorService scheduler = 
        Executors.newScheduledThreadPool(3);
    
    public static void main(String[] args) {
        // 固定频率执行（每5秒执行一次）
        scheduler.scheduleAtFixedRate(() -> {
            System.out.println("固定频率任务执行：" + getCurrentTime());
            businessTask1();
        }, 0, 5, TimeUnit.SECONDS);
        
        // 固定延迟执行（上次执行完成后延迟3秒再执行）
        scheduler.scheduleWithFixedDelay(() -> {
            System.out.println("固定延迟任务执行：" + getCurrentTime());
            businessTask2();
        }, 2, 3, TimeUnit.SECONDS);
        
        // 一次性延迟执行
        scheduler.schedule(() -> {
            System.out.println("一次性任务执行：" + getCurrentTime());
            businessTask3();
        }, 10, TimeUnit.SECONDS);
        
        // 运行20秒后关闭
        scheduler.schedule(() -> {
            System.out.println("关闭调度器");
            scheduler.shutdown();
        }, 20, TimeUnit.SECONDS);
    }
    
    private static String getCurrentTime() {
        return LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    }
    
    private static void businessTask1() {
        System.out.println("执行业务任务1");
    }
    
    private static void businessTask2() {
        System.out.println("执行业务任务2");
    }
    
    private static void businessTask3() {
        System.out.println("执行业务任务3");
    }
}
```

#### 优势
- 多线程并发执行
- 异常隔离，稳定性好
- 提供多种调度策略
- 支持线程池配置

#### 劣势
- 不支持持久化
- 不支持集群
- 调度策略相对简单

## Spring定时任务

### @Scheduled注解

#### 特点
- Spring框架原生支持
- 基于注解配置，简单易用
- 支持cron表达式
- 与Spring容器无缝集成

#### 适用场景
- Spring应用中的定时任务
- 中小型项目的定时任务需求
- 需要与Spring Bean集成的场景

#### 示例代码

```java
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

@Component
public class ScheduledTasks {
    
    private static final DateTimeFormatter formatter = 
        DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    
    // 每5秒执行一次
    @Scheduled(fixedRate = 5000)
    public void fixedRateTask() {
        System.out.println("固定频率任务执行：" + getCurrentTime());
        // 执行业务逻辑
        processDataSync();
    }
    
    // 上次执行完成后延迟3秒执行
    @Scheduled(fixedDelay = 3000)
    public void fixedDelayTask() {
        System.out.println("固定延迟任务执行：" + getCurrentTime());
        // 执行业务逻辑
        processLogCleanup();
    }
    
    // 使用cron表达式：每天凌晨1点执行
    @Scheduled(cron = "0 0 1 * * ?")
    public void cronTask() {
        System.out.println("定时报表任务执行：" + getCurrentTime());
        // 执行业务逻辑
        generateDaillyReport();
    }
    
    // 工作日上午9点执行
    @Scheduled(cron = "0 0 9 * * MON-FRI")
    public void workdayTask() {
        System.out.println("工作日任务执行：" + getCurrentTime());
        // 执行业务逻辑
        sendWorkdayNotification();
    }
    
    private String getCurrentTime() {
        return LocalDateTime.now().format(formatter);
    }
    
    private void processDataSync() {
        System.out.println("执行数据同步...");
        // 具体的数据同步逻辑
    }
    
    private void processLogCleanup() {
        System.out.println("执行日志清理...");
        // 具体的日志清理逻辑
    }
    
    private void generateDaillyReport() {
        System.out.println("生成日报表...");
        // 具体的报表生成逻辑
    }
    
    private void sendWorkdayNotification() {
        System.out.println("发送工作日通知...");
        // 具体的通知发送逻辑
    }
}
```

#### 启用配置

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;
import java.util.concurrent.Executors;

@Configuration
@EnableScheduling
public class SchedulingConfig implements SchedulingConfigurer {
    
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        // 配置线程池，避免任务阻塞
        taskRegistrar.setScheduler(Executors.newScheduledThreadPool(10));
    }
}
```

#### 优势
- 与Spring无缝集成
- 支持cron表达式
- 配置简单
- 支持多种触发方式

#### 劣势
- 不支持持久化
- 不支持集群
- 功能相对简单

### TaskScheduler

#### 特点
- Spring提供的更灵活的任务调度器
- 支持动态调度
- 可以程序化配置任务

#### 示例代码

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.TaskScheduler;
import org.springframework.scheduling.support.CronTrigger;
import org.springframework.stereotype.Service;
import java.util.concurrent.ScheduledFuture;

@Service
public class DynamicSchedulingService {
    
    @Autowired
    private TaskScheduler taskScheduler;
    
    private ScheduledFuture<?> scheduledFuture;
    
    // 动态启动任务
    public void startTask(String cronExpression) {
        if (scheduledFuture != null) {
            scheduledFuture.cancel(false);
        }
        
        scheduledFuture = taskScheduler.schedule(() -> {
            System.out.println("动态任务执行：" + System.currentTimeMillis());
            executeBusinessLogic();
        }, new CronTrigger(cronExpression));
    }
    
    // 停止任务
    public void stopTask() {
        if (scheduledFuture != null) {
            scheduledFuture.cancel(false);
            scheduledFuture = null;
        }
    }
    
    private void executeBusinessLogic() {
        System.out.println("执行动态业务逻辑");
    }
}
```

## Quartz定时任务框架

### 特点
- 功能强大的开源定时任务框架
- 支持持久化
- 支持集群
- 提供丰富的调度策略
- 支持任务的暂停、恢复、删除等操作

### 适用场景
- 复杂的定时任务调度需求
- 需要持久化的场景
- 集群环境下的定时任务
- 需要动态管理任务的场景

### 示例代码

#### Maven依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

#### Job类定义

```java
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.stereotype.Component;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

@Component
public class DataSyncJob implements Job {
    
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        System.out.println("Quartz任务执行：" + getCurrentTime());
        
        try {
            // 执行具体业务逻辑
            processDataSync();
        } catch (Exception e) {
            // 异常处理
            System.err.println("任务执行异常：" + e.getMessage());
            throw new JobExecutionException(e);
        }
    }
    
    private String getCurrentTime() {
        return LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    }
    
    private void processDataSync() {
        System.out.println("执行数据同步业务逻辑...");
        // 模拟业务处理时间
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println("数据同步完成");
    }
}
```

#### 配置类

```java
import org.quartz.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class QuartzConfig {
    
    @Bean
    public JobDetail dataSyncJobDetail() {
        return JobBuilder.newJob(DataSyncJob.class)
                .withIdentity("dataSyncJob", "dataGroup")
                .withDescription("数据同步任务")
                .storeDurably() // 持久化
                .build();
    }
    
    @Bean
    public Trigger dataSyncTrigger() {
        return TriggerBuilder.newTrigger()
                .forJob(dataSyncJobDetail())
                .withIdentity("dataSyncTrigger", "dataGroup")
                .withDescription("数据同步触发器")
                .withSchedule(CronScheduleBuilder.cronSchedule("0 */5 * * * ?")) // 每5分钟执行一次
                .build();
    }
    
    // 配置Scheduler属性
    @Bean
    public SchedulerFactoryBean schedulerFactoryBean() {
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        factory.setJobDetails(dataSyncJobDetail());
        factory.setTriggers(dataSyncTrigger());
        return factory;
    }
}
```

#### 动态任务管理

```java
import org.quartz.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class QuartzJobService {
    
    @Autowired
    private Scheduler scheduler;
    
    // 动态添加任务
    public void addJob(String jobName, String jobGroup, String cronExpression, Class<? extends Job> jobClass) {
        try {
            JobDetail jobDetail = JobBuilder.newJob(jobClass)
                    .withIdentity(jobName, jobGroup)
                    .build();
            
            Trigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity(jobName + "Trigger", jobGroup)
                    .withSchedule(CronScheduleBuilder.cronSchedule(cronExpression))
                    .build();
            
            scheduler.scheduleJob(jobDetail, trigger);
        } catch (SchedulerException e) {
            System.err.println("添加任务失败：" + e.getMessage());
        }
    }
    
    // 暂停任务
    public void pauseJob(String jobName, String jobGroup) {
        try {
            scheduler.pauseJob(JobKey.jobKey(jobName, jobGroup));
        } catch (SchedulerException e) {
            System.err.println("暂停任务失败：" + e.getMessage());
        }
    }
    
    // 恢复任务
    public void resumeJob(String jobName, String jobGroup) {
        try {
            scheduler.resumeJob(JobKey.jobKey(jobName, jobGroup));
        } catch (SchedulerException e) {
            System.err.println("恢复任务失败：" + e.getMessage());
        }
    }
    
    // 删除任务
    public void deleteJob(String jobName, String jobGroup) {
        try {
            scheduler.deleteJob(JobKey.jobKey(jobName, jobGroup));
        } catch (SchedulerException e) {
            System.err.println("删除任务失败：" + e.getMessage());
        }
    }
}
```

### 优势
- 功能强大，支持复杂调度
- 支持持久化和集群
- 提供完善的任务管理API
- 支持多种触发器类型
- 异常处理机制完善

### 劣势
- 配置相对复杂
- 学习成本较高
- 资源消耗相对较大

## 分布式定时任务

### XXL-JOB

#### 特点
- 轻量级分布式任务调度平台
- 提供Web管理界面
- 支持任务失败重试
- 支持任务依赖
- 支持多种执行模式

#### 适用场景
- 分布式系统中的定时任务
- 需要可视化管理的场景
- 需要任务监控和报警的场景
- 大规模定时任务调度

#### 示例代码

```java
import com.xxl.job.core.handler.annotation.XxlJob;
import org.springframework.stereotype.Component;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

@Component
public class XxlJobHandler {
    
    @XxlJob("dataSyncJobHandler")
    public void dataSyncJobHandler() throws Exception {
        System.out.println("XXL-JOB数据同步任务开始：" + getCurrentTime());
        
        try {
            // 执行业务逻辑
            processDataSync();
            System.out.println("XXL-JOB数据同步任务完成：" + getCurrentTime());
        } catch (Exception e) {
            System.err.println("XXL-JOB任务执行异常：" + e.getMessage());
            throw e;
        }
    }
    
    @XxlJob("reportJobHandler")
    public void reportJobHandler() throws Exception {
        System.out.println("XXL-JOB报表生成任务开始：" + getCurrentTime());
        
        try {
            // 执行报表生成逻辑
            generateReport();
            System.out.println("XXL-JOB报表生成任务完成：" + getCurrentTime());
        } catch (Exception e) {
            System.err.println("XXL-JOB报表任务执行异常：" + e.getMessage());
            throw e;
        }
    }
    
    private String getCurrentTime() {
        return LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    }
    
    private void processDataSync() {
        System.out.println("执行数据同步逻辑...");
        // 模拟业务处理
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    private void generateReport() {
        System.out.println("生成业务报表...");
        // 模拟报表生成
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

#### 配置

```yaml
# application.yml
xxl:
  job:
    admin:
      addresses: http://127.0.0.1:8080/xxl-job-admin
    executor:
      appname: xxl-job-executor-sample
      ip: 
      port: 9999
      logpath: /data/applogs/xxl-job/jobhandler
      logretentiondays: 30
    accessToken: 
```

### Elastic-Job

#### 特点
- 当当网开源的分布式调度解决方案
- 支持分片执行
- 支持故障转移
- 提供运维平台
- 支持多种作业类型

#### 适用场景
- 需要分片处理大量数据的场景
- 对高可用要求较高的分布式任务
- 需要精确的分片策略的场景

#### 示例代码

```java
import org.apache.shardingsphere.elasticjob.api.ShardingContext;
import org.apache.shardingsphere.elasticjob.simple.job.SimpleJob;
import org.springframework.stereotype.Component;

@Component
public class MyElasticJob implements SimpleJob {
    
    @Override
    public void execute(ShardingContext shardingContext) {
        System.out.println("分片项：" + shardingContext.getShardingItem());
        System.out.println("分片参数：" + shardingContext.getShardingParameter());
        System.out.println("作业名称：" + shardingContext.getJobName());
        
        // 根据分片项执行不同的业务逻辑
        switch (shardingContext.getShardingItem()) {
            case 0:
                processShardingData1();
                break;
            case 1:
                processShardingData2();
                break;
            case 2:
                processShardingData3();
                break;
            default:
                processDefaultShardingData();
                break;
        }
    }
    
    private void processShardingData1() {
        System.out.println("处理分片1的数据...");
    }
    
    private void processShardingData2() {
        System.out.println("处理分片2的数据...");
    }
    
    private void processShardingData3() {
        System.out.println("处理分片3的数据...");
    }
    
    private void processDefaultShardingData() {
        System.out.println("处理默认分片数据...");
    }
}
```

## 技术选型对比

| 技术方案 | 复杂度 | 功能丰富度 | 性能 | 集群支持 | 持久化 | 管理界面 | 适用场景 |
|---------|--------|------------|------|----------|--------|----------|----------|
| Timer | 简单 | 低 | 低 | 不支持 | 不支持 | 无 | 简单单机任务 |
| ScheduledExecutorService | 简单 | 中 | 中 | 不支持 | 不支持 | 无 | 中等复杂度单机任务 |
| Spring @Scheduled | 简单 | 中 | 中 | 不支持 | 不支持 | 无 | Spring应用中的定时任务 |
| Quartz | 中等 | 高 | 高 | 支持 | 支持 | 有 | 复杂的企业级应用 |
| XXL-JOB | 中等 | 高 | 高 | 支持 | 支持 | 有 | 分布式任务调度 |
| Elastic-Job | 复杂 | 高 | 高 | 支持 | 支持 | 有 | 大数据量分片处理 |

## 最佳实践

### 1. 任务执行时间监控

```java
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

@Component
public class MonitoredScheduledTask {
    
    @Scheduled(fixedRate = 30000) // 每30秒执行一次
    public void monitoredTask() {
        long startTime = System.currentTimeMillis();
        String taskName = "数据同步任务";
        
        try {
            System.out.println(String.format("[%s] %s 开始执行", getCurrentTime(), taskName));
            
            // 执行业务逻辑
            performBusinessLogic();
            
            long duration = System.currentTimeMillis() - startTime;
            System.out.println(String.format("[%s] %s 执行完成，耗时：%d ms", 
                getCurrentTime(), taskName, duration));
                
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            System.err.println(String.format("[%s] %s 执行失败，耗时：%d ms，错误：%s", 
                getCurrentTime(), taskName, duration, e.getMessage()));
        }
    }
    
    private String getCurrentTime() {
        return LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    }
    
    private void performBusinessLogic() {
        // 模拟业务处理
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### 2. 任务异常处理

```java
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Retryable;

@Component
public class RobustScheduledTask {
    
    private static final int MAX_RETRY_ATTEMPTS = 3;
    
    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨2点执行
    public void dailyTask() {
        try {
            executeWithRetry();
        } catch (Exception e) {
            // 最终失败后的处理
            handleFinalFailure(e);
        }
    }
    
    @Retryable(value = {Exception.class}, maxAttempts = MAX_RETRY_ATTEMPTS, 
               backoff = @Backoff(delay = 1000, multiplier = 2))
    public void executeWithRetry() throws Exception {
        System.out.println("尝试执行任务...");
        
        // 模拟可能失败的操作
        if (Math.random() < 0.7) { // 70%的概率失败
            throw new RuntimeException("模拟任务执行失败");
        }
        
        System.out.println("任务执行成功");
    }
    
    private void handleFinalFailure(Exception e) {
        System.err.println("任务最终执行失败：" + e.getMessage());
        // 发送告警通知
        sendAlertNotification(e);
    }
    
    private void sendAlertNotification(Exception e) {
        System.out.println("发送告警通知：定时任务执行失败 - " + e.getMessage());
        // 实际的告警逻辑，如发送邮件、短信等
    }
}
```

### 3. 分布式锁防止重复执行

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import java.util.concurrent.TimeUnit;

@Component
public class DistributedLockScheduledTask {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    private static final String LOCK_KEY = "scheduled_task_lock";
    private static final int LOCK_TIMEOUT = 60; // 锁超时时间（秒）
    
    @Scheduled(fixedRate = 30000) // 每30秒执行一次
    public void distributedTask() {
        String lockValue = String.valueOf(System.currentTimeMillis());
        
        try {
            // 尝试获取分布式锁
            if (acquireLock(lockValue)) {
                System.out.println("获取锁成功，开始执行任务...");
                executeBusinessLogic();
                System.out.println("任务执行完成");
            } else {
                System.out.println("获取锁失败，跳过本次执行");
            }
        } finally {
            // 释放锁
            releaseLock(lockValue);
        }
    }
    
    private boolean acquireLock(String lockValue) {
        return Boolean.TRUE.equals(redisTemplate.opsForValue()
            .setIfAbsent(LOCK_KEY, lockValue, LOCK_TIMEOUT, TimeUnit.SECONDS));
    }
    
    private void releaseLock(String lockValue) {
        // 使用Lua脚本确保原子性释放锁
        String luaScript = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                          "return redis.call('del', KEYS[1]) else return 0 end";
        redisTemplate.execute((connection) -> 
            connection.eval(luaScript.getBytes(), 1, 
                LOCK_KEY.getBytes(), lockValue.getBytes()));
    }
    
    private void executeBusinessLogic() {
        System.out.println("执行分布式任务业务逻辑...");
        // 模拟业务处理时间
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### 4. 配置建议

```yaml
# application.yml - Spring Boot应用配置
spring:
  task:
    scheduling:
      pool:
        size: 10  # 线程池大小
      thread-name-prefix: scheduled-task-
    execution:
      pool:
        core-size: 5
        max-size: 15
        queue-capacity: 100

# 数据库连接池配置（用于Quartz持久化）
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5

# Redis配置（用于分布式锁）
  redis:
    host: localhost
    port: 6379
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0
```

### 5. 监控和告警

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;

@Component
public class ScheduledTaskHealthIndicator implements HealthIndicator {
    
    private LocalDateTime lastSuccessTime;
    private LocalDateTime lastFailureTime;
    private String lastError;
    
    @Override
    public Health health() {
        if (lastSuccessTime == null) {
            return Health.down()
                .withDetail("status", "任务尚未执行")
                .build();
        }
        
        long minutesSinceLastSuccess = ChronoUnit.MINUTES.between(lastSuccessTime, LocalDateTime.now());
        
        if (minutesSinceLastSuccess > 10) { // 10分钟内没有成功执行
            return Health.down()
                .withDetail("status", "任务执行超时")
                .withDetail("lastSuccessTime", lastSuccessTime)
                .withDetail("minutesSinceLastSuccess", minutesSinceLastSuccess)
                .build();
        }
        
        return Health.up()
            .withDetail("status", "正常")
            .withDetail("lastSuccessTime", lastSuccessTime)
            .withDetail("lastFailureTime", lastFailureTime)
            .withDetail("lastError", lastError)
            .build();
    }
    
    public void recordSuccess() {
        this.lastSuccessTime = LocalDateTime.now();
    }
    
    public void recordFailure(String error) {
        this.lastFailureTime = LocalDateTime.now();
        this.lastError = error;
    }
}
```

### 总结

选择合适的定时任务技术方案需要考虑以下因素：

1. **项目规模**：小型项目可以使用Spring @Scheduled，大型项目建议使用Quartz或分布式方案
2. **集群需求**：集群环境必须使用支持分布式的方案
3. **管理需求**：需要可视化管理的选择XXL-JOB或Elastic-Job  
4. **性能要求**：高性能场景选择Quartz或分布式方案
5. **维护成本**：考虑团队技术栈和维护能力

在实际使用中，还要注意异常处理、监控告警、分布式锁等最佳实践，确保定时任务的稳定可靠运行。