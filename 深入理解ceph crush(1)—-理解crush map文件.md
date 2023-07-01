### 一. 前言

------

**CRUSH** 是ceph 最核心的几个设计之一，为了深入理解crush，写了crush系列博客，一是为了记录，以免忘记，二是为想更深入学习crush的同行提供一些参考。

**本系列文章基于 Luminous 12.2.8**

博文的顺序如下

![image](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/crushblog%20%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0.png)

### 二. 理解crush map文件

------

##### 这里从crush map 文件入手，讲解一下crush map 文件配置项的含义和作用

- #### 1. 首先从一个最简单的ceph集群架构开始，如下图

  ![image](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/crush-%E5%9B%BE%E7%A4%BA.png)

  一个集群中只有1台主机，这台主机有3个osd，集群为副本模式，副本数为3，故障域为osd

------

- #### 2. 生成的crush map文件内容如下

  ```
  # begin crush map
  tunable choose_local_tries 0
  tunable choose_local_fallback_tries 0
  tunable choose_total_tries 50
  tunable chooseleaf_descend_once 1
  tunable chooseleaf_vary_r 1
  tunable chooseleaf_stable 1
  tunable straw_calc_version 1
  tunable allowed_bucket_algs 54
      
  # devices
  device 0 osd.0 class ssd
  device 1 osd.1 class ssd
  device 2 osd.2 class ssd
      
  # types
  type 0 osd
  type 1 host
  type 2 chassis
  type 3 rack
  type 4 row
  type 5 pdu
  type 6 pod
  type 7 room
  type 8 datacenter
  type 9 region
  type 10 root
      
  # buckets
  host host1 {
      id -2        # do not change unnecessarily
      id -3 class ssd        # do not change unnecessarily
      # weight 3.000
      alg straw2
      hash 0    # rjenkins1
      item osd.0 weight 1.000
      item osd.1 weight 1.000
      item osd.2 weight 1.000
  }
  root default {
      id -1        # do not change unnecessarily
      id -4 class ssd        # do not change unnecessarily
      # weight 3.000
      alg straw2
      hash 0    # rjenkins1
      item host1 weight 3.000
  }
      
  # rules
  rule replicated_rule {
      id 0
      type replicated
      min_size 1
      max_size 10
      step take default
      step choose firstn 0 type osd
      step emit
  }
      
  # end crush map
  ```

  因为在实际环境中通常是 *“数据中心–>机架–>主机–>磁盘”* 这样的层级拓扑，所以crush map设计为一种**树状结构**

------

