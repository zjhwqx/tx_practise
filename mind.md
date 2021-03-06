####思路
A B C D  每个区域座位数为1950 总数就是4*1950 = 7800  
    排号规则：
        A区左下角从1 开始， 到A区右上角座位号为1950  
        B区左下角从1951开始，到B区右上角为3900  
        C区左下角从3901开始，到B区右上角为5850  
        D区左下角从5851开始，到B区右上角为7800  
        每排的座位数依次从左到右开始递增，如：1,2,3,4....50  
暂定用xml存储座位信息  
随机数选取的时候,获取未选中的数据，从未选中数据中随机选取。    
体育馆座位优劣排序： 优等 B C区 前10排 ； ;良好： A D 区前10排，B C 区11排-26排 ；劣：A D 区 11-26排

1. 落点尽量靠近已选中大多座位区
2. 多张票需要考虑能否连座或者前后排靠近
3. 票的落点 最佳位置选择 （B C 两区，前十排位置）
4. 考虑区域座位选择人数平衡情况

不管买几张票，购票前都需要判断当前剩余票数是否满足购买需求。  
1张票逻辑： 落点优先选择优区域座位（不考虑让人群扎堆情况），优区域座位数目充足，计算落点，落点需要满足平衡BC两区域人数平衡  
多张票逻辑： 落点优先选择优区域座位（不考虑人群扎堆情况），判断优选区域是否有连坐座位，座位的前后左右，是否有空缺。优先选择左右空余座位。  
           在优区域座位不足情况下（不考虑优区域座位不足，但是可以安排良区域相邻座位情况），在良区域进行查询判断。逻辑与优区域查询判断一致。  
           良区域座位不足情况下，考虑列区域。
多张票的时候，选择最简单的选票方式。  
两张票：随机一张固定，另一张在其四周任意都可。优先选择左右的位置。  
三张票：随机一张固定，左右相邻是否有空座，有就选择，没有就看上下相邻,保证上下左右有两个座位可选  
四张票：随机一个固定，看上下左右相邻是否有三个座位选择。  
五张票：随机一个固定，看上下左右相邻四个座位是否都是空的。
  
            
架构方面：
主要分为这几层：  
请求分发：DNS+nginx  
服务：多台服务器集群 （异步处理请求）  
服务器监控：Open-Falcon
日志收集分析：elk
数据cache：Redis Cluster  将全部座位数据提前写入 还有7800个座位资格 具体见如下图：    
[架构图](https://www.processon.com/view/link/5a684894e4b010a6e72eadf2)
数据持久化层：最后还是要将数据保存在硬盘上。

经过面试官再三提醒，才注意到侧重点在于架构知识。    
其实体育馆选票这个，可以理解为变相的抢票秒杀。    
一共7800张票，也就是说存放入数据队列中的数据为7800条。可以按照1-7800进行排号。用户发送请求，进行抢购。实际上是抢到座位资格，具体座位还需要
随机算法算出分配。   
请求过来之后，DNS轮询，将请求配到对应的服务上去。部署相对简单，成本低。  
也会有问题，不知道服务器当前状态是否是健康的，有可能一个服务挂了，请求就永远不会有所回应。不过这个问题可以通过监控服务器体系来解决。    
该服务节点上nginx在通过轮询（默认），指定权重，或者url_hash等负载均衡策略，将请求分配给nginx下所属服务器。    
利用nginx+fastcgi的方式可以进行异步处理，或者直接采用swoole。通过将同步处理转换成异步处理，增加服务器处理能力。   
这里将抢资格以及生成随机座位逻辑分开。用户抢到资格后就开始异步处理随机座位的生成。  
处理生成后，将用户id以及对应随机座位信息key类型进行储存。  
目前就只是简单的实现了座位的随机，没有结合具体的随机策略生成随机座位。     
当然异步处理需要配合消息队列来使用，这里使用redis，这里数据的读写都通过redis在内存中进行。  
不使用mysql，是因为数据在进行读写的时候有事务以及锁，性能不佳，使用mysql应对高并发请求就可能需要数据缓存，然后又涉及到缓存数据的同步问题。  
而且mysql数据的储存是对硬盘的IO，跟直接内存IO相比效率低。  
综合考虑，这边就使用redis来作为数据存储工具。  
最后是数据层，这里用了redis集群Redis Cluster，其采用无中心结构，每个节点保存数据和整个集群的状态，每个节点都跟其他节点相连接。
客户端可以像使用单redis一样使用。通过这个集群来保证数据的安全性，某个redis挂了，数据也不会丢失。  
最后就需要将数据持久化，从redis集群中将抢票结束后的数据存储在硬盘上。  
当然整个架构过程中还需要两个东西。elk日志收集分析，对线上业务日志能够进行实时监控、有异常及时定位排除故障等等优势。  
利用Open-Falcon对服务器资源，关键业务进程资源消耗，进行全面监控，出现问题及时报警。














