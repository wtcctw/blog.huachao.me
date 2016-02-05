title:  "分布式RDBMS(Cobar)"
date: 2016-02-05
categories:
- db
tags:
- db
---

> 分布式关系型数据库，不是分布式存储系统

# 引子
传统的关系型数据库在高并发，大数据的情况下性能会急剧降低，比如，一般MySQL的qps在10k以下，单表的数据量千万级已经封顶。有鉴于此，程序员，DBA们想到通过多台MySQL来分担大量的查询和分多台机器来存储，分布式RDBMS由此而生。
本文以阿里巴巴开源的[Cobar](https://github.com/alibaba/cobar)为蓝本，简单叙述其设计的思想，要解决的问题，如果可能的话会简单解释一下其代码结构。注意目前阿里云上分布式关系型数据库的解决方案是[DRDS](https://www.aliyun.com/product/drds/)，其使用的核心就是[Cobar](https://github.com/alibaba/cobar)和[TDDL](https://github.com/alibaba/tb_tddl)。TDDL复杂度高，当前公布的文档较少，只开源动态数据源，分表分库部分还未开源，还需要依赖diamond，所以不讨论了。

# Cobar能用来干啥
一图胜前言。对外表现为一个关系型数据库，对内将记录路由到各个工作DB，可以进行无限的水平扩展(scale out)。
![](/images/2016/2/分布式RDBMS/cobar-1.png)
Cobar也支持垂直拆分，即不同业务存储不同的database。

# Cobar作为一个proxy怎么路由到各个工作DB
Cobar作为一个proxy，实现了接收客户端的链接和jdbc的协议，然后根据sql语句，查询路由定义，然后路由到工作DB的。

- schema定义，**注意：** schema/table中的rule，即为这个表对应的拆分规则

``` xml
<?xmlversion="1.0"encoding="UTF-8"?>
<!DOCTYPEcobar:schemaSYSTEM"schema.dtd">
<cobar:schema xmlns:cobar="http://cobar.alibaba.com/">
<!--schema定义 --> 
<schema name="dbtest" dataNode="dnTest1">
	<table name="tb2" dataNode="dnTest2,dnTest3" rule="rule1"/>
</schema>

<!--数据节点定义,数据节点由数据源和其他一些参数组织而成。--> 
<dataNode name="dnTest1">
	<property name="dataSource"> 
		<dataSourceRef>dsTest[0]</dataSourceRef>
	</property>
</dataNode>

<dataNode name="dnTest2">
	<property name="dataSource">
		<dataSourceRef>dsTest[1]</dataSourceRef>
	</property>
</dataNode>

<dataNode name="dnTest3">
	<property name="dataSource">
		<dataSourceRef>dsTest[2]</dataSourceRef>
	</property>
</dataNode>

<!--数据源定义,数据源是一个具体的后端数据连接的表示。-->
<dataSource name="dsTest" type="mysql">
	<property name="location">
		<location>192.168.0.1:3306/dbtest1</location>
		<location>192.168.0.1:3306/dbtest2</location>
		<location>192.168.0.1:3306/dbtest3</location>
	</property>
	<property name="user">test</property>
	<property name="password"></property>
	<property name="sqlMode">STRICT_TRANS_TABLES</property>
</dataSource>
</cobar:schema>
```

- 拆分键定义

``` xml
<?xmlversion="1.0"encoding="UTF-8"?>
<!DOCTYPEcobar:ruleSYSTEM"rule.dtd">
<cobar:rulexmlns:cobar="http://cobar.alibaba.com/">
<!--路由规则定义,定义什么表,什么字段,采用什么路由算法。-->
	<tableRule name="rule1">
		<rule>
			<columns>id</columns>
			<algorithm><![CDATA[func1(${id})]]></algorithm>
		</rule>
	</tableRule>
	<!--路由函数定义,应用在路由规则的算法定义中,路由函数可以自定义扩展。-->
	<function name="func1" class="com.alibaba.cobar.route.function.PartitionByLong">
		<property name="partitionCount">2</property>
		<property name="partitionLength">512</property>
	</function>
</cobar:rule>
```

# Cobar的工作流程
同样的一图胜千言。
![](/images/2016/2/分布式RDBMS/cobar-2.png)

# 最佳实践
## 数据拆分策略
- 容量和访问均衡。拆分键能够使得记录均匀地分布到各个工作DB上。
- 事务边界。尽量地在一个工作DB上进行事务；如果涉及到多台工作DB，那么使用消息队列来达到最终一致性；分布式数据库事务会有严重的性能问题，只在必须的时候使用

## 数据库连接池
推荐使用阿里巴巴的[druid](https://github.com/alibaba/druid/?spm=5176.docdrds/best-practice/best_connection_pool.2.1.F6nELr)作为数据库连接池。注意其中的url，要配置为proxy的ip，也就是Cobar机器所在的ip。
``` xml
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource" init-method="init" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver" />
    <!-- 基本属性 url、user、password -->
    <property name="url" value="jdbc:mysql://ip:port/db?autoReconnect=true&amp;rewriteBatchedStatements=true&amp;socketTimeout=30000&amp;connectTimeout=3000" />
    <property name="username" value="root" />
    <property name="password" value="123456" />
    <!-- 配置初始化大小、最小、最大 -->
    <property name="maxActive" value="20" />
    <property name="initialSize" value="3" />
    <property name="minIdle" value="3" />
    <!-- maxWait获取连接等待超时的时间 -->
    <property name="maxWait" value="60000" />

    <!-- timeBetweenEvictionRunsMillis间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
    <property name="timeBetweenEvictionRunsMillis" value="60000" />
    <!-- minEvictableIdleTimeMillis一个连接在池中最小空闲的时间，单位是毫秒-->
    <property name="minEvictableIdleTimeMillis" value="300000" />
    <property name="validationQuery" value="SELECT 'z'" />
    <property name="testWhileIdle" value="true" />
    <property name="testOnBorrow" value="false" />
    <property name="testOnReturn" value="false" />
</bean>
```
## SQL优化
传统的关系型数据库SQL优化在于减少磁盘IO，分布式关系型数据库的SQL优化在于减少网络IO。SELECT语句带上拆分键，能让proxy直接找到对应的实际工作库。

## Reference
- [Cobar文档](/refs/2016/2/分布式RDBMS/Cobar - Alibaba Open Sesame.pdf)
- [Cobar Solution](/refs/2016/2/分布式RDBMS/cobarSolution.ppt) 如果你全看懂了，麻烦教教我 :-D