- #### 3. cursh map 的组成部分

  ##### crush map 由5部分组成：tunable 参数, device, type, bucket, rule

  ##### 3.1 tunable 参数

  - *choose_local_tries* : 已废弃，为做向后兼容应保持为 0
  - *choose_local_fallback_tries* : 已废弃，为做向后兼容应保持为 0
  - *choose_total_tries* : 可调，选择bucket的最大尝试次数，缺省值为50，已被验证是比较合理的值,如果50还无法选择出期待的bucket，可以适当调整这个参数
  - *chooseleaf_descend_once* : 已废弃，为做向后兼容应保持 1
  - *chooseleaf_vary_r* : 这个参数是为了fix一些bug而存在，应该保持为1
  - *chooseleaf_stable* : 这个参数的实现也是为了避免一些不必要的pg迁移，应该保持为1
  - *straw_calc_version* : straw 算法的版本，为了向后兼容这个值应该保持为 1
  - *allowed_bucket_algs* : 允许使用的bucket 选择算法

  这里再重点讲解一下 *allowed_bucket_algs* 参数 ，crush 的bucket选择算法总共有下面5种

  ```c++
  enum crush_algorithm {
      CRUSH_BUCKET_UNIFORM = 1    // uniform
      CRUSH_BUCKET_LIST = 2       // list
      CRUSH_BUCKET_TREE = 3       // tree
      CRUSH_BUCKET_STRAW = 4      // straw
      CRUSH_BUCKET_STRAW2 = 5     // straw2
  }
  ```

  *allowed_bucket_algs* 计算方法为：

  ```
  1 << CRUSH_BUCKET_* : 1 左移 CRUSH_BUCKET_* 位数代表这个算法可用， 
  再将所有得到的结果，做二进制数的或运算，得到allowed_bucket_algs 值
  ```

  上面crush文件中 *allowed_bucket_algs = 54* 的计算过程如下，这是默认的参数

  ```
  allowed_bucket_algs = (1 << CRUSH_BUCKET_UNIFORM) |             // 1 右移 1位 
                        (1 << CRUSH_BUCKET_LIST) |                // 1 右移 2位
                        (1 << CRUSH_BUCKET_STRAW) |               // 1 右移 4位
                        (1 << CRUSH_BUCKET_STRAW2))               // 1 右移 5位
      
  allowed_bucket_algs = 00000010 | 00000100 | 00010000 | 00100000 = 00110110 = 54
  ```

  > 从这里也可以看出，默认情况下是不会允许 *CRUSH_BUCKET_TREE* 算法，因为在L这个版本禁用了tree算法。

  ------

  ##### 3.2 device

  每一个最末端的的物理设备，也叫叶子节点就叫device，比如osd,从L版开始引入了crush device class概念，可以对磁盘进行智能识别为 **hdd ssd nvme**类型，这里先不深入class,先知道有这么一个东西

  ```
  # devices
  device 0 osd.0 class ssd
  device 1 osd.1 class ssd
  device 2 osd.2 class ssd
  ```

  ------

  ##### 3.3 type

  **type** 是可以自定义的, 是bucket的类型，一般设计为层级结构，编号必须为正整数。

  ```
  type 0 osd
  type 1 host
  type 2 chassis
  type 3 rack
  type 4 row
  type 5 pdu
  type 6 pod
  type 7 room
  type 8 datacenter
  type 9 region
  type 10 root
  ```

  ------

  ##### 3.4 bucket

  所有的中间节点就叫做bucket，bucket可以是一些devices的集合也可以是低一级的buckets的集合, 根节点称为root是整个集群的入口， bucket的id必须是负数且唯一，一个bucket在crush map 实际存储位置是 *buckets[-1-(bucket id)]* ， 后面分析代码的时候会细说

  ```
  # buckets
  host host1 {
      id -2        # do not change unnecessarily
      id -3 class ssd        # do not change unnecessarily
      # weight 3.000
      alg straw2                      # 指定bucket选择的时候使用的crush算法
      hash 0    # rjenkins1             # 指定hash的算法
      item osd.0 weight 1.000
      item osd.1 weight 1.000
      item osd.2 weight 1.000
  }
  root default {
      id -1        # do not change unnecessarily
      id -4 class ssd        # do not change unnecessarily
      # weight 3.000
      alg straw2
      hash 0    # rjenkins1
      item host1 weight 3.000
  }
  ```

  ------

  ##### 3.5 rule

  **placement rule** 相当于crush的行为准则，crush做的每一步都是按照rule规则来执行

  ```
  rule <rulename> {
      ruleset <ruleset>
      type [replicated|erasure]
      min_size <min-size>
      max_size <max-size>
      step take <bucket-name>
      step select [choose|chooseleaf] [firstn|indep] <num> type <bucket-type>
      step emit
  }
  ```

  > - ruleset : 相当于rule的id
  > - type : 存储池pool的类型，是副本还是纠删码
  > - min_size : 如果副本数小于这个数值，就不会应用这条rule
  > - max_size : 如果副本数大于这个数值，就不会应用这条rule
  > - step take : crush规则的入口，一般是类型为root的bucket
  > - step sleect : 分为choose 和chooseleaf两种， num 代表选择的数量，bucket-type是预期的bucket类型
  > - step emit : 代表从take开始到这个操作结束。

  这里着重讲解下rule的step select，**需要注意的是select开始的起点，是上一个step的输出**

  select 分为两种

  - *choose* : choose 在选择到预期类型的bucket后就到此结束，进行下一个select操作
  - *chooseleaf* ： chooseleaf 在选择到预期的bucket后会继续递归选到osd
  - *firstn 和indep* : 都是深度优先遍历算法，主要区别在于如果选择的num为4，如果无法选够4个结果的时候 *firstn* 会返回[1,2,4] 这种结果，而indep会返回[1,2,CRUSH_ITEM_NONE,4], 一半情况下都是使用*firstn*
  - num : 是期望选择的数量，如果是0 代表选择 *repnum* （副本数） 个，如果是负数， 代表选择 *|repnum - num|* 个。比如 *repnum = 3， num = -1* ，代表选择 *|3 - 1| = 2* 个

  回到我们的crush map

  ```
  rule replicated_rule {
      id 0
      type replicated
      min_size 1
      max_size 10
      step take default
      step choose firstn 0 type osd
      step emit
  }
  ```

  *step choose firstn 0 type osd* 代表选择3个类型为osd的结果，因为只有一个host,且故障域为osd

  其实crush map 的rule还可以写成下面这个样子

  ```
  rule replicated_rule2 {
      id 0
      type replicated
      min_size 1
      max_size 10
      step take default
      step choose firstn 1 type host      // 多了这一步
      step choose firstn 0 type osd
      step emit
  }
  ```

  得到的结果是一样的，这里的区别在什么地方呢？

  - *replicated_rule* ：每一次选择osd的时候，都是从root入口的，所以每次都需要先选择一个host，再选择一个osd，这里每一次host的选择都是重复的,这一种最少需要6次才能得到结果。
  - *replicated_rule2* : 从root进入，第一次选择一个host，这个host就确定下来了，在这个host上面找3个osd，这一种最少需要4次就能找到结果

  ![image](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/diff-rule.png)

  整个流程示意如上，所以 **不同的rule规则可以得到相同的结果，但是计算的次数是不一样的，消耗的计算资源自然就有所不同**

