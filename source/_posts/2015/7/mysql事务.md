title:  "MySQL事务"
date: 2015-07-14
updated: 2015-12-26
categories:
- db
tags:
- db
- transaction
---
> InnoDB的事务详解

## 事务隔离级别

- 未提交读（Read uncommitted）
- 已提交读（Read committed）
- 可重复读（Repeatable read） InnoDB默认隔离级别
- 可串行化（Serializable）

*注意*
- 这个隔离级别是针对“读”操作的，也就是针对 select * from xxx where ??? 的
- “可重复读”指的是：在当前事务中，只能看到自己的操作，别人commit的东西，看不到。
- 由此导致的问题是：如果想看到别人的更改，需要 select * for update，但是在这个时候会加上悲观锁

一些自己的理解
- 对于select读操作，RR级别只能看到自己事务中的情况，对于别的事务，一无所知。
- 但是在提交的时候，RR级别会根据是否有select for update, update/delete/insert这些语句，对各个事务中的顺序进行调整，也就是说如果是行锁，那么就是在行锁上wait；如果是表锁就是在表上wait；如果是区域锁，就是在区域上wait。
- 编程的时候，需要针对以上两种情况，心中有数，哪些地方只需要“快照读”，哪些地方需要“select for update”。特别是如果在本事务中，需要知道其他事务当前的状态，来决定本事务回滚，那么就用select for update。在本事务中insert了，但是其他事务中需要查看状态，也需要select for update。总之，insert/update/delete本身，在RR级别下，只是保证了数据库的隔离，但是业务逻辑的回滚等操作，是需要在业务代码中进行select for update进行查询，然后决定是否回滚的。

## 订单的例子
涉及多张表时，更加要注意锁粒度和事务读等操作。注意看里面的step注释

