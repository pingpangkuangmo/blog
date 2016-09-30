# 1 目录

-	[Timer和ScheduledThreadPoolExecutor的定时任务](http://my.oschina.net/pingpangkuangmo/blog/745704)

# 2 调度概述

-	1 说到调度，有最简单的Timer、ScheduledThreadPoolExecutor，又有Spring Task、quartz。

-	2 说到分布式调度，有基于数据库实现的quartz集群方案、当当网开源的基于ZooKeeper的elastic-job

-	3 说到大数据方向的调度，如apache的oozie、阿里的zeus。他们更多是定制大数据方向的job，如MapReduce job，spark job，hive job等，还有解决了上面都没触碰的job依赖管理。

-	4 说到集群资源的调度，如Yarn、Mesos。他们则是更高度的抽象，他们把调度拆分成**资源调度**和**任务调度**，他们只负责资源方面的调度。

	**资源调度**：仅仅负责对集群所有机器的CPU 、内存、网络等资源进行统一规划和管理，有任务到来时，通过合理的分配（达到资源充分利用的目的）挑选出对应资源交付给任务来执行。

	**任务调度**：就要提到任务模型了，如MapReduce是一种任务模型，每一种任务模型有有对应的ApplicationMaster来负责任务调度，如MapReduce的就是MRAppMaster。MRAppMaster负责任务的分片，任务的failover，任务之间的逻辑、向Yarn申请资源来执行Map任务或者Reduce任务等等。目前Yarn已经集成了对MapReduce、Spark等任务模型的处理，如果你还想在Yarn上运行自己的任务模型，就需要实现一个自己的ApplicationMaster来负责任务调度。

# 3 quartz

## 3.1 使用案例

	SchedulerFactory schedFact = new org.quartz.impl.StdSchedulerFactory();
	Scheduler sched = schedFact.getScheduler();
	sched.start();
	
	// define the job and tie it to our MyJob class
	JobDetail job = JobBuilder.newJob(MyJob.class)
	    .withIdentity("job1", "group1")
	    .build();
	
	// Trigger the job to run now, and then repeat every 40 seconds
	Trigger trigger = TriggerBuilder.newTrigger()
	    .withIdentity("trigger1", "group1")
	    .startNow()
	    .withSchedule(SimpleScheduleBuilder.simpleSchedule()
	    .withIntervalInSeconds(40)
	    .repeatForever())
	    .build();
	
	// Tell quartz to schedule the job using our trigger
	sched.scheduleJob(job, trigger);

就是三个概念

-	JobDetail：job的配置信息
-	Trigger：触发器的配置信息
-	Scheduler：调度器

下面来详细说明

## 3.2 运行原理

# 4 Spring Task

## 4.1 

# 5 quartz的分布式集群

# 5 未完待续

下一篇就来说说quartz的简单模式，Spring Task对定时任务的封装，以及quartz基于数据库的集群模式。