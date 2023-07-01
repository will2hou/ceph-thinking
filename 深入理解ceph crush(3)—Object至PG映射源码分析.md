### 一. 前言

------

在上一篇[《深入理解crush(2)—-手动编译ceph集群并使用librados读写文件》](https://www.dovefi.com/post/深入理解crush2手动编译ceph集群并使用librados读写文件/)博文中，初步使用了c语言客户端程序 *rados_write* ，写入文件到ceph测试集群中，现在开始通过使用gdb debug客户端程序 *rados_write* 的整个写入流程，来分析crush的计算过程。

ceph rados对象的映射过程分为两个阶段： - 第一阶段：object 到PG的映射 - 第二阶段：PG 到OSD的映射

![image](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/object_to_osd.png)

考虑到篇幅的问题，这一篇博文先分析第一阶段，**object 到PG的映射**

### 二. Object 到 PG的映射

------

接下来的函数分析会非常的冗长，所以这里先将 **object 到PG的映射** 的流程图先贴出来，方便对照着流程图去分析代码。

![image](https://dovefi-1256247019.cos.ap-guangzhou.myqcloud.com/ceph/object_to_pg%E5%87%BD%E6%95%B0%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

- #### 1. GDB的基本设置,安装gdb调试的一些依赖包

  在此之前，需要学会gdb的基本使用，gdb的使用比较简单，网上资料也有比较多的，这里就不在详细说明,

  设置gdb的基本配置

  ```
  # cat ~/.gdbinit
  set history save on    
  set print pretty on 
  ```

  > *set print pretty on* : 为了美化输出

  gdb debug程序的时候会依赖很多包，需要配置debuginfo 的yum 源，并的下载gdb提示的相应依赖

  ```
  # cat /etc/yum.repos.d/CentOS-Debug.repo
  [debug]
  name=CentOS-$releasever - DebugInfo
  baseurl=http://debuginfo.centos.org/$releasever/$basearch/
  gpgcheck=0
  enabled=1
  protect=1
  priority=1
  ```

  debug *rados_write* 程序需要的依赖

  ```
  # debuginfo-install glibc-2.17-260.el7.x86_64 \
  libblkid-2.23.2-59.el7.x86_64 libgcc-4.8.5-36.el7.x86_64 \
  libibverbs-1.1.8mlnx1-OFED.3.3.0.0.9.33100.x86_64 \
  libnl-1.1.4-3.el7.x86_64 librados2-12.2.8-0.el7.x86_64 \
  libstdc++-4.8.5-36.el7.x86_64 \
  libuuid-2.23.2-59.el7.x86_64 lttng-ust-2.4.1-4.el7.x86_64 \
  nspr-4.19.0-1.el7_5.x86_64 nss-3.36.0-7.el7_5.x86_64 \
  nss-softokn-3.36.0-5.el7_5.x86_64 \
  nss-softokn-freebl-3.36.0-5.el7_5.x86_64 \
  nss-util-3.36.0-1.el7_5.x86_64 sqlite-3.7.17-8.el7.x86_64 \
  userspace-rcu-0.7.16-1.el7.x86_64 \
  zlib-1.2.7-18.el7.x86_64 \
      
  ```

- #### 2. 分析object 到 PG的函数栈

  接下来会将debug的整个过程都记录下来，每一步加上解释和说明，所以篇幅会特别的长，

  使用gdb 调试 *rados_write* 程序

  ```
  # gdb ./rados_write
  GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-110.el7
  Copyright (C) 2013 Free Software Foundation, Inc.
  License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
  This is free software: you are free to change and redistribute it.
  There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
  and "show warranty" for details.
  This GDB was configured as "x86_64-redhat-linux-gnu".
  For bug reporting instructions, please see:
  <http://www.gnu.org/software/gdb/bugs/>...
  Reading symbols from /home/user/CEPH_related/librados_example/rados_write...done.
  ```

  启动程序后，需要设置断点，这里选择的是 *crush_do_rule* 函数，因为这个函数是 **object–>到PG** 流程的终点，而且这个函数太出名了，笔者也是通过学习其他人的博文，才选择了这个函数断点。

  ```
  (gdb) b crush_do_rule       // 设置断点
  Function "crush_do_rule" not defined.
  Make breakpoint pending on future shared library load? (y or [n]) y
  Breakpoint 1 (crush_do_rule) pending.
  (gdb) run                   // 开始运行程序
  Starting program: /home/user/CEPH_related/librados_example/./rados_write
  [Thread debugging using libthread_db enabled]
  Using host libthread_db library "/lib64/libthread_db.so.1".
  [New Thread 0x7fffeb651700 (LWP 3600)]
      
  Created a cluster handle.
      
  Read the config file.
      
  Read the command line arguments.
  [New Thread 0x7fffea6d4700 (LWP 3601)]
  [New Thread 0x7fffe9ed3700 (LWP 3602)]
  [New Thread 0x7fffe96d2700 (LWP 3603)]
  [New Thread 0x7fffe8ed1700 (LWP 3604)]
  [New Thread 0x7fffdbfff700 (LWP 3605)]
  [New Thread 0x7fffdb7fe700 (LWP 3606)]
  [New Thread 0x7fffdaffd700 (LWP 3607)]
  [New Thread 0x7fffda7fc700 (LWP 3608)]
  [New Thread 0x7fffd9ffb700 (LWP 3609)]
  [New Thread 0x7fffd97fa700 (LWP 3610)]
  [New Thread 0x7fffd8ff9700 (LWP 3611)]
  [New Thread 0x7fffb7fff700 (LWP 3612)]
  [New Thread 0x7fffb77fe700 (LWP 3613)]
      
  Connected to the cluster.
      
  Created I/O context.
  // 在断点处卡住
  Breakpoint 1, crush_do_rule (map=0x7fffcc010040, ruleno=0, x=-2119696504, result=0x7fffffffd5d8, result_max=3, weight=0x7fffcc00fea0,
      weight_max=3, cwin=0x7fffffffd508, choose_args=0x0) at /home/user/ceph/src/crush/mapper.c:887
  887    {
  Missing separate debuginfos, use: debuginfo-install libibverbs-1.1.8mlnx1-OFED.3.3.0.0.9.33100.x86_64
  (gdb) bt                // 查看当前的函数堆栈
  #0  crush_do_rule (map=0x7fffcc010040, ruleno=0, x=-2119696504, result=0x7fffffffd5d8, result_max=3, weight=0x7fffcc00fea0,
      weight_max=3, cwin=0x7fffffffd508, choose_args=0x0) at /home/user/ceph/src/crush/mapper.c:887
  #1  0x00007fffeee59274 in do_rule<std::vector<unsigned int, mempool::pool_allocator<(mempool::pool_index_t)15, unsigned int> > > (
      choose_args_index=<optimized out>, weight=std::vector of length 3, capacity 4 = {...}, maxout=3,
      out=std::vector of length 0, capacity 0, x=-2119696504, rule=<optimized out>, this=<optimized out>)
      at /home/user/ceph/src/crush/CrushWrapper.h:1502
  #2  OSDMap::_pg_to_raw_osds (this=this@entry=0x78b960, pool=..., pg=..., osds=osds@entry=0x7fffffffd6d0, ppps=ppps@entry=0x7fffffffd6c4)
      at /home/user/ceph/src/osd/OSDMap.cc:2061
  #3  0x00007fffeee5b036 in OSDMap::_pg_to_up_acting_osds (this=0x78b960, pg=..., up=up@entry=0x7fffffffd890,
      up_primary=up_primary@entry=0x7fffffffd834, acting=acting@entry=0x7fffffffd8b0, acting_primary=acting_primary@entry=0x7fffffffd838,
      raw_pg_to_pg=raw_pg_to_pg@entry=true) at /home/user/ceph/src/osd/OSDMap.cc:2295
  #4  0x00007ffff7b42377 in pg_to_up_acting_osds (acting_primary=0x7fffffffd838, acting=0x7fffffffd8b0, up_primary=0x7fffffffd834,
      up=0x7fffffffd890, pg=..., this=<optimized out>) at /home/user/ceph/src/osd/OSDMap.h:1159
  #5  Objecter::_calc_target (this=this@entry=0x78b2b0, t=t@entry=0x7acf18, con=con@entry=0x0, any_change=any_change@entry=false)
      at /home/user/ceph/src/osdc/Objecter.cc:2846
  #6  0x00007ffff7b4ff89 in Objecter::_op_submit (this=this@entry=0x78b2b0, op=op@entry=0x7acef0, sul=..., ptid=ptid@entry=0x7fffffffdcc8)
      at /home/user/ceph/src/osdc/Objecter.cc:2393
  #7  0x00007ffff7b5d128 in Objecter::_op_submit_with_budget (this=this@entry=0x78b2b0, op=op@entry=0x7acef0, sul=...,
      ptid=ptid@entry=0x7fffffffdcc8, ctx_budget=ctx_budget@entry=0x0) at /home/user/ceph/src/osdc/Objecter.cc:2307
  #8  0x00007ffff7b5d3aa in Objecter::op_submit (this=0x78b2b0, op=0x7acef0, ptid=0x7fffffffdcc8, ctx_budget=0x0)
      at /home/user/ceph/src/osdc/Objecter.cc:2274
  #9  0x00007ffff7b0c4f0 in librados::IoCtxImpl::operate (this=this@entry=0x7a1ec0, oid=..., o=o@entry=0x7fffffffe020,
      pmtime=pmtime@entry=0x0, flags=flags@entry=0) at /home/user/ceph/src/librados/IoCtxImpl.cc:715
  #10 0x00007ffff7b17f5a in librados::IoCtxImpl::write (this=this@entry=0x7a1ec0, oid=..., bl=..., len=len@entry=16, off=off@entry=0)
      at /home/user/ceph/src/librados/IoCtxImpl.cc:648
  #11 0x00007ffff7ae241d in rados_write (io=0x7a1ec0, o=<optimized out>, buf=0x401039 "Hello World!", len=16, off=0)
      at /home/user/ceph/src/librados/librados.cc:3691
  #12 0x0000000000400d0c in main (argc=1, argv=0x7fffffffe2e8) at rados_write.c:75
  ```

  从上面的一系列操作下来，得到的函数流程如下

  \```

  1. main
  2. rados_write
  3. librados::IoCtxImpl::operate
  4. Objecter::op_submit
  5. Objecter::_op_submit_with_budget
  6. Objecter::_op_submit
  7. Objecter::_calc_target
  8. pg_to_up_acting_osds
  9. _pg_to_up_acting_osds
  10. _pg_to_raw_osds
  11. do_rule
  12. crush_do_rule ```

  因为这里并不关心librados是如何封装请求，只关心object到pg的计算过程，所以这里决定从 **Objecter::_calc_target** 函数开始debug 整个过程，重新开始，然后再次设置断点

  ------

  - **1、Objecter::_calc_target** : 开始的入口,计算 object的hash值 *ps*

  ```
  (gdb) b Objecter::_calc_target
  (gdb) run
  Breakpoint 1, Objecter::_calc_target (this=this@entry=0x78b280, t=t@entry=0x7ace08, con=con@entry=0x0,
      any_change=any_change@entry=false) at /home/user/ceph/src/osdc/Objecter.cc:2774
  2774    {
  ```

  卡住在断点处，现在我们打开tui跟踪一下代码， **ctl + x + a** 可以切换到tui界面

  ```
     ┌──/home/user/ceph/src/osdc/Objecter.cc───────────────────────────────────────────────────────────────────────────────────────┐
     │2773    int Objecter::_calc_target(op_target_t *t, Connection *con, bool any_change)                                                │
  B+ │2774    {                                                                                                                           │
     │2775      // rwlock is locked                                                                                                       │
     │2776      bool is_read = t->flags & CEPH_OSD_FLAG_READ;                                                                             │
     │2777      bool is_write = t->flags & CEPH_OSD_FLAG_WRITE;                                                                           │
     │2778      t->epoch = osdmap->get_epoch();     // 获取osd当前omap 版本                                                                                     │
     │2779      ldout(cct,20) << __func__ << " epoch " << t->epoch                                                                        │
     │2780                    << " base " << t->base_oid << " " << t->base_oloc                                                           │
     │2781                    << " precalc_pgid " << (int)t->precalc_pgid                                                                 │
     │2782                    << " pgid " << t->base_pgid                                                                                 │
     │2783                    << (is_read ? " is_read" : "")                                                                              │
     │2784                    << (is_write ? " is_write" : "")                                                                            │
     │2785                    << dendl;                                                                                                   │
     │2786                                                                                                                                │
     │2787      const pg_pool_t *pi = osdmap->get_pg_pool(t->base_oloc.pool); // 根据pool id 获取pool实体                                                            │
     │2788      if (!pi) {                                                                                                                │
     │2789        t->osd = -1;                                                                                                            │
     │2790        return RECALC_OP_TARGET_POOL_DNE;                                                                                       │
     │2791      }                                                                                                                         │
    >│2792      ldout(cct,30) << __func__ << "  base pi " << pi                                                                           │
     │2793                    << " pg_num " << pi->get_pg_num() << dendl;                                                                 │
     │2794                                                                                                                                │
     │2795      bool force_resend = false;                                                                                                │
     │2796      if (osdmap->get_epoch() == pi->last_force_op_resend) {                                                                    │
     └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  multi-thre Thread 0x7ffff In: Objecter::_calc_target                                                       Line: 2792 PC: 0x7ffff7b41f6c
  (gdb)
  ```

  这里按 *n* 逐行debug代码， 这里我想显示打印 *pg_pool_t* **p* 和 *op_target_t* **t* 的信息，其中 *pg_pool_t* 是pool的结构体，包含pool相关的所有信息。

  ```
  (gdb) p *pi
  $13 = {
    static APPLICATION_NAME_CEPHFS = 0x7fffef10f7f9 "cephfs",
    static APPLICATION_NAME_RBD = 0x7fffef10598b "rbd",
    static APPLICATION_NAME_RGW = 0x7fffef105954 "rgw",
    flags = 1,
    type = 1 '\001',
    size = 3 '\003',
    min_size = 1 '\001',
    crush_rule = 0 '\000',
    object_hash = 2 '\002',
    pg_num = 8,
    pgp_num = 8,
    properties = std::map with 0 elements,
    erasure_code_profile = "",
    last_change = 17,
    last_force_op_resend = 0,
    last_force_op_resend_preluminous = 0,
    snap_seq = {
      val = 0
    },
    snap_epoch = 0,
    auid = 0,
    crash_replay_interval = 0,
    quota_max_bytes = 0,
    quota_max_objects = 0,
    snaps = std::map with 0 elements,
    removed_snaps = {
      _size = 0,
      m = std::map with 0 elements
    },
    pg_num_mask = 7,
    pgp_num_mask = 7,
    tiers = std::set with 0 elements,
    tier_of = -1,
    read_tier = -1,
    write_tier = -1,
    cache_mode = pg_pool_t::CACHEMODE_NONE,
    target_max_bytes = 0,
    target_max_objects = 0,
    cache_target_dirty_ratio_micro = 400000,
    cache_target_dirty_high_ratio_micro = 600000,
    cache_target_full_ratio_micro = 800000,
    cache_min_flush_age = 0,
    cache_min_evict_age = 0,
    hit_set_params = {
      _vptr.Params = 0x7ffff7dcdb90 <vtable for HitSet::Params+16>,
      impl = {
        px = 0x0
      }
    },
    hit_set_period = 0,
    hit_set_count = 0,
    use_gmt_hitset = true,
    min_read_recency_for_promote = 0,
    min_write_recency_for_promote = 0,
    hit_set_grade_decay_rate = 0,
    hit_set_search_last_n = 0,
    stripe_width = 0,
    expected_num_objects = 0,
    fast_read = false,
    opts = {
      opts = std::map with 0 elements
    },
    application_metadata = std::map with 1 elements = {
      ["rgw"] = std::map with 0 elements
    },
    grade_table = std::vector of length 0, capacity 0
  }
  ```

  而 *op_target_t* 则是整个写入操作封装的结构信息，包含对象的名字，写入pool的id

  ```
  (gdb) p *t
  $19 = {
    flags = 32,
    epoch = 24,
    base_oid = {
      name = "test-write"
    },
    base_oloc = {
      pool = 3,
      key = "",
      nspace = "",
      hash = -1
    },
    target_oid = {
      name = "test-write"
    },
    target_oloc = {
      pool = 3,          
      key = "",          
      nspace = "",        
      hash = -1
    },
    precalc_pgid = false,
    pool_ever_existed = false,
    base_pgid = {
      m_pool = 0,
      m_seed = 0,
      m_preferred = -1,
      static calc_name_buf_size = 36 '$'
    },
    pgid = {
      m_pool = 0,
      m_seed = 0,
      m_preferred = -1,
      static calc_name_buf_size = 36 '$'
    },
    actual_pgid = {
      pgid = {
        m_pool = 0,
        m_seed = 0, 
        m_preferred = -1,
        static calc_name_buf_size = 36 '$'
      },
      shard = {
        id = -1 '\377',
        static NO_SHARD = {
          id = -1 '\377',
          static NO_SHARD = <same as static member of an already seen type>
        }
      },
      static calc_name_buf_size = 40 '('
    },
    pg_num = 0,
    pg_num_mask = 0,
    up = std::vector of length 0, capacity 0,
    acting = std::vector of length 0, capacity 0,
    up_primary = -1,
    acting_primary = -1,
    size = -1,
    min_size = -1,
    sort_bitwise = false,
    recovery_deletes = false,
    used_replica = false,
    paused = false,
    osd = -1,
    last_force_resend = 0
  }
  
  ```

  在这我们看看 *t->target_oloc* 的信息

  ```
  target_oloc = {
      pool = 3,           // 这个pool就是rados_write指定的pool在集群中的id
      key = "",           // 提供的key 名字，因为我们并没有提供key的名字所以为空
      nspace = "",        // namespace 用来区分同个pool的不同命名空间的对象
      hash = -1           // hash的算法      
    }
  
  ```

  > 这里的 *namespace* 会跟object一起作为hash算法的输入，对namespace有疑惑的可以看看这篇博文[Ceph rados namespace的设计](https://www.dovefi.com/post/ceph_rados_namespace设计/)

  继续 n 单步调试，这里我们会进去 *osdmap->object_locator_to_pg* 函数

  ------

  - **1.1、OSDMap::object_locator_to_pg** ： 计算object 到 pg 的 seed，seed 中文翻译为种子

  > 个人认为因为这一步还不是最终的结果，所以这里用的seed 命名

  ```
     ┌──/home/user/ceph/src/osdc/Objecter.cc───────────────────────────────────────────────────────────────────────────────────────┐
     │2812          t->target_oloc.pool = pi->write_tier;                                                                                 │
     │2813        pi = osdmap->get_pg_pool(t->target_oloc.pool);                                                                          │
     │2814        if (!pi) {                                                                                                              │
     │2815          t->osd = -1;                                                                                                          │
     │2816          return RECALC_OP_TARGET_POOL_DNE;                                                                                     │
     │2817        }                                                                                                                       │
     │2818      }                                                                                                                         │
     │2819                                                                                                                                │
     │2820      pg_t pgid;                                                                                                                │
     │2821      if (t->precalc_pgid) {                                                                                                    │
     │2822        assert(t->flags & CEPH_OSD_FLAG_IGNORE_OVERLAY);                                                                        │
     │2823        assert(t->base_oid.name.empty()); // make sure this is a pg op                                                          │
     │2824        assert(t->base_oloc.pool == (int64_t)t->base_pgid.pool());                                                              │
     │2825        pgid = t->base_pgid;                                                                                                    │
     │2826      } else {                                                                                                                  │
     │2827        int ret = osdmap->object_locator_to_pg(t->target_oid, t->target_oloc,                                                   │
    >│2828                                               pgid);                                                                           │
     │2829        if (ret == -ENOENT) {                                                                                                   │
     │2830          t->osd = -1;                                                                                                          │
     │2831          return RECALC_OP_TARGET_POOL_DNE;                                                                                     │
     │2832        }                                                                                                                       │
     │2833      }                                                                                                                         │
     │2834      ldout(cct,20) << __func__ << " target " << t->target_oid << " "                                                           │
     │2835                    << t->target_oloc << " -> pgid " << pgid << dendl;                                                          │
     └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  multi-thre Thread 0x7ffff In: Objecter::_calc_target                                                       Line: 2828 PC: 0x7ffff7b42748
  (gdb) n
  (gdb) p    t->target_oid
  $21 = {
    name = "test-write"
  }
  (gdb) p    t->target_oloc
  $22 = {
    pool = 3,
    key =    "",
    nspace = "",
    hash = -1n
  }
      
  
  ```

  这里 *object_locator_to_pg* 的输入参数为 *t->target_oid*, *t->target_oloc*, *pgid* 按 *s* 进入 *object_locator_to_pg* 函数

  ```
  │2000    int OSDMap::object_locator_to_pg(                                                                                           │
  │2001      const object_t& oid, const object_locator_t& loc, pg_t &pg) const                                                         │
  │2002    {                                                                                                                           │
  │2003      if (loc.hash >= 0) {         // 从上面参数可以知道的hash的值为 -1                                                                                                    │
  │2004        if (!get_pg_pool(loc.get_pool())) {                                                                                     │
  │2005          return -ENOENT;                                                                                                       │
  │2006        }                                                                                                                       │
  │2007        pg = pg_t(loc.hash, loc.get_pool());                                                                                    │
  │2008        return 0;                                                                                                               │
  │2009      }                                                                                                                         │
  >│2010      return map_to_pg(loc.get_pool(), oid.name, loc.key, loc.nspace, &pg);                                                     │
  │2011    }            
  └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  multi-thre Thread 0x7ffff In: OSDMap::object_locator_to_pg                                                 Line: 2010 PC: 0x7fffeee46240
  }
  OSDMap::object_locator_to_pg (this=0x78b930, oid=..., loc=..., pg=...) at /home/user/ceph/src/osd/OSDMap.cc:2002
  
  ```

  这里他会再次进入 *map_to_pg* 函数， 参数为 *pool id*, *obj name*, *obj key*, *namespace*, *pg实体*

  ------

  - **1.1.1、OSDMap::map_to_pg**

  ```
  │1979    // mapping                                                                                                                  │
  │1980    int OSDMap::map_to_pg(                                                                                                      │
  │1981      int64_t poolid,                                                                                                           │
  │1982      const string& name,                                                                                                       │
  │1983      const string& key,                                                                                                        │
  │1984      const string& nspace,                                                                                                     │
  │1985      pg_t *pg) const                                                                                                           │
  │1986    {                                                                                                                           │
  │1987      // calculate ps (placement seed)                                                                                          │
  │1988      const pg_pool_t *pool = get_pg_pool(poolid);   // 根据poolid 获取pool实体                                                                           │
  │1989      if (!pool)                                                                                                                │
  │1990        return -ENOENT;                                                                                                         │
  │1991      ps_t ps;                                                                                                                  │
  │1992      if (!key.empty())                       // 判断客户端写入的时候是否指定了key                                                                                  │
  │1993        ps = pool->hash_key(key, nspace);                                                                                       │
  │1994      else                                                                                                                      │
  >│1995        ps = pool->hash_key(name, nspace);  // 没指定就用文件名当做key                                                                                    │
  │1996      *pg = pg_t(ps, poolid);                                                                                                         │
  │1997      return 0;                                                                                                                 │
  │1998    }                                                  
  └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  multi-thre Thread 0x7ffff In: OSDMap::map_to_pg                                                            Line: 1995 PC: 0x7fffeee4615d
      opts = std::map with 0 elements
  },
  application_metadata = std::map with 1 elements = {
      ["rgw"] = std::map with 0 elements
  },
  grade_table =    std::vector of length 0, capacity 0
  }
  (gdb) n
  (gdb) p    key.empty()
  $25 = true
  
  ```

  因为写入的文件并没有设置key 所以，会使用文件名，当做key 跟nspace 一起作为hash的输入 让我们看看 *pool->hash_key(name, nspace)* 会做些什么

  ------

  - **1.1.1.1、pg_pool_t::hash_key**

  ```
      │1374    uint32_t pg_pool_t::hash_key(const string& key, const string& ns) const                                                     │
     │1375    {                                                                                                                           │
     │1376     if (ns.empty())        // 判断namespace 是否为空                                                                                                    │
    >│1377        return ceph_str_hash(object_hash, key.data(), key.length());                                                            │
     │1378      int nsl = ns.length();                                                                                                    │
     │1379      int len = key.length() + nsl + 1;                                                                                         │
     │1380      char buf[len];                                                                                                            │
     │1381      memcpy(&buf[0], ns.data(), nsl);                                                                                          │
     │1382      buf[nsl] = '\037';                                                                                                        │
     │1383      memcpy(&buf[nsl+1], key.data(), key.length());                                                                            │
     │1384      return ceph_str_hash(object_hash, &buf[0], len);                                                                          │
     │1385    }                                                           
     └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  multi-thre Thread 0x7ffff In: pg_pool_t::hash_key                                                          Line: 1377 PC: 0x7fffeee88360
  }
  (gdb) n
  (gdb) p    key.empty()
  $25 = true 
  
  ```

  在这会根据是否有 *namespace* 进行不同的处理， 到这应该就清楚namespace的作用了吧， 这里 object_hash 就是hash 的算法类型,这里是2，是个默认值，继续进入函数 *ceph_str_hash(object_hash, key.data(), key.length())*

  ------

  - **1.1.1.1.1、unsigned ceph_str_hash**: string to hash

  ```
     │95      unsigned ceph_str_hash(int type, const char *s, unsigned len)                                                               │
     │96      {        // 选择不同的hash算法                                                                                                                   │
     │97              switch (type) {                                                                                                     │
     │98              case CEPH_STR_HASH_LINUX:                                                                                           │
     │99                      return ceph_str_hash_linux(s, len);                                                                         │
     │100             case CEPH_STR_HASH_RJENKINS:                                                                                        │
    >│101                     return ceph_str_hash_rjenkins(s, len);                                                                      │
     │102             default:                                                                                                            │
     │103                     return -1;                                                                                                  │
     │104             }                                                                                                                   │
     │105     }                                                                                                                           │
     └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  (gdb) s
  ceph_str_hash (type=2, s=0x72cc48 "test-write",    len=10)    at /home/user/ceph/src/common/ceph_hash.cc:97
  
  ```

  在这会调用 **ceph_str_hash_rjenkins(s, len)** 哈希算法，到此就不深入了，具体到的hash的算法实现，不是此次的目的，回到开始的入口 *OSDMap::map_to_pg*

  ```
      │1980    int OSDMap::map_to_pg(                                                                                                      │
     │1981      int64_t poolid,                                                                                                           │
     │1982      const string& name,                                                                                                       │
     │1983      const string& key,                                                                                                        │
     │1984      const string& nspace,                                                                                                     │
     │1985      pg_t *pg) const                                                                                                           │
     │1986    {                                                                                                                           │
     │1987      // calculate ps (placement seed)                                                                                          │
     │1988      const pg_pool_t *pool = get_pg_pool(poolid);                                                                              │
     │1989      if (!pool)                                                                                                                │
     │1990        return -ENOENT;                                                                                                         │
     │1991      ps_t ps;                                                                                                                  │
     │1992      if (!key.empty())                                                                                                         │
     │1993        ps = pool->hash_key(key, nspace);                                                                                       │
     │1994      else                                                                                                                      │
     │1995        ps = pool->hash_key(name, nspace);                                                                                      │
    >│1996      *pg = pg_t(ps, poolid);           // 计算得到的 ps 和 poolid 组装成 pg实体                                                                                                  │
     │1997      return 0;                                                                                                                 │
     │1998    }                                                                                                                           │
     │1999                                                                                                                                │
     └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  multi-thre Thread 0x7ffff In: OSDMap::map_to_pg                                                            Line: 1996 PC: 0x7fffeee46165
  (gdb) p    *pg
  $32 = {
    m_pool = 3,
    m_seed = 207473616,
    m_preferred =    -1,
    static calc_name_buf_size = 36 '$'
  }
  
  ```

  在这，函数会根据hash 生成的 *ps* （placement seed） 和 poolid 生成一个 *pg* 实体 内容如下

  ```
  pg = {
    m_pool = 3,           // pool id
    m_seed = 207473616,   // seed
    m_preferred =    -1,
    static calc_name_buf_size = 36 '$'
  }
  
  ```

  **到此，我们仅仅得到了，object的hash 值**

  ------

  - **2、 继续debug _calc_target 函数**

  ```
      │2845      osdmap->pg_to_up_acting_osds(pgid, &up, &up_primary,                                                                      │
    >│2846                                   &acting, &acting_primary);                                                                   │
     │2847      bool sort_bitwise = osdmap->test_flag(CEPH_OSDMAP_SORTBITWISE);                                                           │
     │2848      bool recovery_deletes = osdmap->test_flag(CEPH_OSDMAP_RECOVERY_DELETES);                                                  │
     │2849      unsigned prev_seed = ceph_stable_mod(pgid.ps(), t->pg_num, t->pg_num_mask);                                               │
     │2850      pg_t prev_pgid(prev_seed, pgid.pool());                                                                                   │
     │2851      if (any_change && PastIntervals::is_new_interval(                                                                         │
     │2852            t->acting_primary,                                                                                                  │
     │2853            acting_primary,                                                                                                     │
     └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  
  ```

  这一步会进入 *pg_to_up_acting_osds* 输入的参数 为，上一步计算出来的 *pg* 实体

  - **2.1、void pg_to_up_acting_osds**

  ```
     │1157      void pg_to_up_acting_osds(pg_t pg, vector<int> *up, int *up_primary,                                                      │
     │1158                                vector<int> *acting, int *acting_primary) const {                                               │
    >│1159        _pg_to_up_acting_osds(pg, up, up_primary, acting, acting_primary);                                                      │
     │1160      }                                                                                                                         │
     └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   
  pg_to_up_acting_osds (acting_primary=<optimized out>, acting=<optimized out>, up_primary=<optimized out>, up=<optimized out>, pg=...,
      this=<optimized out>) at /home/user/ceph/src/osd/OSDMap.h:1159
  (gdb)
  
  ```

  再进入 *_pg_to_up_acting_osds*

  ------

  - **2.1.1、OSDMap::_pg_to_up_acting_osds**

  ```
  void OSDMap::_pg_to_up_acting_osds(
    const pg_t& pg, vector<int> *up, int *up_primary,
    vector<int> *acting, int *acting_primary,
    bool raw_pg_to_pg) const
  {
    const pg_pool_t *pool = get_pg_pool(pg.pool());
    if (!pool ||
        (!raw_pg_to_pg && pg.ps() >= pool->get_pg_num())) {
      if (up)
        up->clear();
      if (up_primary)
        *up_primary = -1;
      if (acting)
        acting->clear();
      if (acting_primary)
        *acting_primary = -1;
      return;
    }
    vector<int> raw;
    vector<int> _up;
    vector<int> _acting;
    int _up_primary;
    int _acting_primary;
    ps_t pps;
    _get_temp_osds(*pool, pg, &_acting, &_acting_primary);  // 这个是判断是否有pg temp,如果有就会从使用pg_temp中的osd
    if (_acting.empty() || up || up_primary) {
      _pg_to_raw_osds(*pool, pg, &raw, &pps); // 开始计算pg 实体的hash值
      _apply_upmap(*pool, pg, &raw);
      _raw_to_up_osds(*pool, raw, &_up);
      _up_primary = _pick_primary(_up);
      _apply_primary_affinity(pps, *pool, &_up, &_up_primary);
      if (_acting.empty()) {
        _acting = _up;
        if (_acting_primary == -1) {
          _acting_primary = _up_primary;
        }
      }
  
  ```

  首先这里会根据是否有 *pg_temp* 做一次osd的选择，如果没有则进行正常的 PG 到 osd的选择流程 *_pg_to_raw_osds(*pool, pg, &raw, &pps)*

  ------

  - **2.1.1.1、OSDMap::_pg_to_raw_osds** : 获取pg 到 osd的映射

  ```
     │2049    void OSDMap::_pg_to_raw_osds(                                                                                               │
     │2050      const pg_pool_t& pool, pg_t pg,                                                                                           │
     │2051      vector<int> *osds,                                                                                                        │
     │2052      ps_t *ppps) const                                                                                                         │
     │2053    {                                                                                                                           │
     │2054      // map to osds[]                                                                                                          │
    >│2055      ps_t pps = pool.raw_pg_to_pps(pg);  // placement ps  这里就是pg+poolid 的hash计算过程                                                                     │
     │2056      unsigned size = pool.get_size();                                                                                          │
     │2057                                                                                                                                │
     │2058      // what crush rule?                                                                                                       │
     │2059      int ruleno = crush->find_rule(pool.get_crush_rule(), pool.get_type(), size);                                              │
     │2060      if (ruleno >= 0)                                                                                                          │
     │2061        crush->do_rule(ruleno, pps, *osds, size, osd_weight, pg.pool());                                                        │
     │2062                                                                                                                                │
     │2063      _remove_nonexistent_osds(pool, *osds);                                                                                    │
     │2064                                                                                                                                │
     │2065      if (ppps)                                                                                                                 │
     │2066        *ppps = pps;                                                                                                            │
     │2067    }                                                                                                                           │
     │2068                                                                                                                                │
     └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  multi-thre Thread 0x7ffff In: OSDMap::_pg_to_raw_osds                                                      Line: 2055 PC: 0x7fffeee590f8 
  (gdb) s
  OSDMap::_pg_to_raw_osds (this=this@entry=0x78b950, pool=..., pg=..., osds=osds@entry=0x7fffffffd6d0, ppps=ppps@entry=0x7fffffffd6c4)
      at /home/user/ceph/src/osd/OSDMap.cc:2055
  (gdb) p    pg
  $7 = {m_pool = 3, m_seed = 207473616, m_preferred = -1,    static calc_name_buf_size = 36 '$'}
  
  ```

  在这会进入 *pool.raw_pg_to_pps(pg)*, 还记得从上面获取的pgid吗？就是这里的输入，这里得到的结果就 *object–>pg* 的完整hash 这里称作 *pps*

  ------

  - **2.1.1.1.1、pg_pool_t::raw_pg_to_pps(pg_t pg)**

  ```
     │1406    ps_t pg_pool_t::raw_pg_to_pps(pg_t pg) const                                                                                │
     │1407    {                                                                                                                           │
     │1408      if (flags & FLAG_HASHPSPOOL) {                                                                                            │
     │1409        // Hash the pool id so that pool PGs do not overlap. 
                  //为了防止不同pool的pg混淆，所以需要加上pool id的hash。                                                                    │
     │1410        return                                                                                                                  │
     │1411          crush_hash32_2(CRUSH_HASH_RJENKINS1,                                                                                  │
    >│1412                         ceph_stable_mod(pg.ps(), pgp_num, pgp_num_mask),                                                       │
     │1413                         pg.pool());                                                                                            │
     │1414      } else {                                                                                                                  │
     │1415        // Legacy behavior; add ps and pool together.  This is not a great                                                      │
     │1416        // idea because the PGs from each pool will essentially overlap on                                                      │
     │1417        // top of each other: 0.5 == 1.4 == 2.3 == ...                                                                          │
     │1418        return                                                                                                                  │
     │1419          ceph_stable_mod(pg.ps(), pgp_num, pgp_num_mask) +                                                                     │
     │1420          pg.pool();                                                                                                            │
     │1421      }                                                                                                                         │
     │1422    }                                                                                                                           │
     │1423                                                                                                                                │
     └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  multi-thre Thread 0x7ffff In: pg_pool_t::raw_pg_to_pps                                                     Line: 1412 PC: 0x7fffeee883d5
      at /home/user/ceph/src/osd/OSDMap.cc:2055
  
  ```

  在这里会先对pgid 中的 *seed* 和 *pgp_num* , *pgp_num_mask* 进行stable mod 运算，这里是 **pgp_num** , **ceph_stable_mod 算法是PG映射和PG分裂的重要依据** ，这里不细说，后面会专门深入这一块。在这一步骤会得出 **真正的pg id** ， 在这我们就不进入这个函数的实现，直接进入 *crush_hash32_2*

  ------

  - **2.1.1.1.1.1、 __u32 crush_hash32(int type, __u32 a)** ： 根据pg id 和pool id 确定pg的唯一hash值

  ```
     │103     __u32 crush_hash32_2(int type, __u32 a, __u32 b)                                                                            │
     │104     {                                                                                                                           │
    >│105             switch (type) {                                                                                                     │
     │106             case CRUSH_HASH_RJENKINS1:                                                                                          │
     │107                     return crush_hash32_rjenkins1_2(a, b);                                                                      │
     │108             default:                                                                                                            │
     │109                     return 0;                                                                                                   │
     │110             }                                                                                                                   │
     │111     }                                                                                                                           │
     └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  multi-thre Thread 0x7ffff In: crush_hash32_2                                                               Line: 105  PC: 0x7fffef057600
  (gdb) crush_hash32_2 (type=0,       a=0, b=3) at /home/user/ceph/src/crush/hash.c:105
  ```

  这里直接调用 *crush_hash32_rjenkins1_2* hash 算法，输入为 pgid 和 pool id 到此，可以返回, 已经得到 **pps** 也就是 **hash(pgid, poolid)** 的值，回到 **_pg_to_raw_osds** 的函数。

### 三. 总结

------

到此 **OBJEC–>PG** 映射的流程分析就完成了，简单的总结下这个过程

- *hash(obj, nspace)* ： 获取object的哈希值 *ps* ，这里需要注意的点是 **namespace** 也参与了哈希
- *ceph_stable_mod(ps, pgp_num, pgp_mask)* ：这一步是获取真实pg的id，而且这个函数也是ceph PG分裂和迁移非常核心的一部分
- *hash(pg, pool_id)* ：这一步是将 pg 和poolid进行哈希得到唯一值 **pps**, 这个数值是作为选择osd的参数。

### 四. 参考文档

------

- [ceph source code](https://github.com/ceph/ceph/tree/v12.2.8)
- [CEPH CRUSH 算法源码分析 原文CEPH CRUSH algorithm source code analysis](https://blog.csdn.net/XingKong_678/article/details/51590459)

摘自：[深入理解ceph crush(3)—Object至PG映射源码分析](https://www.dovefi.com/post/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3crush3object%E8%87%B3pg%E6%98%A0%E5%B0%84%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
