---
title: Quartz实现持久化
date: 2018-05-29 19:26:22
tags: ['Quartz']
---

最近项目中出现需求为：给定参数和时间，定时发送短信。同时希望如果服务器挂了能够重新运行这个定时任务。

准备使用quartz做。

所有代码在[GitHub](https://github.com/qianhangkang/justdoit/tree/master/quartz)上，帮助到你就点个赞呗~

quartz的任务存储（Job Stores）模式有三种：RAMJobStore、JDBCJobStore、TerracottaJobStore

<!--more-->



# 三种任务存储模式

##  RAMJobStore

最简单的、性能最高的、也是**默认**的一种jobstore，所有数据存放在内存中，如果程序重启或关闭，所有的job都会丢失。

##  JDBCJobStore

性能比RAMJobStore差，所有数据存放在数据库中， JDBCJobStore几乎适用于任何数据库。要使用JDBCJobStore，您必须首先为Quartz创建一组数据库表以供使用。当然，你**不需要**实现任何一条SQL语句。

在配置和启动JDBCJobStore之前，您需要确定应用程序需要什么类型的事务。

- 如果您不需要将调度命令（例如添加和删除触发器）与其他事务绑定在一起，那么您可以让Quartz使用**JobStoreTX**作为JobStore（这是最常见的选择）来管理事务。
- 如果您需要Quartz与其他事务一起工作（即在J2EE应用程序服务器中），那么您应该使用**JobStoreCMT--**在这种情况下，Quartz将让应用程序服务器容器管理事务。



##  TerracottaJobStore

TerracottaJobStore是Quartz 1.7的新成员。它提供了一种扩展和健壮性的方法，而不需要使用数据库。这意味着您的数据库可以从Quartz中免费加载，并且可以将其所有资源保存到其他应用程序中。



# 原生持久化解决方案

> IDE：idea
>
> jdk：1.8
>
> 框架：springboot (v2.0.2.RELEASE)、quartz(v2.3.0)、lombok



## quartz持久化需要的表SQL

```sql
# In your Quartz properties file, you will need to set   
# org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate 
# 
# By: Ron Cordell - roncordell
# I didn't see this anywhere, so I thought I'd post it here. 
# This is the script from Quartz to create the tables in a MySQL database,
# modified to use INNODB instead of MYISAM. 
 

DROP TABLE IF EXISTS QRTZ_FIRED_TRIGGERS;  
DROP TABLE IF EXISTS QRTZ_PAUSED_TRIGGER_GRPS;  
DROP TABLE IF EXISTS QRTZ_SCHEDULER_STATE;  
DROP TABLE IF EXISTS QRTZ_LOCKS;  
DROP TABLE IF EXISTS QRTZ_SIMPLE_TRIGGERS;  
DROP TABLE IF EXISTS QRTZ_SIMPROP_TRIGGERS;  
DROP TABLE IF EXISTS QRTZ_CRON_TRIGGERS;  
DROP TABLE IF EXISTS QRTZ_BLOB_TRIGGERS;  
DROP TABLE IF EXISTS QRTZ_TRIGGERS;  
DROP TABLE IF EXISTS QRTZ_JOB_DETAILS;  
DROP TABLE IF EXISTS QRTZ_CALENDARS;  
  
CREATE TABLE QRTZ_JOB_DETAILS(  
SCHED_NAME VARCHAR(120) NOT NULL,  
JOB_NAME VARCHAR(200) NOT NULL,  
JOB_GROUP VARCHAR(200) NOT NULL,  
DESCRIPTION VARCHAR(250) NULL,  
JOB_CLASS_NAME VARCHAR(250) NOT NULL,  
IS_DURABLE VARCHAR(1) NOT NULL,  
IS_NONCONCURRENT VARCHAR(1) NOT NULL,  
IS_UPDATE_DATA VARCHAR(1) NOT NULL,  
REQUESTS_RECOVERY VARCHAR(1) NOT NULL,  
JOB_DATA BLOB NULL,  
PRIMARY KEY (SCHED_NAME,JOB_NAME,JOB_GROUP))  
ENGINE=InnoDB;  
  
CREATE TABLE QRTZ_TRIGGERS (  
SCHED_NAME VARCHAR(120) NOT NULL,  
TRIGGER_NAME VARCHAR(200) NOT NULL,  
TRIGGER_GROUP VARCHAR(200) NOT NULL,  
JOB_NAME VARCHAR(200) NOT NULL,  
JOB_GROUP VARCHAR(200) NOT NULL,  
DESCRIPTION VARCHAR(250) NULL,  
NEXT_FIRE_TIME BIGINT(13) NULL,  
PREV_FIRE_TIME BIGINT(13) NULL,  
PRIORITY INTEGER NULL,  
TRIGGER_STATE VARCHAR(16) NOT NULL,  
TRIGGER_TYPE VARCHAR(8) NOT NULL,  
START_TIME BIGINT(13) NOT NULL,  
END_TIME BIGINT(13) NULL,  
CALENDAR_NAME VARCHAR(200) NULL,  
MISFIRE_INSTR SMALLINT(2) NULL,  
JOB_DATA BLOB NULL,  
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),  
FOREIGN KEY (SCHED_NAME,JOB_NAME,JOB_GROUP)  
REFERENCES QRTZ_JOB_DETAILS(SCHED_NAME,JOB_NAME,JOB_GROUP))  
ENGINE=InnoDB;  
  
CREATE TABLE QRTZ_SIMPLE_TRIGGERS (  
SCHED_NAME VARCHAR(120) NOT NULL,  
TRIGGER_NAME VARCHAR(200) NOT NULL,  
TRIGGER_GROUP VARCHAR(200) NOT NULL,  
REPEAT_COUNT BIGINT(7) NOT NULL,  
REPEAT_INTERVAL BIGINT(12) NOT NULL,  
TIMES_TRIGGERED BIGINT(10) NOT NULL,  
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),  
FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)  
REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))  
ENGINE=InnoDB;  
  
CREATE TABLE QRTZ_CRON_TRIGGERS (  
SCHED_NAME VARCHAR(120) NOT NULL,  
TRIGGER_NAME VARCHAR(200) NOT NULL,  
TRIGGER_GROUP VARCHAR(200) NOT NULL,  
CRON_EXPRESSION VARCHAR(120) NOT NULL,  
TIME_ZONE_ID VARCHAR(80),  
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),  
FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)  
REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))  
ENGINE=InnoDB;  
  
CREATE TABLE QRTZ_SIMPROP_TRIGGERS  
  (            
    SCHED_NAME VARCHAR(120) NOT NULL,  
    TRIGGER_NAME VARCHAR(200) NOT NULL,  
    TRIGGER_GROUP VARCHAR(200) NOT NULL,  
    STR_PROP_1 VARCHAR(512) NULL,  
    STR_PROP_2 VARCHAR(512) NULL,  
    STR_PROP_3 VARCHAR(512) NULL,  
    INT_PROP_1 INT NULL,  
    INT_PROP_2 INT NULL,  
    LONG_PROP_1 BIGINT NULL,  
    LONG_PROP_2 BIGINT NULL,  
    DEC_PROP_1 NUMERIC(13,4) NULL,  
    DEC_PROP_2 NUMERIC(13,4) NULL,  
    BOOL_PROP_1 VARCHAR(1) NULL,  
    BOOL_PROP_2 VARCHAR(1) NULL,  
    PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),  
    FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)   
    REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))  
ENGINE=InnoDB;  
  
CREATE TABLE QRTZ_BLOB_TRIGGERS (  
SCHED_NAME VARCHAR(120) NOT NULL,  
TRIGGER_NAME VARCHAR(200) NOT NULL,  
TRIGGER_GROUP VARCHAR(200) NOT NULL,  
BLOB_DATA BLOB NULL,  
PRIMARY KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP),  
INDEX (SCHED_NAME,TRIGGER_NAME, TRIGGER_GROUP),  
FOREIGN KEY (SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP)  
REFERENCES QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP))  
ENGINE=InnoDB;  
  
CREATE TABLE QRTZ_CALENDARS (  
SCHED_NAME VARCHAR(120) NOT NULL,  
CALENDAR_NAME VARCHAR(200) NOT NULL,  
CALENDAR BLOB NOT NULL,  
PRIMARY KEY (SCHED_NAME,CALENDAR_NAME))  
ENGINE=InnoDB;  
  
CREATE TABLE QRTZ_PAUSED_TRIGGER_GRPS (  
SCHED_NAME VARCHAR(120) NOT NULL,  
TRIGGER_GROUP VARCHAR(200) NOT NULL,  
PRIMARY KEY (SCHED_NAME,TRIGGER_GROUP))  
ENGINE=InnoDB;  
  
CREATE TABLE QRTZ_FIRED_TRIGGERS (  
SCHED_NAME VARCHAR(120) NOT NULL,  
ENTRY_ID VARCHAR(95) NOT NULL,  
TRIGGER_NAME VARCHAR(200) NOT NULL,  
TRIGGER_GROUP VARCHAR(200) NOT NULL,  
INSTANCE_NAME VARCHAR(200) NOT NULL,  
FIRED_TIME BIGINT(13) NOT NULL,  
SCHED_TIME BIGINT(13) NOT NULL,  
PRIORITY INTEGER NOT NULL,  
STATE VARCHAR(16) NOT NULL,  
JOB_NAME VARCHAR(200) NULL,  
JOB_GROUP VARCHAR(200) NULL,  
IS_NONCONCURRENT VARCHAR(1) NULL,  
REQUESTS_RECOVERY VARCHAR(1) NULL,  
PRIMARY KEY (SCHED_NAME,ENTRY_ID))  
ENGINE=InnoDB;  
  
CREATE TABLE QRTZ_SCHEDULER_STATE (  
SCHED_NAME VARCHAR(120) NOT NULL,  
INSTANCE_NAME VARCHAR(200) NOT NULL,  
LAST_CHECKIN_TIME BIGINT(13) NOT NULL,  
CHECKIN_INTERVAL BIGINT(13) NOT NULL,  
PRIMARY KEY (SCHED_NAME,INSTANCE_NAME))  
ENGINE=InnoDB;  
  
CREATE TABLE QRTZ_LOCKS (  
SCHED_NAME VARCHAR(120) NOT NULL,  
LOCK_NAME VARCHAR(40) NOT NULL,  
PRIMARY KEY (SCHED_NAME,LOCK_NAME))  
ENGINE=InnoDB;  
  
CREATE INDEX IDX_QRTZ_J_REQ_RECOVERY ON QRTZ_JOB_DETAILS(SCHED_NAME,REQUESTS_RECOVERY);  
CREATE INDEX IDX_QRTZ_J_GRP ON QRTZ_JOB_DETAILS(SCHED_NAME,JOB_GROUP);  
  
CREATE INDEX IDX_QRTZ_T_J ON QRTZ_TRIGGERS(SCHED_NAME,JOB_NAME,JOB_GROUP);  
CREATE INDEX IDX_QRTZ_T_JG ON QRTZ_TRIGGERS(SCHED_NAME,JOB_GROUP);  
CREATE INDEX IDX_QRTZ_T_C ON QRTZ_TRIGGERS(SCHED_NAME,CALENDAR_NAME);  
CREATE INDEX IDX_QRTZ_T_G ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_GROUP);  
CREATE INDEX IDX_QRTZ_T_STATE ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_STATE);  
CREATE INDEX IDX_QRTZ_T_N_STATE ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP,TRIGGER_STATE);  
CREATE INDEX IDX_QRTZ_T_N_G_STATE ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_GROUP,TRIGGER_STATE);  
CREATE INDEX IDX_QRTZ_T_NEXT_FIRE_TIME ON QRTZ_TRIGGERS(SCHED_NAME,NEXT_FIRE_TIME);  
CREATE INDEX IDX_QRTZ_T_NFT_ST ON QRTZ_TRIGGERS(SCHED_NAME,TRIGGER_STATE,NEXT_FIRE_TIME);  
CREATE INDEX IDX_QRTZ_T_NFT_MISFIRE ON QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME);  
CREATE INDEX IDX_QRTZ_T_NFT_ST_MISFIRE ON QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME,TRIGGER_STATE);  
CREATE INDEX IDX_QRTZ_T_NFT_ST_MISFIRE_GRP ON QRTZ_TRIGGERS(SCHED_NAME,MISFIRE_INSTR,NEXT_FIRE_TIME,TRIGGER_GROUP,TRIGGER_STATE);  
  
CREATE INDEX IDX_QRTZ_FT_TRIG_INST_NAME ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,INSTANCE_NAME);  
CREATE INDEX IDX_QRTZ_FT_INST_JOB_REQ_RCVRY ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,INSTANCE_NAME,REQUESTS_RECOVERY);  
CREATE INDEX IDX_QRTZ_FT_J_G ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,JOB_NAME,JOB_GROUP);  
CREATE INDEX IDX_QRTZ_FT_JG ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,JOB_GROUP);  
CREATE INDEX IDX_QRTZ_FT_T_G ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,TRIGGER_NAME,TRIGGER_GROUP);  
CREATE INDEX IDX_QRTZ_FT_TG ON QRTZ_FIRED_TRIGGERS(SCHED_NAME,TRIGGER_GROUP);  
  
commit;
```



## pom文件

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.hongkong</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>demo</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.2.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!--quartz-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-quartz</artifactId>
        </dependency>
        <!--mysql-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <!--lombok-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <!--mybatis，提供DataSource，也可以使用springboot的jdbc，如下-->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>

        <!--&lt;!&ndash;springboot的jdbc&ndash;&gt;-->
        <!--<dependency>-->
            <!--<groupId>org.springframework.boot</groupId>-->
            <!--<artifactId>spring-boot-starter-jdbc</artifactId>-->
        <!--</dependency>-->
        <!--<dependency>-->
            <!--<groupId>com.h2database</groupId>-->
            <!--<artifactId>h2</artifactId>-->
        <!--</dependency>-->

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>

```



## Java代码设置quartz配置

```java
/**
 * quartz的Java配置文件
 */
@Configuration
@Slf4j
public class QuartzConfig {

    @Autowired
    private QuartzListener listener;

    @Bean
    public SchedulerFactoryBean schedulerFactoryBean(DataSource dataSource) {
        SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
        schedulerFactoryBean.setSchedulerName("MsmJobSchedule");
        //设置数据库数据源
        schedulerFactoryBean.setDataSource(dataSource);

        //quartz参数
        Properties prop = new Properties();
        prop.put("org.quartz.scheduler.instanceName", "MsmJobSchedule");
        prop.put("org.quartz.scheduler.instanceId", "AUTO");
        //线程池配置
        prop.put("org.quartz.threadPool.class", "org.quartz.simpl.SimpleThreadPool");
        //并行的线程数，默认为10
        prop.put("org.quartz.threadPool.threadCount", "10");
        //线程优先级，默认为5
        prop.put("org.quartz.threadPool.threadPriority", "5");
        //JobStore配置
        prop.put("org.quartz.jobStore.class", "org.quartz.impl.jdbcjobstore.JobStoreTX");
        //配置是否启动自动加载数据库内的定时任务，默认true
        prop.put("org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread","true");
        //持久化方式配置数据驱动，MySQL数据库
        prop.put("org.quartz.jobStore.driverDelegateClass","org.quartz.impl.jdbcjobstore.StdJDBCDelegate");
        //设置property
        schedulerFactoryBean.setQuartzProperties(prop);
        //延时启动10s
        schedulerFactoryBean.setStartupDelay(10);
        return schedulerFactoryBean;
    }

    @Bean
    public Scheduler scheduler(DataSource dataSource) {
        Scheduler scheduler = schedulerFactoryBean(dataSource).getScheduler();
        try {
            //每次schedule初始化时添加监听器，否则程序重启后之前的任务无法被监听
            log.info("添加监听器至schedule");
            scheduler.getListenerManager().addJobListener(listener);//listener自动注入
        } catch (SchedulerException e) {
            log.debug("监听器添加至schedule失败");
            e.printStackTrace();
        }
        return scheduler;
    }
}
```



## 实现Job接口

```java
@Slf4j
public class MyJob implements Job {

    /**
     * Job执行的方法
     * @param context job上下文
     * @throws JobExecutionException
     */
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        log.info("任务{}开始执行任务",context.getFireInstanceId());
        //得到传入job的参数
        JobDataMap dataMap = context.getJobDetail().getJobDataMap();
        try {
            Thread.sleep(time);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("任务{}执行结束",context.getFireInstanceId());
        context.setResult(String.format("%s say 'hello' to listener"));
    }
}
```



## 实现JobListener接口

```java
@Slf4j
@Component
public class QuartzListener implements JobListener {
    @Override
    public String getName() {
        return "QuartzListener";
    }

    @Override
    public void jobToBeExecuted(JobExecutionContext context) {
        //监听job执行之前

    }

    @Override
    public void jobExecutionVetoed(JobExecutionContext context) {
        //监听job的trigger被触发，但是job被JobListener禁止执行事件

    }

    @Override
    public void jobWasExecuted(JobExecutionContext context, JobExecutionException jobException) {
        //监听job执行完成后
        log.info("监听器收到job执行结果:{}", context.getResult());//打印job执行结果
    }
}

```



## 向schedule添加自定义MyJob

```java
@Service
@Slf4j
public class QuartzService {

    @Autowired
    private Scheduler scheduler;

    @Autowired
    private QuartzListener listener;//监听器

    /**
     * 添加一个定时任务
     * @param parameter 传入任务的参数
     * @param startDate 执行的时间
     */
    public void addJob(String parameter,Date startDate) {
        //设置一个标识符
        String id = UUID.randomUUID().toString();
        //设置传入job的参数
        JobDataMap dataMap = new JobDataMap();
        dataMap.put("parameter", parameter);
        //jobDetail
        JobDetail jobDetail = JobBuilder.newJob(MyJob.class).withIdentity("job---"+id,id).setJobData(dataMap).build();
        //trigger
        Trigger trigger = TriggerBuilder.newTrigger().withIdentity("trigger---"+id,id).startAt(startDate).build();
        try {
            scheduler.scheduleJob(jobDetail,trigger);
            log.info("job---{}加入任务",id);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }
}

```



## controller层

```java
@RestController
@Slf4j
public class QuartzController {
    @Autowired
    private QuartzService service;

    @Autowired
    private Scheduler scheduler;

    /**
     * 添加一个任务，30s后执行
     * @param parameter 传入的参数
     * @return
     */
    @RequestMapping(value = "/add/{parameter}")
    public String add(@PathVariable(value = "parameter") String parameter) {
        //设置30s后执行
        Date now = new Date();
        log.info("现在时间为{}",now);
        Calendar calendar = Calendar.getInstance();
        calendar.setTime(now);
        calendar.add(Calendar.SECOND, 30);
        //执行时间
        Date startDate = calendar.getTime();
        log.info("任务执行的时间为{}", startDate);
        service.addJob(parameter, startDate);
        return "add successfully";
    }

    /**
     * 得到当前任务执行的列表
     * @return
     */
    @RequestMapping(value = "/job")
    public List job() {
        List<String> res = new ArrayList<>();
        List<Trigger> triggers = null;
        String jobName = null;
        String jobGroup = null;
        Date nextFireTime = null;
        try {
            for (String groupName : scheduler.getJobGroupNames()) {
                for (JobKey jobKey : scheduler.getJobKeys(GroupMatcher.jobGroupEquals(groupName))) {
                    jobName = jobKey.getName();
                    jobGroup = jobKey.getGroup();
                    //get trigger
                    triggers = (List<Trigger>) scheduler.getTriggersOfJob(jobKey);
                    nextFireTime = triggers.get(0).getNextFireTime();
                    res.add("[jobName] : " + jobName + " [groupName] : "
                            + jobGroup + " - " + nextFireTime);
                }
            }
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
        return res;
    }
}
```



## 结果演示

### 通过接口添加一个job

参数：helloworld~~ 

![](https://i.loli.net/2018/05/30/5b0e0f8db01b3.png) 

![](https://i.loli.net/2018/05/30/5b0e0f8dbff34.png)

### 查看数据库

QRTZ_JOB_DETAILS表

![](https://i.loli.net/2018/05/30/5b0e0f8db9358.png)

QRTZ_SIMPLE_TRIGGERS表

![](https://i.loli.net/2018/05/30/5b0e0f8dba9c3.png)

### 通过接口查看待执行的任务

![](https://i.loli.net/2018/05/30/5b0e0f8dc1956.png)

### 一分钟后任务执行

![](https://i.loli.net/2018/05/30/5b0e0f8dc2fff.png)



## 监听器存在的一个问题

如果不在schedule初始化时将监听器添加进去，那么会出现应用程序重启后会无法监听之前job的情况。



# 自定义解决方案

上流程图

![](https://i.loli.net/2018/05/31/5b0fb1ca774b7.png)

## 我的自定义的quartz持久化表

仅供参考，请自行修改

```mysql
# author:hongkong
# date:2018年05月31日15:57:51

CREATE TABLE `PTP_MSM_TASK` (
  `ID` int(11) NOT NULL AUTO_INCREMENT COMMENT '任务id',
  `JOB_NAME` varchar(40) DEFAULT NULL COMMENT '任务名',
  `PARAMETER` varchar(2000) DEFAULT NULL COMMENT '短信参数',
  `EXECUTE_DATE` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '执行日期',
  `STATUS` int(11) NOT NULL COMMENT '执行状态，0--未执行，1--执行成功，2--执行失败',
  `CRON` varchar(40) DEFAULT NULL COMMENT 'cron表达式',
  `TYPE` INT DEFAULT 0 NOT NULL COMMENT '任务执行类型，0--立即执行，1--延时执行一次，2--利用cron表达式执行'
  PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='任务表';
```



## pom文件

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.hongkong</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>demo</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.2.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-quartz</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
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

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```



## Java代码设置quartz配置

```java
/**
 * quartz的Java配置文件
 */
@Configuration
@Slf4j
public class QuartzConfig {

    @Autowired
    private QuartzListener listener;

    @Bean
    public SchedulerFactoryBean schedulerFactoryBean(DataSource dataSource) {
        SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
        schedulerFactoryBean.setSchedulerName("MsmJobSchedule");
        //quartz参数
        Properties prop = new Properties();
        prop.put("org.quartz.scheduler.instanceName", "MsmJobSchedule");
        prop.put("org.quartz.scheduler.instanceId", "AUTO");
        //设置property
        schedulerFactoryBean.setQuartzProperties(prop);
        //延时启动10s
        schedulerFactoryBean.setStartupDelay(10);
        return schedulerFactoryBean;
    }

    @Bean
    public Scheduler scheduler(DataSource dataSource) {
        Scheduler scheduler = schedulerFactoryBean(dataSource).getScheduler();
        try {
            //每次schedule初始化时添加监听器，否则程序重启后之前的任务无法被监听
            log.info("添加监听器至schedule");
            scheduler.getListenerManager().addJobListener(listener);//listener自动注入
        } catch (SchedulerException e) {
            log.debug("监听器添加至schedule失败");
            e.printStackTrace();
        }
        return scheduler;
    }
}

```



## 实现job接口

```java
    /**
     * Job执行的方法
     * @param context job上下文
     */
    @Override
    public void execute(JobExecutionContext context) {
        //得到传入job的参数
        JobDataMap dataMap = context.getJobDetail().getJobDataMap();
        String parameter = dataMap.getString("parameter");
        //quartz自动生成的唯一id
        String instanceId = context.getFireInstanceId();
        log.info("任务{}，参数为{}开始执行任务", instanceId, parameter);

        try {
            int time = random.nextInt(5000) + 1000;
            Thread.sleep(time);
            log.info("任务{}，参数为{}，时间为{}执行结束", instanceId, parameter, time);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```



## 实现JobListener接口

```java
@Slf4j
@Component
public class QuartzListener implements JobListener {
    @Autowired
    private PtpMsmTaskRepository repository;

    @Override
    public String getName() {
        return "QuartzListener";
    }

    @Override
    public void jobToBeExecuted(JobExecutionContext context) {
        //监听job执行之前

    }

    @Override
    public void jobExecutionVetoed(JobExecutionContext context) {
        //监听job的trigger被触发，但是job被JobListener禁止执行事件

    }

    @Override
    public void jobWasExecuted(JobExecutionContext context, JobExecutionException jobException) {
        /*
         * 得到任务的name
         * 根据name查找数据库
         * 若不存在记录，记录异常
         * 否则，判断传入的异常是否为空，更新对应的任务状态
         */
        String name = context.getJobDetail().getKey().getName();
        PtpMsmTask job = repository.findOneByName(name);
        if (Objects.isNull(job)) {
            log.debug("数据库中无法查询到name为{}的任务",name);
            return;
        }
        //存在记录，判断是否执行时抛出异常
        if (Objects.isNull(jobException)) {
            //执行成功，没有异常
            job.setStatus(TaskStatusEnum.SUCCESS.getType());
            int count = repository.updateByName(job,name);
            if (count == 0) {
                log.debug("更新job{}失败", job);
                return;
            }
            log.info("任务为{}，描述为{}的任务执行完成",name, context.getJobDetail().getDescription());
            return;
        }
        //执行时发生异常
        job.setStatus(TaskStatusEnum.FAIL.getType());
        int count = repository.updateByName(job,name);
        if (count == 0) {
            log.debug("更新job{}失败", job);
            return;
        }
        log.info("任务为{}，描述为{}的任务执行发生异常{}",name, context.getJobDetail().getDescription(),jobException.getMessage());
    }
}
```



## 添加job

```java
@Service
@Slf4j
public class QuartzService {
    @Autowired
    private Scheduler scheduler;

    @Autowired
    private PtpMsmTaskRepository repository;

    /**
     * schedule被注入后执行的初始化函数
     */
    @PostConstruct
    public void init() {
        List<PtpMsmTask> list = repository.findListByStatus(TaskStatusEnum.NOT_PERFORME.getType());
        if (Objects.isNull(list) || list.isEmpty()) {
            log.info("{}时间点之前不存在未执行的任务",new Date());
            return;
        }
        //将数据库中未执行的任务重新加入至schedule中
        String name = null;
        Date startDate = null;
        JobDetail jobDetail = null;
        Trigger trigger = null;
        for (PtpMsmTask job : list) {
            JobDataMap dataMap = new JobDataMap();
            dataMap.put("parameter", job.getParameter());
            name = job.getJobName();
            startDate = job.getExecuteDate();
            jobDetail = JobBuilder.newJob(MyJob.class).withIdentity(name,name).setJobData(dataMap).build();
            trigger = TriggerBuilder.newTrigger().withIdentity(name,name).startAt(startDate).build();
            try {
                scheduler.scheduleJob(jobDetail,trigger);
                log.info("job---{}加入至schedule",name);
            } catch (SchedulerException e) {
                e.printStackTrace();
            }
        }
    }
    /**
     * 添加一个定时任务
     * @param parameter 传入任务的参数
     * @param startDate 执行的时间
     */
    public void addJob(String parameter,Date startDate) {
        //设置一个标识符
        String id = UUID.randomUUID().toString();
        //设置传入job的参数
        JobDataMap dataMap = new JobDataMap();
        dataMap.put("parameter", parameter);
        //jobDetail
        JobDetail jobDetail = JobBuilder.newJob(MyJob.class)
                .withIdentity(id,id)
                .setJobData(dataMap)
                .build();
        //trigger
        Trigger trigger = TriggerBuilder.newTrigger()
                .withIdentity(id,id)
                .startAt(startDate)
                .build();
        //写入数据库
        PtpMsmTask job = PtpMsmTask.builder()
                .jobName(id)
                .parameter(parameter)
                .executeDate(startDate)
                .status(TaskStatusEnum.NOT_PERFORME.getType())
                .type(TaskTypeEnum.LATER.getType())
                .build();
        long count = repository.insert(job);
        if (count == 0) {
            log.info("任务{}写入数据库失败，任务不执行", job);
            return;
        }
        try {
            scheduler.scheduleJob(jobDetail,trigger);
            log.info("job---{}加入至schedule",id);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }

    /**
     * 添加立即执行的任务
     * @param parameter
     */
    public void add(String parameter) {
        //设置一个标识符
        String id = UUID.randomUUID().toString();
        //设置传入job的参数
        JobDataMap dataMap = new JobDataMap();
        dataMap.put("parameter", parameter);
        //jobDetail
        JobDetail jobDetail = JobBuilder.newJob(MyJob.class).withIdentity(id,id).setJobData(dataMap).build();
        //trigger
        Trigger trigger = TriggerBuilder.newTrigger().withIdentity(id,id).startNow().build();
        //写入数据库
        PtpMsmTask job = PtpMsmTask.builder()
                .jobName(id)
                .parameter(parameter)
                .executeDate(new Date())
                .status(TaskStatusEnum.NOT_PERFORME.getType())
                .type(TaskTypeEnum.NOW.getType())
                .build();
        long count = repository.insert(job);
        if (count == 0) {
            log.info("任务{}写入数据库失败，任务不执行", job);
            return;
        }
        try {
            scheduler.scheduleJob(jobDetail,trigger);
            log.info("job---{}加入至schedule",id);
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
    }
}
```



## controller层

```java
@RestController
@Slf4j
public class QuartzController {
    @Autowired
    private QuartzService service;

    @Autowired
    private Scheduler scheduler;

    /**
     * 添加一个任务，60s后执行
     * @param parameter 传入的参数
     * @return
     */
    @RequestMapping(value = "/add/{parameter}")
    public String add(@PathVariable(value = "parameter") String parameter) {
        //设置60s后执行
        Date now = new Date();
        log.info("现在时间为{}",now);
        Calendar calendar = Calendar.getInstance();
        calendar.setTime(now);
        calendar.add(Calendar.SECOND, 60);
        //执行时间
        Date startDate = calendar.getTime();
        log.info("任务执行的时间为{}", startDate);
        service.addJob(parameter, startDate);
        return "add successfully";
    }

    /**
     * 得到当前任务执行的列表
     * @return
     */
    @RequestMapping(value = "/job")
    public List job() {
        List<String> res = new ArrayList<>();
        try {
            for (String groupName : scheduler.getJobGroupNames()) {
                for (JobKey jobKey : scheduler.getJobKeys(GroupMatcher.jobGroupEquals(groupName))) {
                    String jobName = jobKey.getName();
                    String jobGroup = jobKey.getGroup();
                    //get trigger
                    List<Trigger> triggers = (List<Trigger>) scheduler.getTriggersOfJob(jobKey);
                    Date nextFireTime = triggers.get(0).getNextFireTime();
                    res.add("[jobName] : " + jobName + ",[groupName] : "
                            + jobGroup + " - " + nextFireTime);
                }
            }
        } catch (SchedulerException e) {
            e.printStackTrace();
        }
        return res;
    }

    /**
     * 添加自定义数量的job
     * @param count job的数量
     */
    @RequestMapping(value = "test/{count}")
    public void test(@PathVariable(value = "count") int count) {
        Calendar calendar = Calendar.getInstance();

        for (int i = 0; i < count; i++) {
            calendar.setTime(new Date());
            calendar.add(Calendar.SECOND, 10);
            service.addJob(Integer.toString(i),calendar.getTime());
        }
    }
}

```



# 两种方案比较

## quartz原生持久化

1. 优点

   - 支持集群
   - 原生支持，不需要写SQL
   - 任务恢复更有保证

2. 缺点

   - 需要是一张表，且表与表之间存在外键关联

   - 运行速度的快慢取决于连接数据库的快慢

   - 历史记录需要自己重新建表记录

   - 如果数据库中存在大量过去未执行的任务（假设有5w条），当程序重启时会马上执行过期的任务，此时会出现一个我暂时无法解决的异常:( ，异常参考[链接](https://dzone.com/articles/quartz-scheduler-misfire) 

   - ```
     Handling the first 20 triggers that missed their scheduled fire-time.  More misfired triggers remain to be processed.
     ```

## 自定义持久化

1. 优点

   - 可控性较高、较为简单
   - 数据库只需要增加一张表

   ```mysql
   CREATE TABLE `PTP_MSM_TASK` (
     `ID` int(11) NOT NULL AUTO_INCREMENT COMMENT '任务id',
     `JOB_NAME` varchar(40) DEFAULT NULL COMMENT '任务名',
     `PARAMETER` varchar(2000) DEFAULT NULL COMMENT '传入参数',
     `EXECUTE_DATE` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '执行日期',
     `STATUS` int(11) NOT NULL COMMENT '执行状态，0--未执行，1--执行成功，2--执行失败',
     `CRON` varchar(40) DEFAULT NULL COMMENT 'cron表达式',
     `TYPE` INT DEFAULT 0 NOT NULL COMMENT '任务执行类型，0--立即执行，1--延时执行一次，2--利用cron表达式执行'
     PRIMARY KEY (`ID`)
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='任务表';
   ```

   

2. 缺点

   - 不支持集群（可以自己添加支持）
   - 运行速度的快慢取决于连接数据库的快慢
   - 因为只有一张表，记录的信息较少，如果系统挂了可能有些任务无法恢复













# 参考链接

http://www.quartz-scheduler.org/documentation/quartz-2.x/tutorials/tutorial-lesson-09.html

https://juejin.im/post/5abb7af36fb9a028cb2dae9a




























> 大部分正义感都是虚伪的 聊胜于无   —————————————————————————沃德彭尤·胡某