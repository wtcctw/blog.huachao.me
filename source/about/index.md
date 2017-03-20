title: About Me
date: 2015-12-26 00:15:22
---
# 个人信息
- #### 姓名：华超
- #### 硕士/中科院软件研究所
- #### 工作年限：7年
- #### Email：[i@huachao.me](mailto:i@huachao.me)

# 工作经历
- ### 2016.6 ~ 至今 [阿里巴巴·互联网汽车]()
	- 互联网汽车, **技术专家**
- ### 2014.5 ~ 至今 [大众点评](http://www.dianping.com)
	- 家装事业部, **Engineering Lead**
	- 结婚事业部, **Senior Software Engineer**
- ### 2010.7 ~ 2014.4 [IBM](http://www.ibm.com.cn)
	- IBM Cloud Management with Openstack, **Senior Software Engineer**
	- China System Technology Lab, **Software Engineer**

# 项目经验
- ### 阿里巴巴互联网汽车大平台
	- 中间件整合（网关、微服务、消息中心、配置中心等）
	- 业务能力提升（内部工具开发、性能调优）
	- 数据平台（大数据搜集和分析）

- ### 通用Java后端开发
	- 做过`上百个`各种类型的需求，涉及到：大众点评结婚家装频道的[前台](http://www.dianping.com/shanghai/home)、商户后台、运营后台、H5游戏活动页面、大众点评手机App的后端API
	- 手机App的后端API `PV`约为100w/天；前台页面 `PV`约为50w/天；秒杀类H5游戏活动 `QPS`大于1k/s（说明：`QPS`主要依靠memcached和redis得到提升，一般memcached和redis的`QPS`大于10k/s，MySql的`QPS`也可以达到几千k的数量级，这里的`QPS`是指应用的`QPS`，包含了复杂的业务逻辑处理）。
	- 设计业务数据存储模型；抽象通用的业务流程；架构可扩展的业务系统
	- 熟悉Java Web开发的`每一个`环节。典型的web应用开发框架![通用架构图](/images/about/java通用后端架构图.png)

- ### 基于状态机的订单系统
	- 订单涉及到很多页面：1）用户通过门户网站或者手机App端预约下单；2）运营同学管理订单；3）商户可以抢单和管理自己的订单；
	- 通过状态机抽象能够：1）封装订单状态的迁移；2）统一所有对订单的操作；3）方便扩展
	- 最终暴露给外部的API就只有2个：1）`State s = fsm.createState(/*数据*/)`；2）`State s = fsm.fire(State, Action)`；其中`fsm`是Finite State Machine的缩写。
	- 下图中彩色部分是需求迭代增加的，如果没有状态机，`if else`满天飞的感觉你们脑补下。![家装客源系统状态机](/images/about/callcenter-fsm.png)

- ### 基于Quartz的调度系统
	- [Quartz](https://quartz-scheduler.org/)是一个优秀的第三方调度组件。
	- 我们做了一个统一的调度中心，让开发人员只需要简单的做2件事情就好了：1）写一个Java Class，包含业务逻辑；2）在调度中心的页面上新建一个job，说明你需要执行的地点（执行机器ip），执行的任务（Class），执行的时间参数（contab）。然后就可以在调度中心页面上进行监控了。
	- ![定时调度中心](/images/about/job-scheduler.png)

- ### PowerVM的Openstack Driver
	- 向[Openstack](https://www.openstack.org/)社区贡献了第一版[PowerVM](http://www-03.ibm.com/systems/power/software/virtualization/)的代码，使得Openstack可以管理PowerVM。
	- [nova-powervm](https://github.com/openstack/nova-powervm)是目前Openstack管理PowerVM的主要手段

- ### Watchdog
	- 监控IBM System x/p系列主机，在发生问题的时候主动重启服务和发送报警邮件
	- C++实现，基于`stdlib`跨Windows和Linux平台

# 文章和代码
- Linux相关文章
	- [Linux知识点小结(本文被 @开发者头条 收录，点赞超100)](https://blog.huachao.me/2016/1/Linux%E7%9F%A5%E8%AF%86%E7%82%B9%E5%B0%8F%E7%BB%93/)
- I/O相关文章
	- [TCP/IP,Java Socket简单分析](https://blog.huachao.me/2015/12/TCP:IP,Java%20Socket%E7%AE%80%E5%8D%95%E5%88%86%E6%9E%90/)
	- [epoll](https://blog.huachao.me/2015/12/epoll/)
- DB相关文章
	- [分布式RDBMS](https://blog.huachao.me/2016/2/%E5%88%86%E5%B8%83%E5%BC%8FRDBMS/)
	- [Java DB必知必会](https://blog.huachao.me/2016/2/Java%20DB%20%E5%BF%85%E7%9F%A5%E5%BF%85%E4%BC%9A/)
	- [MySQL事务](https://blog.huachao.me/2015/7/mysql%E4%BA%8B%E5%8A%A1/)
- 工具相关文章
	- [Spring小结](https://blog.huachao.me/2015/4/Spring%E5%9F%BA%E7%A1%80%E5%B0%8F%E7%BB%93/)
	- [git命令速记](https://blog.huachao.me/2015/4/git%E4%B8%80%E7%AB%99%E5%BC%8F/)
	- [Nginx和HTTPS验证](https://blog.huachao.me/2015/12/%E7%94%A8nginx%E5%81%9A%E5%8F%8D%E5%90%91%E4%BB%A3%E7%90%86%E5%92%8Chttps%E9%AA%8C%E8%AF%81/)
- 代码
	- [基于netty的一个简单RPC实现](https://github.com/wtcctw/sample-rpc)
	- [我对Jedis做的二次封装，主要是Redis Key的管理和JedisPool的可配置化](https://github.com/wtcctw/wed-redis-parent)

# 专业技能
- Java *精通*
 - Java多线程
 - Java I/O
 - JVM(熟练使用`jps jstack jstat jmap`等工具，分析OOM和Full GC)
- Linux *非常熟悉*
 - TCP/IP协议栈
 - 文件系统、磁盘管理(LVM)
 - 用户管理
 - 进程管理
 - Service管理
- DB(MySQL) *非常熟悉*
 - 熟练使用`ibatis`与`Spring事务`
 - 熟练使用MySQL索引来加速读和解决一些慢查询的问题
 - 熟悉MySQL的读写机制
 - 熟悉MySQL的锁机制
- 第三方框架 *熟练使用*
 - `Spring`
 - `Redis` 
 - `netty`
 - `Pigeon` 大众点评内部的一个RPC框架，和淘宝的[dubbo](http://dubbo.io/)类似，是一个针对Java系统用hessian进行序列化和反序列化的。
 - `Swallow` 大众点评内部的一个消息中间件。Kafka也有一些使用经验。
- 工具 *熟练使用*
 - `Git`
 - `Nginx`
- 前端 *有一些经验*
 - `JavaScript` 能够熟练使用`jQuery`，用`node.js`写简单的js脚本 
 - `CSS` 页面不复杂的情况能够胜任H5页面的CSS开发