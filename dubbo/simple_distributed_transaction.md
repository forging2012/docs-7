解决分布式事务最简单的方案就是向前或向后，即补充或回滚。

业务逻辑有如下三步：

1. 调用A服务冻结金额

2. 调用B服务增加额度

3. 记录数据库

进行到第2步失败 -> 写task，回滚第1步(通知A服务方去回滚)

进行到第3步失败 -> 写task，从A/B服务处同步数据至数据库



还有一种方案就是写task，转化为本地事务，由task去执行上述三步，失败则重试task，当然，A/B服务都有幂等性处理。这种方案的缺点之一，是假定了参数合理的情况下，服务最终一定会执行成功，缺点之二是看起来不自然。



复杂的方案就是TCC方案，参见蚂蚁金服的DTS
