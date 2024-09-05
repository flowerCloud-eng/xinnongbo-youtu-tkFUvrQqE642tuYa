
# 原来有这么多时间


六月的那么一天，天气比以往时候都更凉爽，媳妇边收拾桌子，边漫不经心的对我说：你最近好像都没怎么阅读了。 正刷着新闻我，如同被一记响亮的晴空霹雳击中一般，不知所措。是了，最近几月诸事凑一起，加之两大项目接踵而至，确实有些许糟心，于是总是在空闲的时间泡在新闻里聊以解忧，再回首，隐隐有些恍如隔世之感。于是收拾好心情，翻开了躺在书架良久的整洁三步曲。也许是太久没有阅读了， 一口气，Bob大叔 Clean 系列三本都读完了，重点推荐Clear Architecture，部分章节建议重复读，比如第5部分\-软件架构，可以让你有真正的提升，对代码，对编程，对软件都会有不一样的认识。


Clean Code 次之，基本写了一些常见的规约，大部分也是大家熟知，数据结构与面向对象的看法，是少有的让我 哇喔的点，如果真是在码路上摸跋滚打过的，快速翻阅即可。The Clean Coder 对个人而言可能作用最小。 确实写人最难，无法聚焦。讲了很多，但是感觉都不深入，或者作者是在写自己，很难映射到自己身上。 当然，第二章说不，与第14章辅导，学徒与技艺，还是值得一看的。


阅读技术书之余，又战战兢兢的翻开了敬畏已久的朱生豪先生翻译的《莎士比亚》, 不看则已，因为看了根本停不来。其华丽的辞职，幽默的比喻，真的会让人情不自禁的开怀朗读起来。


。。。


再看从6月到现在，电子书阅读时间超过120小时，平均每天原来有1个多小时的空余时间，简直超乎想像。



