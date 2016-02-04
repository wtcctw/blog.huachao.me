title:  "Java DB 必知必会"
date: 2016-02-03
categories:
- db
tags:
- db
---

> RDBMS的内容真的很多，这里只是沧海一粟

## $0 适合的读者
瞄过`java.sql`包里的代码，用过`JdbcTemplate`，写过hibernate或者ibatis，标记过几个`@Transactional`，看到过c3p0，知道sql慢查询一说，但是，跟我一样，脑子里总觉得一团浆糊。
为什么会这样？原因就是你只是在写或者抄代码，没有思考，思考了更加没有记录下来，记录下来以后没有跟别人分享，然后忘了。所以，看下文。

## $1 知识点及关键点
![DB相关知识点](http://7xqqrz.com1.z0.glb.clouddn.com/db-without-sql_explain.png)
[看不清点这里](http://7xqqrz.com1.z0.glb.clouddn.com/db-without-sql_explain.png)

## $2 需要掌握和阅读的内容
- [jdbc specification](https://en.wikipedia.org/wiki/Java_Database_Connectivity)，这个要看一遍的，最重要是对着java.sql里面的interface看
- [spring-jdbc](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/jdbc.html)，过一遍spring-jdbc的文档能够极大帮助看懂spring-jdbc的代码，真的不难，而且结构写得非常清晰，大块大块的`Template Method`模式
- [数据库事务原理](http://tech.meituan.com/innodb-lock.html)，这篇文章对理解事务的原理有帮助
- [spring-tx](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/transaction.html)，也是过一遍文档，然后就很清楚如何配置声明式事务和编程式事务了
- [数据库连接池c3p0](http://www.mchange.com/projects/c3p0/)，瞄一眼大概配置有哪几块就行
- [mybatis-spring](http://www.mybatis.org/spring/zh/index.html)，不懂hibernate，所以就只看mybatis了
- sql相关的内容，比如索引，优化等。这个内容就多了。。。就推荐一本《高性能MySQL》

## $3 使用Explain查看sql执行计划的例子

### 业务描述
> 商户产品和产品标签，一个商户有多个产品，每个产品在WED_ShopProduct中有一条记录，每个产品可能还有一些标签，每个标签在WED_ShopProductTag表中有一条记录。

- WED_ShopProduct表，数据量 162,157

ID|ShopID|ShopName|CityID|ProductCategoryID|Price|OriginalPrice|IsMain|OrderNo|Name|SimpleDescription|PicID|PicPath|AddTime|UpdateTime|RequiredIntegrity|AllIntegrity|ProductType|Qualified|BookingCount|Special|ViewCount
--|------|--------|------|-----------------|-----|-------------|------|-------|----|-----------------|-----|-------|-------|----------|-----------------|------------|-----------|---------|------------|-------|---------

2个联合索引
IX_ShopID_OrderNo (ShopID,OrderNo),
IX_CityID_ProductCategoryID_ProductType (CityID,ProductCategoryID,ProductType)

- WED_ShopProductTag表，数据量 415,037

ID|ProductID|ShopID|UserID|TagNameID|TagValue|ProductCategoryID|CityID|AddTime|UpdateTime
--|---------|------|------|---------|--------|-----------------|------|-------|----------
4个索引
IX_ShopID (ShopID),
IX_ProductID (ProductID),
IX_PCateID` (ProductCategoryID),
IX_Tag (TagNameID,TagValue(255))

### WED_ShopProduct查询
- `select * from WED_ShopProduct where ShopID=4119052`

id|select_type|table|type|possible_keys|key|key_len|ref|rows|Extra
--|-----------|-----|----|-------------|---|-------|---|----|-----
1|SIMPLE|WED_ShopProduct|ref|IX_ShopID_OrderNo|IX_ShopID_OrderNo|4|const|150|NULL

- `select * from WED_ShopProduct where OrderNo=5`， 没有走到联合索引

id|select_type|table|type|possible_keys|key|key_len|ref|rows|Extra
--|-----------|-----|----|-------------|---|-------|---|----|-----
1|SIMPLE|WED_ShopProduct|ALL|NULL|NULL|NULL|NULL|155390|Using where

- `select * from WED_ShopProduct where shopid=4119052 order by orderNo`

id|select_type|table|type|possible_keys|key|key_len|ref|rows|Extra
--|-----------|-----|----|-------------|---|-------|---|----|-----
1|SIMPLE|WED_ShopProduct|ref|IX_ShopID_OrderNo|IX_ShopID_OrderNo|4|const|149|Using where

- `select * from WED_ShopProduct where shopid=4119052 order by ismain`，注意Using filesort

id|select_type|table|type|possible_keys|key|key_len|ref|rows|Extra
--|-----------|-----|----|-------------|---|-------|---|----|-----
1|SIMPLE|WED_ShopProduct|ref|IX_ShopID_OrderNo|IX_ShopID_OrderNo|4|const|149|Using where; Using filesort

### 两表JOIN
- 查出某个商户的所有产品及其产品标签 `select product.shopId, product.ID, product.name, tag.ProductID, tag.TagNameID, tag.TagValue  from WED_ShopProduct product JOIN WED_ShopProductTag tag on product.ID = tag.ProductID where product.ShopID=4119052`

id|select_type|table|type|possible_keys|key|key_len|ref|rows|Extra
--|-----------|-----|----|-------------|---|-------|---|----|-----
1|SIMPLE|product|ref|PRIMARY,IX_ShopID_OrderNo|IX_ShopID_OrderNo|4|const|149|NULL
1|SIMPLE|tag|ref|IX_ProductID|IX_ProductID|4|DianPingWed.product.ID|4|NULL

- 找出某个商户的产品中有标签为“现代简约”的 `select *  from WED_ShopProduct product JOIN WED_ShopProductTag tag on product.ID = tag.ProductID where product.ShopID=4119052 and tag.TagValue='现代简约'`, 走ShopId索引，然后比较TagValue值，所以有个Using Where

id|select_type|table|type|possible_keys|key|key_len|ref|rows|Extra
--|-----------|-----|----|-------------|---|-------|---|----|-----
1|SIMPLE|product|ref|PRIMARY,IX_ShopID_OrderNo|IX_ShopID_OrderNo|4|const|149|NULL
1|SIMPLE|tag|ref|IX_ProductID|IX_ProductID|4|DianPingWed.product.ID|4|Using where

- 找出所有商户的产品中有标签为“现代简约”的 `select *  from WED_ShopProduct product JOIN WED_ShopProductTag tag on product.ID = tag.ProductID where tag.TagValue='现代简约'`，注意先走了tag表，因为有where筛选，然后到product表查询

id|select_type|table|type|possible_keys|key|key_len|ref|rows|Extra
--|-----------|-----|----|-------------|---|-------|---|----|-----
1|SIMPLE|tag|ALL|IX_ProductID|NULL|NULL|NULL|401835|Using where
1|SIMPLE|product|eq_ref|PRIMARY|PRIMARY|4|DianPingWed.tag.ProductID|1|NULL	