------

- #### 4. crush rule 实践

  ##### 4.1 例子1，以官方的crush图片为例，写出相应的rule规则

  ![image](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/placemen%20rule%E4%BE%8B%E5%AD%901.png)

  从图中可以看到，三副本必须 分布在同一个row 的不同三个cabinet上， crush rule如下

  ```
  rule replicated_rule1 {
      ruleset 0
      type replicated
      min_size 1
      max_size 10
      step take root
      step choose firstn 1 type row
      step choose firstn 3 type cabinet
      step choose firstn 1 type osd
      step emit
  }
      
  或者
      
  rule replicated_rule1 {
      ruleset 0
      type replicated
      min_size 1
      max_size 10
      step take root
      step choose firstn 1 type row
      step chooseleaf firstn 3 type cabinet 
      step emit
  }
  ```

  **Important**: 可以看到 **chooseleaf firstn type** 等同于下面两个操作

  - a) choose firstn type
  - b) step choose firstn 1 type osd

  ------

  ##### 4.2 例子2 集群总共三台主机，其中一台为纯ssd主机，另外两台为HDD主机，要求主副本分布在ssd 上，其他副本分布在hdd上

  cursh map 结构如下图所示![image](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/crush_map_example2.png)

  的出来的crush rule 为

  ```
  rule ssd_primary {
      ruleset 1
      type replicated
      min_size 1
      max_size 10
      step take rep_ssd                   // 首先从rep_ssd 入口 
      step chooseleaf firstn 1 type host  // 选择一个ssd host并找到一个osd作为主osd
      step emit                           // 结束第一次查找
      step take rep_hdd                   // 从rep_hdd 入口欧
      step chooseleaf firstn -1 type host // 选择两个HDD host并各找到一个osd作为2个副本osd
      step emit                           // 结束
  }
  ```

### 三. 总结

------

- rule规则中 select开始的起点，是上一个step的输出
- 不同rule规则的设定对计算资源的消耗是有差异的
- object 映射到osd的计算过程是在客户端完成的，计算出结果后，客户端直接与对应的osd联系，读取数据，这也是ceph高性能的原因之一

### 四. 参考资料

------

- 《ceph源码分析》– 常涛
- 《ceph之rados设计原理和实现》
- ceph 源码

  摘自：![](https://www.dovefi.com/post/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3crush1%E7%90%86%E8%A7%A3crush_map%E6%96%87%E4%BB%B6/)
