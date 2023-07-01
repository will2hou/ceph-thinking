### 一. 前言

------

上一篇[《深入理解crush(3)—Object至PG映射源码分析》](https://www.dovefi.com/post/深入理解crush3object至pg映射源码分析/)，分析了 Object至PG的过程，接下来的一篇是本系列 **最重要** 的一部分，也是crush的核心，**crush算法**

### 二. crush的基本数据结构

------

在开始分析代码之前，先温习下测试集群的crush map，因为crush 算法是完全按照crush map进行运算的

- #### 1. 查看测试集群的crush map

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

  接下来，看看ceph 是如何处理 crush map的。

  ------

- #### 2. crush map 基本数据结构 和crush 算法的介绍

  - **2.1、crush map 基本数据结构**

  crush 算法的实现源码，是比较独立的一部分，相比ceph其他模块的源码也简单很多，这里先介绍一下，crush 模块的文件说明

  crush 模块的源码路径为 **src/crush**, 其中

  > - **crush.h 和 crush.c** : *crush map* 的基本数据结构
  > - **build.h 和 build.c** : 实现了如何构造 *crush_map* 数据结构
  > - **CrushCompiler.h 和 CrushCompiler.cc** : 解析 *crush_map* 的词法和语义，相当于翻译crush map 文件
  > - **CrushWarpper** : 是CRUSH核心实现的封装
  > - **mapper.h 和 mapper.c** : CRUSH 算法的核心实现

  还记得在第一篇博文中提到过crush map 由5部分组成：tunable 参数, device, type, bucket, rule，在代码中的结构是这样的

  **(图片太小右键新标签页打开查看)**

  ![image](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/crush%20map%20struct.jpg)

  图片中说明的差不多了，这里对几个比较重要的变量再做一些解释

  **struct crush_map**

  ```
  struct crush_map {
      /*
      *crush map 中的所有bucket都保存在这个变量中，
      * buckets[i] 下标i跟crush map中bucket的id是有关系的，对应的关系是
      * -1 - i = bucket_id. 反过来就可以根据bucket id招到对应的bucket实体了
      * bucket的删除 必须使用 crush_remove_bucket()函数操作, 因为可以
      * 删除，所以buckets[i]可能会有NULL的情况 
      * bucket的添加 必须使用 crush_add_bucket()
      * */
      struct crush_bucket **buckets;
      // 所有的rule规则
      struct crush_rule **rules;
      .
      .
      .
  }
  ```

  **struct crush_bucket**

  ```
  struct crush_bucket {
      /* items 保存的是子bucket 的id , 如果是小于0的代表是bucket，
      大于等于0代表是osd */
      __s32 *items;
      .
      .
      .
  }
  ```

  - **2.2、CRUSH 算法介绍**

  函数流程如下：

  ![image](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/do_rule.png)

  这里简单说明下crush选择bucket的过程

  crush 伪随机算法的决定性参数有3个：

  - hash(pgid, poolid)值 *pps* 这里称为 **x**, 是固定不变的，
  - 随机因子 **r** , **r** 初始值为 *0* ，在选择出一个目标bucket或者osd之后，会 *+1* ， 如果选择出来的有冲突，r 也还会加1，目的是为了选择不同的结果。
  - 还有一个就是osd的reweight 值，在选出osd后，会针对osd的reweight再做一次计算，决定了选中的概率。

  为了防止出现一直无法选择上的而出现死循环，需要对尝试次数做限制，由 **choos_total_tries** 决定， 同时故障域模式下会有递归调用，如果再递归调用的时候还以 **choos_total_tries** 作为尝试限制的话，尝试次数就会成倍增长了，所以，L版的算法中是通过 **chooseleaf_descend_once** 布尔值来决定是 **被调用者** 是否进行重试的。

  *这里借用《CEPH 之rados 设计原理和实现》一书中的示意图解析firstn选择的过程*![image](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/firstn.png)

### 三. CRUSH 算法源码分析

------

- #### 1. 函数详解

  在[深入理解crush(3)—Object至PG映射源码分析](https://www.dovefi.com/post/深入理解crush3object至pg映射源码分析/)这一篇中，已经获取 *hash(pgid, poolid)* 的hash 值 **pps**

  - **1.1、_pg_to_raw_osds**

    

  ```
  void OSDMap::_pg_to_raw_osds(
    const pg_pool_t& pool, pg_t pg,
    vector<int> *osds,
    ps_t *ppps) const
  {
    // map to osds[]
    ps_t pps = pool.raw_pg_to_pps(pg);  // placement ps 到此就获取了 由 pgid + poolid 的hash值，可以唯一确定PG
    unsigned size = pool.get_size();
      
    // what crush rule?
    int ruleno = crush->find_rule(pool.get_crush_rule(), pool.get_type(), size);  //   根据pool 获取crush 中的rule id 为下一步 pg 映射 osd 做准备
    if (ruleno >= 0)
      /* osd_weight 是所有osd reweight的值，0x10000 = "in", 0 = "out" 
      *在 is_out检测的时候需要进行检测，并且决定了osd被选中的概率
      **/
      crush->do_rule(ruleno, pps, *osds, size, osd_weight, pg.pool());
      
    _remove_nonexistent_osds(pool, *osds);
      
    if (ppps)
      *ppps = pps;
  }
  ```

  接着*crush->do_rule(ruleno, pps,* **osds, size, osd_weight, pg.pool())* 正式开始计算 PG 到 osd的映射

  这里需要关注一个参数，就是 **osd_weight**, 这是所有 **osd reweight** 的集合，在后面算法选择的时候会用到,这里测试集群三个osd 的reweigh存储为{65536, 65536, 65536}

  ------

  - **1.1.1、do_rule**

  ```
  void do_rule(int rule, int x, vector<int>& out, int maxout,
             const WeightVector& weight,
             uint64_t choose_args_index) const {
      int rawout[maxout];
      char work[crush_work_size(crush, maxout)];
      crush_init_workspace(crush, work); // 初始化crush 的工作空间
      crush_choose_arg_map arg_map = choose_args_get_with_fallback(
        choose_args_index);   
      int numrep = crush_do_rule(crush, rule, x, rawout, maxout, &weight[0],
                     weight.size(), work, arg_map.args); // 开始根据crush 和pps 计算
      if (numrep < 0)
        numrep = 0;
      out.resize(numrep);
      for (int i=0; i<numrep; i++)
        out[i] = rawout[i];
    }
  ```

  *crush_do_rule* 是crush map的中rule的基本处理函数，会根据rule 的step 一步步执行，

  ------

  - **1.1.1.1、crush_do_rule**

  ```
  /**
   * crush_do_rule - calculate a mapping with the given input and rule
   * @map: the crush_map
   * @ruleno: the rule id
   * @x: hash input
   * @result: pointer to result vector
   * @result_max: maximum result size
   * @weight: weight vector (for map leaves)    // 叶子节点就是osd
   * @weight_max: size of weight vector    
   * @cwin: Pointer to at least map->working_size bytes of memory or NULL.
   */
  int crush_do_rule(const struct crush_map *map,
            int ruleno, int x, int *result, int result_max,
            const __u32 *weight, int weight_max,
            void *cwin, const struct crush_choose_arg *choose_args)
  {
      int result_len;
      struct crush_work *cw = cwin;
      int *a = (int *)((char *)cw + map->working_size);
      int *b = a + result_max;
      int *c = b + result_max;
      int *w = a;
      int *o = b;
      int recurse_to_leaf;    // 是否递归到叶子节点
      int wsize = 0;
      int osize;              // 当前step 选择出来的结果数量
      int *tmp;
      const struct crush_rule *rule;
      __u32 step;
      int i, j;
      int numrep;
      int out_size;
      /*
       * the original choose_total_tries value was off by one (it
       * counted "retries" and not "tries").  add one.
       * crush map 文件中的choose_total_tries变量是重试的次数，所以总次数需要+1
       */
      int choose_tries = map->choose_total_tries + 1;
      int choose_leaf_tries = 0;
      /*
       * the local tries values were counted as "retries", though,
       * and need no adjustment
       */
      int choose_local_retries = map->choose_local_tries;
      int choose_local_fallback_retries = map->choose_local_fallback_tries;
      
      int vary_r = map->chooseleaf_vary_r;
      int stable = map->chooseleaf_stable;
      
      if ((__u32)ruleno >= map->max_rules) {
          dprintk(" bad ruleno %d\n", ruleno);
          return 0;
      }
      
      rule = map->rules[ruleno];
      result_len = 0;
      // 这里开始循环执行rule的每一步
      for (step = 0; step < rule->len; step++) {
          int firstn = 0;        // 是否使用 firstn 深度优先算法
          const struct crush_rule_step *curstep = &rule->steps[step];
      
          switch (curstep->op) {
          case CRUSH_RULE_TAKE:    // 当op 为 take的时候是没有arg2的
              // 判断参数是否正确，bucket是否存在
              if ((curstep->arg1 >= 0 &&
                   curstep->arg1 < map->max_devices) ||
                  (-1-curstep->arg1 >= 0 &&            
                   -1-curstep->arg1 < map->max_buckets &&    // 这里可以看出 bucket的id 是有顺序的，从-1开始-n，存储在map中是0至于n-1, 
                   map->buckets[-1-curstep->arg1])) {        // The bucket found at __buckets[i]__ must have a crush_bucket.id == -1-i 
                  w[0] = curstep->arg1;    // arg1 就是bucket id， 就是root 的id ，作为下一step开始的点
                  wsize = 1;
              } else {
                  dprintk(" bad take value %d\n", curstep->arg1);
              }
              break;
          // CRUSH_RULE_SET_* 相关的参数都是用来设置crush 参数的
          case CRUSH_RULE_SET_CHOOSE_TRIES:
              if (curstep->arg1 > 0)
                  choose_tries = curstep->arg1;
              break;
      
          case CRUSH_RULE_SET_CHOOSELEAF_TRIES:
              if (curstep->arg1 > 0)
                  choose_leaf_tries = curstep->arg1;
              break;
      
          case CRUSH_RULE_SET_CHOOSE_LOCAL_TRIES:
              if (curstep->arg1 >= 0)
                  choose_local_retries = curstep->arg1;
              break;
      
          case CRUSH_RULE_SET_CHOOSE_LOCAL_FALLBACK_TRIES:
              if (curstep->arg1 >= 0)
                  choose_local_fallback_retries = curstep->arg1;
              break;
      
          case CRUSH_RULE_SET_CHOOSELEAF_VARY_R:
              if (curstep->arg1 >= 0)
                  vary_r = curstep->arg1;
              break;
      
          case CRUSH_RULE_SET_CHOOSELEAF_STABLE:
              if (curstep->arg1 >= 0)
                  stable = curstep->arg1;
              break;
      
          case CRUSH_RULE_CHOOSELEAF_FIRSTN:
          case CRUSH_RULE_CHOOSE_FIRSTN:
              firstn = 1;
              /* fall through */
          case CRUSH_RULE_CHOOSELEAF_INDEP:
          case CRUSH_RULE_CHOOSE_INDEP:
              if (wsize == 0)
                  break;
              // 带有CHOOSELEAF的操作都是要递归到子节点的
              recurse_to_leaf =
                  curstep->op ==
                   CRUSH_RULE_CHOOSELEAF_FIRSTN ||
                  curstep->op ==
                  CRUSH_RULE_CHOOSELEAF_INDEP;
      
              /* reset output */
              osize = 0;        // osize 当前step已经选出来的数量
      
              for (i = 0; i < wsize; i++) {
                  int bno;                // bucket id
                  numrep = curstep->arg1; // 这个numrep 是要选择的个数，可能为负数
                  if (numrep <= 0) {            
                      numrep += result_max;
                      if (numrep <= 0)
                          continue;
                  }
                  j = 0;
                  /* make sure bucket id is valid */
                  bno = -1 - w[i];
                  if (bno < 0 || bno >= map->max_buckets) {
                      // w[i] is probably CRUSH_ITEM_NONE
                      dprintk("  bad w[i] %d\n", w[i]);
                      continue;
                  }
                  if (firstn) {                        // 如果使用的是  firstn 深度优先算法
                      int recurse_tries;
                      if (choose_leaf_tries)
                          recurse_tries =
                              choose_leaf_tries;
                      else if (map->chooseleaf_descend_once)    // 这里一直都是设置为1的，因为会造成一些边界问题
                          recurse_tries = 1;
                      else
                          recurse_tries = choose_tries;
                      osize += crush_choose_firstn(
                          map,
                          cw,
                          map->buckets[bno],
                          weight, weight_max,
                          x, numrep,
                          curstep->arg2,
                          o+osize, j,
                          result_max-osize,
                          choose_tries,
                          recurse_tries,
                          choose_local_retries,
                          choose_local_fallback_retries,
                          recurse_to_leaf,
                          vary_r,
                          stable,
                          c+osize,
                          0,
                          choose_args);
                  } else {
                      out_size = ((numrep < (result_max-osize)) ?
                              numrep : (result_max-osize));
                      crush_choose_indep(
                          map,
                          cw,
                          map->buckets[bno],
                          weight, weight_max,
                          x, out_size, numrep,
                          curstep->arg2,
                          o+osize, j,
                          choose_tries,
                          choose_leaf_tries ?
                             choose_leaf_tries : 1,
                          recurse_to_leaf,
                          c+osize,
                          0,
                          choose_args);
                      osize += out_size;
                  }
              }
      
              if (recurse_to_leaf)
                  /* copy final _leaf_ values to output set */
                  memcpy(o, c, osize*sizeof(*o));
      
              /* swap o and w arrays */
              tmp = o;
              o = w;
              w = tmp;        // 上一step输出的结果，作为下一step的开始，在上一步选择好的基础上在进行下一步的选择
              wsize = osize;
              break;
      
      
          case CRUSH_RULE_EMIT:
              for (i = 0; i < wsize && result_len < result_max; i++) {
                  result[result_len] = w[i];
                  result_len++;
              }
              wsize = 0;
              break;
      
          default:
              dprintk(" unknown op %d at step %d\n",
                  curstep->op, step);
              break;
          }
      }
      
      return result_len;
  }
  ```

  crush_do_rule 会根据每一步step 执行，这里特别需要注意的是， **当前step的起点都是在上一step的得出的结果下开始执行的**

  ```
  /* swap o and w arrays */
          tmp = o;
          o = w;
          w = tmp;        // 上一step输出的结果，作为下一step的开始，在上一步选择好的基础上在进行下一步的选择
          wsize = osize;
          break;
  ```

  还记得第一篇中说的不同rule规则的定制可以得到相同的结果但是计算的次数会不一样，就是因为当前step执行的起点不一样

  ![image](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/diff-rule.png)

  ------

  - **1.1.1.1.1、crush_choose_firstn** ：firstn 算法的入口函数

  可以说是比较重要的一步了，基本上的逻辑处理，冲突检测等都在这里

  ```
  /**
   * crush_choose_firstn - choose numrep distinct items of given type
   * @map: the crush_map
   * @bucket: the bucket we are choose an item from
   * @x: crush input value
   * @numrep: the number of items to choose
   * @type: the type of item to choose
   * @out: pointer to output vector
   * @outpos: our position in that vector
   * @out_size: size of the out vector
   * @tries: number of attempts to make
   * @recurse_tries: number of attempts to have recursive chooseleaf make
   * @local_retries: localized retries    
   * @local_fallback_retries: localized fallback retries    
   * @recurse_to_leaf: true if we want one device under each item of given type (chooseleaf instead of choose)
   * @stable: stable mode starts rep=0 in the recursive call for all replicas
   * @vary_r: pass r to recursive calls
   * @out2: second output vector for leaf items (if @recurse_to_leaf)
   * @parent_r: r value passed from the parent
   */
  static int crush_choose_firstn(const struct crush_map *map,
                     struct crush_work *work,
                     const struct crush_bucket *bucket,
                     const __u32 *weight, int weight_max,
                     int x, int numrep, int type,
                     int *out, int outpos,
                     int out_size,
                     unsigned int tries,
                     unsigned int recurse_tries,
                     unsigned int local_retries,
                     unsigned int local_fallback_retries,
                     int recurse_to_leaf,
                     unsigned int vary_r,
                     unsigned int stable,
                     int *out2,
                     int parent_r,
                                 const struct crush_choose_arg *choose_args)
  {
      int rep;    // 计数器，用来记录已经选择的数量
      unsigned int ftotal, flocal;
      int retry_descent, retry_bucket, skip_rep;
      const struct crush_bucket *in = bucket;
      int r;
      int i;
      int item = 0;
      int itemtype;
      int collide, reject;
      int count = out_size;
      
      dprintk("CHOOSE%s bucket %d x %d outpos %d numrep %d tries %d \
  recurse_tries %d local_retries %d local_fallback_retries %d \
  parent_r %d stable %d\n",
          recurse_to_leaf ? "_LEAF" : "",
          bucket->id, x, outpos, numrep,
          tries, recurse_tries, local_retries, local_fallback_retries,
          parent_r, stable);
      
      for (rep = stable ? 0 : outpos; rep < numrep && count > 0 ; rep++) {
          /* keep trying until we get a non-out, non-colliding item */
          ftotal = 0;                    // fail total 失败的总次数
          skip_rep = 0;                // 是否跳过这一次选择
          do {
              retry_descent = 0;
              in = bucket;              /* initial bucket */
      
              /* choose through intervening buckets */
              flocal = 0;                // 当前bucket的选择重试的次数，局部重试次数
              do {
                  collide = 0;        // 判断是否有冲撞
                  retry_bucket = 0;
                  r = rep + parent_r;        // 随机因子r
                  /* r' = r + f_total */
                  r += ftotal;            // 如果选择失败，这里要加上失败次数再进行重试
      
                  /* bucket choose */
                  if (in->size == 0) {
                      reject = 1;
                      goto reject;
                  }
                  if (local_fallback_retries > 0 &&
                      flocal >= (in->size>>1) &&
                      flocal > local_fallback_retries)
                      item = bucket_perm_choose(        // 这是一个后备选择算法，会记录之前冲突过的item，触发的条件比较苛刻
                          in, work->work[-1-in->id],
                          x, r);
                  else
                      item = crush_bucket_choose(    // 这里从输入的bucket中选择一个item 出来，item 就是bucket的id 号
                          in, work->work[-1-in->id],
                          x, r,
                                                  (choose_args ? &choose_args[-1-in->id] : 0),
                                                  outpos);
                  if (item >= map->max_devices) {        // 如果选出来的item id 比 devices个数还大肯定是错误的
                      dprintk("   bad item %d\n", item);
                      skip_rep = 1;
                      break;
                  }
      
                  /* desired type? */
                  if (item < 0)    // bucket id 都是小于0的，如果不是那选出来的就是osd
                      itemtype = map->buckets[-1-item]->type;    
                  else
                      itemtype = 0;    // 不然的话就是osd 类型
                  dprintk("  item %d type %d\n", item, itemtype);
      
                  /* keep going? */
                  if (itemtype != type) {        // 如果选出来的bucket type 跟预期的bucket type不一样
                      if (item >= 0 ||
                          (-1-item) >= map->max_buckets) {
                          dprintk("   bad item type %d\n", type);
                          skip_rep = 1;
                          break;
                      }
                      in = map->buckets[-1-item];    // 将刚刚找到的bucket作为下一次查找的输入（递归）
                      retry_bucket = 1;        // 重新选择
                      continue;
                  }
                  // 到这一步证明找到的是目标类型的bucket或者osd，跟已经找到的进行对比，如果冲突那么需要重新查找
                  /* collision? */        
                  for (i = 0; i < outpos; i++) {
                      if (out[i] == item) {
                          collide = 1;        // 判断选择的是否冲突
                          break;
                      }
                  }
      
                  reject = 0;
                  if (!collide && recurse_to_leaf) { // 如果选出来的bucket不冲突，并且需要递归到叶节点osd
                      if (item < 0) {                // 如果是bucket类型的
                          int sub_r;
                          if (vary_r)
                              sub_r = r >> (vary_r-1);
                          else
                              sub_r = 0;
                          if (crush_choose_firstn(
                                  map,
                                  work,
                                  map->buckets[-1-item],            // 注意这里入口变成了刚刚选出来的bucket
                                  weight, weight_max,
                                  x, stable ? 1 : outpos+1, 0,
                                  out2, outpos, count,
                                  recurse_tries, 0,
                                  local_retries,
                                  local_fallback_retries,
                                  0,
                                  vary_r,
                                  stable,
                                  NULL,
                                  sub_r,
                                                              choose_args) <= outpos)
                              /* didn't get leaf */
                              reject = 1;
                      } else {                // osd
                          /* we already have a leaf! */
                          out2[outpos] = item;        // 这个是应用在需要递归到叶子节点的输出
                                  }
                  }
      
                  if (!reject && !collide) {
                      /* out? */
                      if (itemtype == 0)
                          reject = is_out(map, weight,    // 进行osd reweight 的再次过滤
                                  weight_max,
                                  item, x);
                  }
      
  reject:
                  if (reject || collide) {    // 如果‘冲突‘或者‘故障‘了，那就重新查找随机因子 r 递增
                      ftotal++;
                      flocal++;
      
                      if (collide && flocal <= local_retries)    // 如果再当前bucket下重试次数还达到上限local_retries
                          /* retry locally a few times */
                          retry_bucket = 1;
                      else if (local_fallback_retries > 0 &&
                           flocal <= in->size + local_fallback_retries)
                          /* exhaustive bucket search */
                          retry_bucket = 1;
                      else if (ftotal < tries)            
                          /* then retry descent */
                          retry_descent = 1;
                      else
                          /* else give up */
                          skip_rep = 1;
                      dprintk("  reject %d  collide %d  "
                          "ftotal %u  flocal %u\n",
                          reject, collide, ftotal,
                          flocal);
                  }
              } while (retry_bucket);        // 在当前bucket下重试选择（局部重试），每一次都从头开始是很消耗资源的
          } while (retry_descent);        // 从最开始的bucket处开始重新选择（从头开始）
      
          if (skip_rep) {
              dprintk("skip rep\n");
              continue;
          }
      
          dprintk("CHOOSE got %d\n", item);
          out[outpos] = item;
          outpos++;
          count--;
  #ifndef __KERNEL__
          if (map->choose_tries && ftotal <= map->choose_total_tries)
              map->choose_tries[ftotal]++;
  #endif
      }
      
      dprintk("CHOOSE returns %d\n", outpos);
      return outpos;
  }
  ```

  代码中已经加入了比较详细的注释，还是比较容易理解的

  ------

  - **1.1.1.1.1、crush_bucket_choose**

  ```
  static int crush_bucket_choose(const struct crush_bucket *in,
                 struct crush_work_bucket *work,
                 int x, int r,
                             const struct crush_choose_arg *arg,
                             int position)
  {
      dprintk(" crush_bucket_choose %d x=%d r=%d\n", in->id, x, r);
      BUG_ON(in->size == 0);
      switch (in->alg) {
      case CRUSH_BUCKET_UNIFORM:
          return bucket_uniform_choose(
              (const struct crush_bucket_uniform *)in,
              work, x, r);
      case CRUSH_BUCKET_LIST:
          return bucket_list_choose((const struct crush_bucket_list *)in,
                        x, r);
      case CRUSH_BUCKET_TREE:
          return bucket_tree_choose((const struct crush_bucket_tree *)in,
                        x, r);
      case CRUSH_BUCKET_STRAW:
          return bucket_straw_choose(
              (const struct crush_bucket_straw *)in,
              x, r);
      case CRUSH_BUCKET_STRAW2:
          return bucket_straw2_choose(
              (const struct crush_bucket_straw2 *)in,
              x, r, arg, position);
      default:
          dprintk("unknown bucket %d alg %d\n", in->id, in->alg);
          return in->items[0];
      }
  }
  
  ```

  函数里面就是根据crush map 中指定的算法进行相应选择，L版之后默认都是使用straw2算法了，这里先暂时不深究算法的实现。后面有时间好好研究一下PG分裂和straw2算法吧，当选出osd之后，还需要对osd进行 **reweight** 的过滤， 是在 **is_out** 函数中实现的

  ------

  - **1.1.1.1.1.1 、is_out** ： 进行 osd reweight 的再次过滤

  ```
  /*
   * true if device is marked "out" (failed, fully offloaded)
   * of the cluster
   * weight 是reweight, weight_max 是osd个数
   */
  static int is_out(const struct crush_map *map,
            const __u32 *weight, int weight_max,
            int item, int x)
  {
      if (item >= weight_max)            // 说明不存在这个osd
          return 1;
      if (weight[item] >= 0x10000)    // reweight 为1 
          return 0;
      if (weight[item] == 0)            // reweight 为 0 
          return 1;
      if ((crush_hash32_2(CRUSH_HASH_RJENKINS1, x, item) & 0xffff)    // 原来的item再hash一次，然后‘与‘操作截取最后的32bit数字，
          < weight[item])                                                // 跟 reweight 做比较来决定是否要用这个osd，                                                                    // 这里可以看出，reweight越大越容易选上
          return 0;
      return 1;
  }
  
  ```

  经过一系列的选择最终就可以得到 osd 列表了

### 四. 总结

------

最后用我们测试集群的rule还原一下选择的过程，看看crushmap有没有什么优化的空间

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

- \1. 从 root 开始选择bucket，首先root的item是host，所以选择了一个host出来，发现不是想要的type （osd），那就递归，从选出来的host开始，继续找，找到了一个osd
- \2. 对找到的这个osd，进行冲突检测，看看是不是已经选过了，没有选过，再进行reweight的过滤，因为osd默认reweight是 1，所以这里也就走走过场，就是这个osd了，将这个osd放到输出数组中。
- \3. 开始选择第二个osd, 这里重复 第一，第二步骤。

看看，因为我们只有一个host，所以第一步骤是不是重复的做了三次? 如果我们将选出来的host作为每一次选择osd的入口点，这样是不是就不需要重复去找host啦？理解完选择的原理后，发挥想象力，看看如何定制crushmap能做到更高效，更可靠吧。

### 五. 参考文档

- 《ceph 值rados设计原理与实现》
- 《ceph 源码分析》
- [ceph 源码v12.2.8](https://github.com/ceph/ceph/tree/v12.2.8/src/crush)

  摘自：[深入理解ceph crush(4)—PG至OSD的crush算法源码分析](https://www.dovefi.com/post/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3crush4pg%E8%87%B3osd%E7%9A%84crush%E7%AE%97%E6%B3%95%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