![](https://img2024.cnblogs.com/blog/88102/202409/88102-20240904231210404-386101449.png)![](https://img2024.cnblogs.com/blog/88102/202409/88102-20240904231233769-1688000216.png) ![](https://img2024.cnblogs.com/blog/88102/202409/88102-20240904162547093-192577063.png)


 看了整洁架构一书，就想写代码，于是有了这篇文章。


 



# 灵魂拷问 \- 宕机怎么办


为了解决系统中大量规则配置的问题，与同事一起构建了一个可视化表达式引擎 RuleLink[《非全自研可视化表达引擎\-RuleLinK》](https://github.com)，解决了公司内部几乎所有配置问题。尤为重要的一点，所有配置业务同学即可自助完成。随着业务深入又增加了一些自定义函数，增加了公式及计算功能，增加组件无缝嵌入其他业务...我一度以为现在的功能已经可以满足绝大部分场景了。真到Wsin强同学说了一句：业财项目是**深度依赖**RuleLink的，流水打标，关联科目。。。我知道他看了数据，10分RuleLink执行了5万\+次。这也就意味着，如果RuleLink宕机了，业财服务也就宕机了，也就意味着巨大的事故。这却是是一个问题，公司业务确实属于非常低频，架不住财务数据这么多。如果才能让RuleLink更稳定成了当前的首要问题。


 ![](https://img2024.cnblogs.com/blog/88102/202409/88102-20240905010646165-669219837.png)![](https://img2024.cnblogs.com/blog/88102/202409/88102-20240905092247094-1366428126.png)


 


# 高可用VS少依赖


要提升服务的可用性，增加服务的实例是最快的方式。 但是考虑到我们自己的业务属性，以及业财只是在每天固定的几个时间点短时高频调用。 增加节点似乎不是最经济的方式。看 Bob大叔的《Clear Architecture》书中，对架构的稳定性有这样一个公式：不稳定性，I\=Fan\-out/(Fan\-in\+Fan\-out)


Fan\-in：入向依赖，这个指标指代了组件外部类依赖于组件内部类的数量。


Fan\-out：出向依赖，这个指标指代了组件内部类依赖于组件外部类的数量。


这个想法，对于各个微服务的稳定性同时适用，少一个外部依赖，稳定性就增加一些。站在业财系统来说，如果我能减少调用次数，其稳定性就在提升，批量接口可以一定程度上减少依赖，但并未解决根本问题。那么调用次数减少到极限会是什么样的呢？答案是：**一次。**如果规则不变的话，我只需要启动时加载远程规则，并在本地容器执行规则的解析。如果有变动，我们只需要监听变化即可。这样极大减少了业财对RuleLink的依赖，也不用增RuleLink的节点。实际上大部分配置中心都是这样的设计的，比如apollo，nacos。 当然，本文的实现方式也有非常多借鉴（copy）了apollo的思想与实现。


# 服务端设计


模型比较比较简单，应用订阅场景，场景及其规则变化时，或者订阅关系变化时，生成应用与场景变更记录。类似于生成者\-消费都模型，使用DB做存储。


![](https://img2024.cnblogs.com/blog/88102/202409/88102-20240905110207718-124680682.png)


 


![](https://img2024.cnblogs.com/blog/88102/202409/88102-20240905111536341-1083638786.png)


 


# ”推送”原理


整体逻辑参考apollo实现方式。 服务端启动后 创建Bean ReleaseMessageScanner 注入变更监听器 NotificationController。ReleaseMessageScanner 一个线程定时扫码变更，如果有变化 通知到所有监听器。


NotificationController在得知有配置发布后是如何通知到客户端的呢？实现方式如下：1，客户端会发起一个Http请求到RuleLink的接口，NotificationController2，NotificationController不会立即返回结果，而是通过Spring DeferredResult把请求挂起3，如果在60秒内没有该客户端关心的配置发布，那么会返回Http状态码304给客户端4，如果有该客户端关心的配置发布，NotificationController会调用DeferredResult的setResult方法，传入有变化的场景列表，同时该请求会立即返回。客户端从返回的结果中获取到有变化的场景后，会直接更新缓存中场景，并更新刷新时间


ReleaseMessageScanner 比较简单，如下。NotificationController 代码也简单，就是收到更新消息，setResult返回（如果有请求正在等待的话）





```
public class ReleaseMessageScanner implements InitializingBean {
  private static final Logger logger = LoggerFactory.getLogger(ReleaseMessageScanner.class);

  private final AppSceneChangeLogRepository changeLogRepository;
  private int databaseScanInterval;
  private final List listeners;
  private final ScheduledExecutorService executorService;

  public ReleaseMessageScanner(final AppSceneChangeLogRepository changeLogRepository) {
    this.changeLogRepository = changeLogRepository;
    databaseScanInterval = 5000;
    listeners = Lists.newCopyOnWriteArrayList();
    executorService = Executors.newScheduledThreadPool(1, RuleThreadFactory
        .create("ReleaseMessageScanner", true));
  }

  @Override
  public void afterPropertiesSet() throws Exception {
    executorService.scheduleWithFixedDelay(() -> {
      try {
        scanMessages();
      } catch (Throwable ex) {
        logger.error("Scan and send message failed", ex);
      } finally {

      }
    }, databaseScanInterval, databaseScanInterval, TimeUnit.MILLISECONDS);

  }

  /**
   * add message listeners for release message
   * @param listener
   */
  public void addMessageListener(ReleaseMessageListener listener) {
    if (!listeners.contains(listener)) {
      listeners.add(listener);
    }
  }

  /**
   * Scan messages, continue scanning until there is no more messages
   */
  private void scanMessages() {
    boolean hasMoreMessages = true;
    while (hasMoreMessages && !Thread.currentThread().isInterrupted()) {
      hasMoreMessages = scanAndSendMessages();
    }
  }

  /**
   * scan messages and send
   *
   * @return whether there are more messages
   */
  private boolean scanAndSendMessages() {
    //current batch is 500
    List releaseMessages =
        changeLogRepository.findUnSyncAppList();
    if (CollectionUtils.isEmpty(releaseMessages)) {
      return false;
    }
    fireMessageScanned(releaseMessages);

    return false;
  }


  /**
   * Notify listeners with messages loaded
   * @param messages
   */
  private void fireMessageScanned(Iterable messages) {
    for (AppSceneChangeLogEntity message : messages) {
      for (ReleaseMessageListener listener : listeners) {
        try {
          listener.handleMessage(message.getAppId(), "");
        } catch (Throwable ex) {
          logger.error("Failed to invoke message listener {}", listener.getClass(), ex);
        }
      }
    }
  }
}
```


 



 


# 客户端设计


![](https://img2024.cnblogs.com/blog/88102/202409/88102-20240905112325762-69439313.png)


上图简要描述了客户端的实现原理：


* 客户端和服务端保持了一个长连接，从而能第一时间获得配置更新的推送。（通过Http Long Polling实现）
* 客户端还会定时从RuleLink配置中心服务端拉取应用的最新配置。
	+ 这是一个fallback机制，为了防止推送机制失效导致配置不更新
	+ 客户端定时拉取会上报本地版本，所以一般情况下，对于定时拉取的操作，服务端都会返回304 \- Not Modified
	+ 定时频率默认为每5分钟拉取一次，客户端也可以通过在运行时指定配置项: rule.refreshInterval来覆盖，单位为分钟。
* 客户端从RuleLink配置中心服务端获取到应用的最新配置后，会写入内存保存到SceneHolder中，
* 可以通过RuleLinkMonitor 查看client 配置刷新时间，以及内存中的规则是否远端相同


 


# 客户端工程


客户端以starter的形式，通过注解EnableRuleLinkClient 开始初始化。




```
 1 /**
 2  * @author JJ
 3  */
 4 @Retention(RetentionPolicy.RUNTIME)
 5 @Target(ElementType.TYPE)
 6 @Documented
 7 @Import({EnableRuleLinkClientImportSelector.class})
 8 public @interface EnableRuleLinkClient {
 9 
10   /**
11    * The order of the client config, default is {@link Ordered#LOWEST_PRECEDENCE}, which is Integer.MAX_VALUE.
12    * @return
13    */
14   int order() default Ordered.LOWEST_PRECEDENCE;
15 }
```


 


 


![](https://img2024.cnblogs.com/blog/88102/202409/88102-20240905114217858-828358250.png)


 


# 在最需求的地方应用起来


花了大概3个周的业余时间，搭建了client工程，经过一番斗争后，决定直接用到了最迫切的项目 \- 业财。当然，也做了完全准备，可以随时切换到RPC版本。 得益于DeferredResult的应用，变更总会在60s内同步，也有兜底方案：每300s主动查询变更，即便是启动后RuleLink宕机了，也不影响其运行。这样的准备之下，上线后几乎没有任何波澜。当然，也就没有人会担心宕机了。这真可以算得上一次愉快的编程之旅。


 


成为一名优秀的程序员！


 


 本博客参考[veee加速器](https://liuyunzhuge.com)。转载请注明出处！