```java
Object retCode = sbsTransactionTemplate.execute(new TransactionCallback() {
    @Override
    public Object doInTransaction(TransactionStatus status) {
        try {
            /**
            step1: 这一步就会锁住CallCenterLeadsStatus表中这一行，所有需要表中这一行的，都会wait
                   这个操作比code级别的锁要好。
            */
            int leadsId = (Integer)context.get("leadsId");
            CallCenterLeadsStatusDTO callCenterLeadsStatusDTO = callCenterLeadsStatusService.loadCallCenterLeadsStatusByIdForUpdate(leadsId);
            if (callCenterLeadsStatusDTO.getGrabShopCount() >= Integer.parseInt(LionConfigUtils.getProperty("wed-customer-service.home.callcenter.maxLeadsGrabCount", "3"))){
                throw new RuntimeException("GrabLeadsAction 单子已被抢完, leadsId:"+leadsId);
            }

            /**
            step2: 这一步就会锁住CallCenterShopGrabLeads表，因为Innodb根据索引如果找不到行锁，就会锁住整表
            */
            //根据leadsId查找到哪些商户抢了，如果这个商户已经抢了，抛exception
            final int shopId = (Integer)context.get("shopId");
            List<Integer> shopIdsThatGrabedThisLeads = callCenterShopGrabLeadsService.findShopIdsByLeadsIdForUpdate(leadsId);
            if (CollectionUtils.isNotEmpty(shopIdsThatGrabedThisLeads)
                    && shopIdsThatGrabedThisLeads.contains(shopId)){
                throw new RuntimeException("GrabLeadsAction 单子你已经抢过了, leadsId:"+leadsId+" shopId:"+shopId);
            }

            //更新Leads
            callCenterLeadsStatusDTO.setGrabLeadsTime(new Date());
            callCenterLeadsStatusDTO.setGrabShopCount(callCenterLeadsStatusDTO.getGrabShopCount()+1);
            updateLeadsStatus(callCenterLeadsStatusDTO, to);

            if (event == CallCenterLeadsFSMEvent.ShopGrabLeads) {
                //商户抢单
                //该商户是否满足
                /**
                step n: 这一步，如果抛exception，就会rollback，然后释放掉锁
                */
                HomeCallCenterShopConfigDTO homeCallCenterShopConfigDTO = homeCallCenterShopConfigService.loadCallCenterShopConfigByShopId(shopId);
                HomeCallCenterLeadsInfoDTO homeCallCenterLeadsInfoDTO = (HomeCallCenterLeadsInfoDTO) context.get("HomeCallCenterLeadsInfoDTO");
                if ( !homeCallCenterShopConfigService.checkIfShopConfigSatisfyGrabLeads(homeCallCenterShopConfigDTO, callCenterLeadsStatusDTO, homeCallCenterLeadsInfoDTO) ){
                    throw new RuntimeException("GrabLeadsAction 商户不满足Leads条件, shopId:"+shopId+", leadsId"+leadsId);
                }

                /**
                step n+1: 这一步加锁，也是需要的，因为可能有其他来源的抢单或者分单更新这张表，比如多个op在抢不同的单子。
                那这个时候上面加锁的leads无法限制了，但是对于商户抢单来讲还是需要有限制的。
                */
                int shopGrabedLeadsCount = callCenterShopGrabLeadsService.findShopGrabLeadsCountByShopIdForUpdate(shopId, null);
                int bonusChance = homeCallCenterShopConfigDTO.getBonusChance();
                if (shopGrabedLeadsCount<Integer.parseInt(LionConfigUtils.getProperty("wed-customer-service.home.callcenter.shopGrabLeadsChancesOneDay", "3"))){
                    //抢单次数还没到达阈值
                    //下面就可以更新了
                } else if (bonusChance>0){
                    //可以用bonusChance
                    updateShopConfig(shopId);
                } else {
                    throw new RuntimeException("GrabLeadsAction 商户抢单次数已超过阈值，并且其bonusChance也没了,shopId:"+shopId+", leadsId"+leadsId);
                }

                insertShopGrabLeads(leadsId, shopId, 1);

                int shopGrabedLeadsCountAgain = callCenterShopGrabLeadsService.findShopGrabLeadsCountByShopIdForUpdate(shopId, null);
                if (shopGrabedLeadsCountAgain>Integer.parseInt(LionConfigUtils.getProperty("wed-customer-service.home.callcenter.shopGrabLeadsChancesOneDay", "3"))&&bonusChance==0){
                    throw new RuntimeException("GrabLeadsAction 并发问题，商户当日抢单超过阈值, shopId:"+shopId+", leadsId"+leadsId);
                }

                return new Integer(1);
            } else {
                //运营分单
                insertShopGrabLeads(leadsId, shopId, 2);
//                        int shopGrabedLeadsCountAgain = callCenterShopGrabLeadsService.findShopGrabLeadsCountByShopId(shopId, null);
//                        if (shopGrabedLeadsCountAgain>Integer.parseInt(LionConfigUtils.getProperty("wed-customer-service.home.callcenter.shopGrabLeadsChancesOneDay", "3"))){
//                            throw new RuntimeException("GrabLeadsAction 并发问题，商户当日抢单超过阈值, shopId:"+shopId+", leadsId"+leadsId);
//                        }
                HomeCallCenterLeadsInfoDTO homeCallCenterLeadsInfoDTO = (HomeCallCenterLeadsInfoDTO) context.get("HomeCallCenterLeadsInfoDTO");
                sms2Shop(SmsTypeConstants.Home.CALLCENTER_NEW_LEADS_ADMIN_TO_MERCHANT, new ArrayList<Integer>(){{add(shopId);}}, callCenterLeadsStatusDTO, homeCallCenterLeadsInfoDTO);
                return new Integer(2);
            }

        }catch (Exception ex){
            status.setRollbackOnly();
            logger.error("GrabLeadsAction throw Exception!!", ex);
            return new Integer(-1);
        }
    }
});
```
