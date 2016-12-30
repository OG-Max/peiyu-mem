<h1>简介</h1>
主要实现优惠券促销活动，首先创建活动，然后创建券组，采用预处理的方式提前进行制券，在第一版本主要实现，功能的基本业务。然后在分支实现，大数量和高并发问题。
<h2>分支1.1</h2>
1：解决优惠券编码重复问题，原先采用的是获取数据库所有的券，然后去比对是否重复，如果库数据量达百万的时候就会出现非常缓慢，而且会 出现经常制券失败等，所以此版本舍弃原先采用随机数的模式，通过推特的雪花算法来避免唯一，但是依然保留优惠券前缀和后缀。
2：由原来的异步采用线程修改为线程池，以为高并发时候存在大量的线程占内存空间。
3：由原来制券采用for循环模式修改为批量制券，而且采用分配插入优惠券，一批次目前定为5000.
4：加入消息队列（采用rabbitMQ）对于某一批次添加失败，把失败的放入对列中，通过队列进行补救，已到达高可用。避免大批量优惠券来回重新导入 消息队列对于异常信息拒绝解决并重返消息队列中，配置2个消费者以避免其中一个服务异常，消息处理出现死循环
<h2>分支1.2</h2>
1：加入操作日志<br/>
目的：跟踪热点数据，查询日志快速跟踪应用程序中的慢查询或慢操作，为后面的优化奠定基础<br/>
2：加入异常日志<br/>
目的：快速的获取线程的异常问题，通过日志中的数据能快速修改<br/>
3：采用技术
通过aop和rabbitmq中间件来做，这样减少由于日志问题给程序带来的效率问题
<h2>未做优化效率统计</h2>
采用数据库mysql<br/>
数据：添加25个有效活动，每个活动下分别有2个券组，每个券组下制券是5万张。优惠券表中250万条记录<br/>
业务：一个会员消费同时满足这25个活动要送50张优惠券。<br/>
统计：整个发券过程经过10次统计得出大约消耗是306s，其中每次获取优惠券耗时6s。如果多次循环必然带来性能的瓶颈<br/>
更新优惠券状态大约耗时是0.5s，从上我们可以看出我们的性能问题主要出在获取优惠券上。所以才1.3版本主要通过程序来解决这个问题
<h2>分支1.3</h2>
目的：通过程序代码和优化数据库来提高性能
具体方案：
1：以前获取券组下所有的优惠券修改为获取100条(经测试统计得出发送50张券消耗时间是106s，每次获取优惠券大约耗时是2s多，整体性能提升近3倍)<br/>
2：优化sql，加入组合索引（统计得出发送50张优惠券消耗总时间是2.5s，每次获取优惠券大约耗时是0.015s，整体的性能提升了近42倍）<br/>
3：加入本地缓存（如果一次性获取的优惠券先放入map中,那么下次如果还有就不需要从库中获取优惠券。统计发现：10件商品，每件商品发50张优惠券<br/>
不加本地缓存效率耗时是7.5s，加入本地缓存后耗时约5.5s，整体性能提升了2s）<br/>
效果分析：<br/>
4：对于发券采用批量更新来替代for循环
