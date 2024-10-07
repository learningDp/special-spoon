# mysql

## innodb 引擎

----

### checkPoint 技术

1. 缓冲区:  由于 cpu速度很快.而磁盘速度相对很慢, 于是, 在内存中划出一块区域,用于弥补磁盘速度较慢的对数据库性能的影响.
2. checkPoint 所解决的问题 :    

   1. 缩短数据库的拒付时间:

   1. 缓冲池不够用时,将脏页(最新页)刷新到磁盘.

   1. 重做日志不可用时,刷新脏页,
3. **所以,当数据库宕机的时候 :数据库不需要重做所有的日志,只需对checkPoint 后的重做日志进行恢复.(缩短数据库宕机恢复的时间)**

### innodb 关键特性

1. 插入缓冲(Insert Buffer)(提升性能):  

   1. insert buffer 

      对于非聚簇索引的插入h或更新操作时,

      1. 先判断非聚簇索引页,是否存在缓冲池中,存在则插入.

      2. 不存在则先放入Insert Buffer 对象中,但未放入数据库中.

      3. 然后再以一定频率和情况的进行Insert Buffer 和辅助索引页子节点的merge 操作.

         ---

      4. insert buffer 的数据结构仍是B+树

      5. 当辅助索引插入到页中时,根据一定规则构造一个search key(space , maker , offest ) ,接下来查询插入到insert buffer 这颗B+树中.

      6. space 表空间id , 每个表的唯一id, marker 兼容老版本的 insert buffer , offest 表示页所在的偏移量

   2. change buffer

      1. 1.0.x引入 change buffer ,可视为insert buffer 的升级
      2. 适用对象仍然为非唯一的辅助索引
      3. 相比于insert buffer ,change buffer 可以对INSERT , DELETE , UPDATE 都进行缓冲

   3. merge Insert Buffer 

      1. 在合并操作中,insert buffer 随机读取其B+树上的一个页,读取该页中的space及之后所需要数量的页.
      2. 合并时,若需要merge的表已经被删除,则直接丢弃已经被Insert/Change buffer的记录.

2. 两次写(Double Write )(提高数据可靠性):

   1. 一部分是内存中的doublewrite buffer ,另一部分是物理磁盘上共享表空间中连续的128个页. 大小均为 2MB
   2. 对缓冲池中的脏页刷新时,先复制一份到 doublewrite buffer
   3. 然后分两次,以每次1MB顺序写入共享表空间的物理磁盘上,然后调用fsync 函数, 同步磁盘,避免缓冲写带来的问题

3. 自适应哈希索引(Adaptive Hash Index)

   1. 监控对表上各索引页的查询,若观察到建立哈希索引可以提高素的,则建立哈希索引,称为自适应哈希索引(AHI)
   2. 要求:
      1. 对这个页的访问模式必须是一样的(即查询条件一样)
      2. 对联合索引(a,b),其访问模式可以是 WHERE a = xxx和 WHERE a = xxx  AND b = xxx 

4. 异步IO(Async IO):优势

   1. 当全部请求发送完成后,等待所有IO操作的完成
   2. 进行IO merge 操作,可以 将 多个IO合并为1个IO

5. 刷新邻接表(Flush Neighbor Pages)

   1. 控制参数 innodb_flush_neighbors
   2. 当刷新一个脏页时,Innodb会检测该页所在区的所有页,如果是脏页,那么进行一起刷新.对于机械硬盘有显著优势
   3. 对于固态硬盘,有着超高的IOPS,则建议关闭,即将innodb_flush_neighbors设置为0

