```java

 task项目启动类
@Component(taskJob)
public class TaskJob implements Runnable {
	private static Logger log = Logger.getLogger(TaskJob.class);
	@Resource
	private TaskCronService taskCronServiceImpl;
	private org.quartz.Scheduler quartzScheduler = null;

	public TaskJob() throws SchedulerException {
		log.info(syncJob Construct);
		quartzScheduler = StdSchedulerFactory.getDefaultScheduler();
		quartzScheduler.start();
	}

	@PostConstruct
	public void init() {
		log.info(syncJob init);
		 构造函数完毕，执行启动
		ApplicationContextHelper.addApplicationRunnable(new Runnable() {
			@Override
			public void run() {
				TaskJob.this.run();
			}
		});
	}

	public void run() {
		try {
		     找到后台配置的任务，依次放入调度器里面
			ListTaskCron projects = taskCronServiceImpl.findAll();
			for (TaskCron project  projects) {
				if (project.getStatus() == TaskCronStatus.START.value
						 project.getStatus() == TaskCronStatus.FAILED.value){
					boolean flag= taskCronServiceImpl.modifyProjectStatus(project,
							TaskCronStatus.READY);
					if(!flag){
						continue;
					}
				}
				runProject(project);
			}
		} catch (Exception e) {
			log.error(e.getMessage(), e);
		}
	}

	private void runProject(TaskCron project) {
		try {
		     返回的其实是一个执行单元runnable，并不是quartz的调度器
			Scheduler scheduleJob = createScheduleJob(project);
			 以上皆为封装，此处才是quartz的调度器调度jobdetail和trigger的入口
			addScheduler(scheduleJob);
		} catch (SchedulerException  TaskCronException e) {
			log.error(e.getMessage(), e);
		}
	}

	private Scheduler createScheduleJob(TaskCron project) throws TaskCronException {
	     通过包名和类名反射获取执行类
		String schedulerClassName = project.getScheduler();
		if (StringUtil.isEmpty(schedulerClassName)) {
			throw new TaskCronException(Class name  + schedulerClassName
					+  is empty);
		}
		Class schedulerClass;
		try {
			schedulerClass = Class.forName(schedulerClassName);
			Scheduler scheduleJob = (Scheduler) schedulerClass.newInstance();
			 注入容器并初始化
			ApplicationContextHelper.injectDependencies(scheduleJob);
			scheduleJob.setTaskCron(project);
			return scheduleJob;
		} catch (ClassNotFoundException  InstantiationException
				 IllegalAccessException e) {
			log.error(e.getMessage(), e);
			throw new TaskCronException(Can not create sheduler with
					+ schedulerClassName);
		}
	}

	public Scheduler runProjectNow(TaskCron project) throws SchedulerException,
			TaskCronException {
		if (project.getStatus() == TaskCronStatus.START.value) {
			log.warn(Can not run project  + project.getName()
					+  for status is start);
			return null;
		}
		boolean flag = taskCronServiceImpl.modifyProjectStatus(project, TaskCronStatus.READY);
		if(flag==false){
			log.warn(Can not run project  + project.getName()
					+  for update status fail);
			return null;
		}
		String name = project.getName();
		String namespace = project.getNamespace();
		JobKey jobKey = new JobKey(name, namespace);
		SimpleTrigger trigger = TriggerBuilder.newTrigger()
				.withIdentity(name +  + Long.toString(System.currentTimeMillis()),namespace).startNow()
				.withSchedule(SimpleScheduleBuilder.simpleSchedule())
				.forJob(jobKey).build();
		JobDetail jobDetail = quartzScheduler.getJobDetail(jobKey);
		Scheduler scheduleJob = null;
		if (jobDetail == null) {
			scheduleJob = createScheduleJob(project);
			jobDetail = JobBuilder.newJob(TaskCronJob.class)
					.withIdentity(jobKey).build();
			jobDetail.getJobDataMap().put(TaskCronJob.SCHEDULENAME, scheduleJob);
			quartzScheduler.scheduleJob(jobDetail, trigger);
		} else {
			scheduleJob = (Scheduler) jobDetail.getJobDataMap().get(
					TaskCronJob.SCHEDULENAME);
			quartzScheduler.scheduleJob(trigger);
		}

		 启动
		if (quartzScheduler.isShutdown()) {
			quartzScheduler.start();
		}
		return scheduleJob;
	}

	public boolean interrupt(TaskCron project)
			throws UnableToInterruptJobException {
		if (project.getStatus() == TaskCronStatus.START.value)
			return false;
		JobKey jobKey = new JobKey(project.getName(), project.getNamespace());
		return quartzScheduler.interrupt(jobKey);
	}

    
      quartz的调度器调度jobdetail和trigger
      @param scheduleJob runnable的封装，其实没必要用runnable封装，更多的是一个bean
      @throws SchedulerException quartz依赖包执行调度时抛出的异常
     
	private void addScheduler(Scheduler scheduleJob) throws SchedulerException {
	     后台的配置任务名称，对应了quartz任务的名称
		String name = scheduleJob.getTaskCron().getName();
		 后台配置的组，对应了quartz任务的组
		String namespace = scheduleJob.getTaskCron().getNamespace();
		 构建jobDetail
		JobDetail jobDetail = JobBuilder.newJob(TaskCronJob.class)
				.withIdentity(name, namespace).build();
		 使用JobDataMap传递参数，key是常量，value是执行单元runnable
		jobDetail.getJobDataMap().put(TaskCronJob.SCHEDULENAME, scheduleJob);
		Trigger trigger;
		 构建Trigger，参数会用到后台配置的cron表达式，由scheduleJob执行单元携带该参数
		if (!StringUtil.isEmpty(scheduleJob.getTaskCron().getCron())) {
			TriggerBuilderCronTrigger triggerBuilder = TriggerBuilder
					.newTrigger()
					.withIdentity(name, namespace)
					.withSchedule(
							CronScheduleBuilder.cronSchedule(scheduleJob.getTaskCron()
									.getCron()));
			trigger = triggerBuilder.build();
		} else {
			TriggerBuilderSimpleTrigger triggerBuilder = TriggerBuilder
					.newTrigger().withIdentity(name, namespace)
					.withSchedule(SimpleScheduleBuilder.simpleSchedule());
			trigger = triggerBuilder.build();
		}
		 加入调度器
		quartzScheduler.scheduleJob(jobDetail, trigger);
		 启动
		if (quartzScheduler.isShutdown()) {
			quartzScheduler.start();
		}
	}

	 取消某个任务，需要名字和组
	public boolean delete(TaskCron project) throws SchedulerException {
		TriggerKey triggerKey = new TriggerKey(project.getName(),
				project.getNamespace());
		quartzScheduler.unscheduleJob(triggerKey);
		return true;
	}

	public void refresh(TaskCron syncProject) throws SchedulerException {
		delete(syncProject);
		runProject(syncProject);
	}
}
```

