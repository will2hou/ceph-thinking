### 一. 前言

------

在[理解完crush map文件](https://www.dovefi.com/post/深入理解crush1理解crush_map文件/)之后，开始分析整个映射过程,首先第一步得学会手动编译ceph，学会使用ceph的开发者模式去trace ceph，对分析源码和找问题都非常有帮助。

还记得上一篇说的， **crush的计算过程是在客户端完成** ，所以需要编写客户端程序，调用librados库读写文件，然后通过debug 客户端程序的方式分析crush计算的整个过程。

### 二. 编译ceph，学会使用ceph开发者模式

------

- #### 1. 编译ceph

  **1.1、 首先clone 指定的版本的ceph到服务器，因为ceph 分支和版本很多，全部clone下来会比较大**

  ```
  # git clone --branch v12.2.8 https://github.com/ceph/ceph.git
  ```

  **1.2、 进入ceph目录，下载ceph代码依赖**

  ```
  # git submodule update --init --recursive
  ```

  **1.3、 下载构建代码的依赖**

  ```
  // ceph 自带的解决依赖的脚本
  # ./install-deps.sh    
  ```

  **1.4、 修改cmake参数**

  因为我们后面需要使用gdb debug客户端程序，客户端程序会依赖librados库，所以我们必须以debug的模式去编译ceph，否则编译器会优化掉很多参数，导致很多信息缺失

  首先需要修改一下ceph cmake的参数，打开 *do_cmake.sh* 脚本

  ```
  #!/bin/sh -x
  git submodule update --init --recursive         # 其实cmake的时候这里也会去下载子模块
  if test -e build; then
      echo 'build dir already exists; rm -rf build and re-run'
      exit 1
  fi
      
  ARGS=""
  if which ccache ; then
      echo "enabling ccache"
      ARGS="$ARGS -DWITH_CCACHE=ON"
  fi
      
  mkdir build
  cd build
  # cmake -DBOOST_J=$(nproc) $ARGS "$@" ..   # 注释掉原来的命令
  cmake -DCMAKE_C_FLAGS="-O0 -g3 -gdwarf-4" -DCMAKE_CXX_FLAGS="-O0 -g3 -gdwarf-4" -DBOOST_J=$(nproc) $ARGS "$@" ..    
  # minimal config to find plugins
  cat <<EOF > ceph.conf
  plugin dir = lib
  erasure code dir = lib
  EOF
      
  # give vstart a (hopefully) unique mon port to start with
  echo $(( RANDOM % 1000 + 40000 )) > .ceph_port
      
  echo done.
  ```

  可以看到这里修改了cmake的参数,增加了两个配置项，稍微解释一下

  > - *CMAKE_C_FLAGS=“-O0 -g3 -gdwarf-4”* ： c 语言编译配置
  > - *CMAKE_CXX_FLAGS=“-O0 -g3 -gdwarf-4”* ：c++ 编译配置
  > - *-O0* : 关闭编译器的优化，如果没有，使用GDB追踪程序时，大多数变量被优化,无法显示, 生产环境必须关掉
  > - *-g3* : 意味着会产生大量的调试信息
  > - *-gdwarf-4* : dwarf 是一种调试格式，dwarf-4 版本为4

  **1.5、 编译**

  ```
  # ./do_cmake
  # cd build
  # make -j 32     // 如果cpu够好，可以加大一点参数，编译更快
  ```

  **1.6、 在开发者模式下启动ceph**

  ```
  # make vstart  
  # MDS=0 RGW=1 ../src/vstart.sh -d -l -n --bluestore
  ```

  不得不说ceph这个开发者模式太棒了，一键创建启动集群，对debug代码很方便 这里看看 *vstart.sh* 的一些启动参数，更详细的使用就看[官方文档](http://docs.ceph.com/docs/luminous/dev/quick_guide/)吧

  ```
  usage: ../src/vstart.sh [option]...
  ex: ../src/vstart.sh -n -d --mon_num 3 --osd_num 3 --mds_num 1 --rgw_num 1
  options:
      -d, --debug
      -s, --standby_mds: Generate standby-replay MDS for each active
      -l, --localhost: use localhost instead of hostname
      -i <ip>: bind to specific ip
      -n, --new
      -N, --not-new: reuse existing cluster config (default)
      --valgrind[_{osd,mds,mon,rgw}] 'toolname args...'
      --nodaemon: use ceph-run as wrapper for mon/osd/mds
      --smallmds: limit mds cache size
      -m ip:port        specify monitor address
      -k keep old configuration files
      -x enable cephx (on by default)
      -X disable cephx
      --hitset <pool> <hit_set_type>: enable hitset tracking
      -e : create an erasure pool
      -o config         add extra config parameters to all sections
      --mon_num specify ceph monitor count
      --osd_num specify ceph osd count
      --mds_num specify ceph mds count
      --rgw_num specify ceph rgw count
      --mgr_num specify ceph mgr count
      --rgw_port specify ceph rgw http listen port
      --rgw_frontend specify the rgw frontend configuration
      --rgw_compression specify the rgw compression plugin
      -b, --bluestore use bluestore as the osd objectstore backend
      --memstore use memstore as the osd objectstore backend
      --cache <pool>: enable cache tiering on pool
      --short: short object names only; necessary for ext4 dev
      --nolockdep disable lockdep
      --multimds <count> allow multimds with maximum active coun
  ```

  查看一下启动后集群的状态, 是单个host三个osd的集群

  ```
  # [user@gz-cs-2-41 build]$ bin/ceph -s
  *** DEVELOPER MODE: setting PATH, PYTHONPATH and LD_LIBRARY_PATH ***
  2019-02-15 21:08:44.840344 7f4a2fbf1700 -1 WARNING: all dangerous and experimental features are enabled.
  2019-02-15 21:08:44.867806 7f4a2fbf1700 -1 WARNING: all dangerous and experimental features are enabled.
    cluster:
      id:     b6536de6-f479-49c8-b872-c9703af73371
      health: HEALTH_OK
      
    services:
      mon: 3 daemons, quorum a,b,c
      mgr: x(active)
      osd: 3 osds: 3 up, 3 in
      rgw: 1 daemon active
      
    data:
      pools:   4 pools, 32 pgs
      objects: 237 objects, 3.86KiB
      usage:   3.00GiB used, 27.0GiB / 30GiB avail
      pgs:     31 active+clean
               1  active+clean+scrubbing+deep
      
    io:
      client:   10.7KiB/s rd, 0B/s wr, 10op/s rd, 5op/s wr
  ```

  > ceph 所有的命令都在 *build/bin* 目录

  ------

- #### 2. 编写客户端程序，调用librados 往测试集群写入数据

  **2.1、 使用c语言调用librados 库写入数据**

  这里不解释客户端的代码，不是本文的重点，只要修改相应的配置项即可编译，更详细的librados库的API文档可以直接看[官网LIBRADOS ©](http://docs.ceph.com/docs/mimic/rados/api/librados/)

  ```c
  #include <stdio.h>
  #include <stdlib.h>
  #include <string.h>
  #include <rados/librados.h>
      
  int main (int argc, const char* argv[])
  { 
          rados_t cluster;
          char cluster_name[] = "ceph";           // 集群名字，默认为ceph
          char user_name[] = "client.admin";      // 用户名称
          char conf_file[] = "/home/user/ceph/build/ceph.conf";  // 配置文件路径
          char poolname[] = "default.rgw.meta";                   // 存储池名字
          char objname[] = "test-write";                          // 对象名字
          char obj_content[] = "Hello World,ceph!";                  // 对象内容
          uint64_t flags;
      
          /* Initialize the cluster handle with the "ceph" cluster name and the "client.admin" user */
          int err;
          err = rados_create2(&cluster, cluster_name, user_name, flags);
      
          if (err < 0) {
                  fprintf(stderr, "%s: Couldn't create the cluster handle! %s\n", argv[0], strerror(-err));
                  exit(EXIT_FAILURE);
          } else {
                  printf("\nCreated a cluster handle.\n");
          }
      
      
          /* Read a Ceph configuration file to configure the cluster handle. */
          //err = rados_conf_read_file(cluster, "/etc/ceph/ceph.conf");
          err = rados_conf_read_file(cluster, conf_file);
          if (err < 0) {
                  fprintf(stderr, "%s: cannot read config file: %s\n", argv[0], strerror(-err));
                  exit(EXIT_FAILURE);
          } else {
                  printf("\nRead the config file.\n");
          }
      
          /* Read command line arguments */
          err = rados_conf_parse_argv(cluster, argc, argv);
          if (err < 0) {
                  fprintf(stderr, "%s: cannot parse command line arguments: %s\n", argv[0], strerror(-err));
                  exit(EXIT_FAILURE);
          } else {
                  printf("\nRead the command line arguments.\n");
          }
      
          /* Connect to the cluster */
          err = rados_connect(cluster);
          if (err < 0) {
                  fprintf(stderr, "%s: cannot connect to cluster: %s\n", argv[0], strerror(-err));
                  exit(EXIT_FAILURE);
          } else {
                  printf("\nConnected to the cluster.\n");
          }
      
      /*
           * Continued from previous C example, where cluster handle and
           * connection are established. First declare an I/O Context.
           */
      
          rados_ioctx_t io; 
          err = rados_ioctx_create(cluster, poolname, &io);
          if (err < 0) {
                  fprintf(stderr, "%s: cannot open rados pool %s: %s\n", argv[0], poolname, strerror(-err));
                  rados_shutdown(cluster);
                  exit(EXIT_FAILURE);
          } else {
                  printf("\nCreated I/O context.\n");
          }
      
          /* Write data to the cluster synchronously. */
          err = rados_write(io, objname, obj_content, 16, 0);
          if (err < 0) {
                  fprintf(stderr, "%s: Cannot write object \"test-write\" to pool %s: %s\n", argv[0], poolname, strerror(-err));
                  rados_ioctx_destroy(io);
                  rados_shutdown(cluster);
                  exit(1);
          } else {
                  printf("\nWrote \"Hello World\" to object \"test-write\".\n");
          }
      
          char xattr[] = "en_US";
          err = rados_setxattr(io, "test-write", "lang", xattr, 5);
          if (err < 0) {
                  fprintf(stderr, "%s: Cannot write xattr to pool %s: %s\n", argv[0], poolname, strerror(-err));
                  rados_ioctx_destroy(io);
                  rados_shutdown(cluster);
                  exit(1);
          } else {
                  printf("\nWrote \"en_US\" to xattr \"lang\" for object \"test-write\".\n");
          }
      
  }
  
  ```

  **2.2、 编译客户端程序 rados_write.c**

  ```
  # gcc -g rados_write.c -lrados -L/home/user/ceph/build/lib -o rados_write -Wl,-rpath,/home/user/ceph/build/lib
  
  ```

  这里解释一下gcc 几个参数，首先需要理解的是c程序在 **编译时** 依赖的库和 **运行时** 依赖库是分开指定的，也就是说，编译的时候使用的库，不一定就是运行时使用的库

  - *-g* : 允许gdb调试
  - *-lrados* : -l 指定依赖库的名字为rados
  - *-L* : 指定编译时依赖库的的路径， 如果不指定将在系统目录下寻找
  - *-o* : 编译的二进制文件名
  - *-Wl* : 指定编译时参数
  - *-rpath* : 指定运行时依赖库的路径， 如果不指定将在系统目录下寻找

  因为需要debug 手动编译的librados库，所以这里路径需要指定库的路径，如何确定运行时依赖库的路径呢？使用 **ldd**

  ```
  # ldd rados_write
  linux-vdso.so.1 =>  (0x00007ffc1c852000)
  librados.so.2 => /home/user/ceph/build/lib/librados.so.2 (0x00007fdc3065b000)
  libc.so.6 => /lib64/libc.so.6 (0x00007fdc3027f000)
  libceph-common.so.0 => /home/user/ceph/build/lib/libceph-common.so.0 (0x00007fdc27595000)
  .
  .
  .
  
  ```

  **2.3、 运行客户端程序测试一下**

  ```
  # chmod 700 rados_write
      
  # ./rados_write
  
  Created a cluster handle.
      
  Read the config file.
      
  Read the command line arguments.
      
  Connected to the cluster.
      
  Created I/O context.
      
  Wrote "Hello World" to object "test-write".
      
  Wrote "en_US" to xattr "lang" for object "test-write".
  
  ```

  在集群中确认一下是否写入数据

  ```
  bin/rados ls -p default.rgw.meta
  2019-02-15 22:12:19.984812 7fe83dfaffc0 -1 WARNING: all dangerous and experimental features are enabled.
  2019-02-15 22:12:19.984971 7fe83dfaffc0 -1 WARNING: all dangerous and experimental features are enabled.
  2019-02-15 22:12:20.005066 7fe83dfaffc0 -1 WARNING: all dangerous and experimental features are enabled.
  test-write              // 在这里
  ```

  好了，到此手动编译ceph，并通过客户单程序调用librados写入文件到集群完成了，下一篇开始通过gdb debug 客户端程序，分析crush计算的流程。

### 三. 总结

------

ceph的开发者模式是测试ceph功能和调试代码非常方便的途径，因为集群默认开启了debug模式，所有的日志都会详细的输出，并且为了调试的方便，在正式环境中的多线程多队列，在这都会简化。

### 参考文档

- [ceph github](https://github.com/ceph/ceph)
- [lIBRADOS©](http://docs.ceph.com/docs/mimic/rados/api/librados/)
- [客户端代码参考](https://github.com/PinkGabriel/CEPH_related/tree/master/librados_example)
- [CEPH CRUSH 算法源码分析 原文CEPH CRUSH algorithm source code analysis](https://blog.csdn.net/XingKong_678/article/details/51590459)

  摘自：[深入理解ceph crush(2)—-手动编译ceph集群并使用librados读写文件](https://www.dovefi.com/post/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3crush2%E6%89%8B%E5%8A%A8%E7%BC%96%E8%AF%91ceph%E9%9B%86%E7%BE%A4%E5%B9%B6%E4%BD%BF%E7%94%A8librados%E8%AF%BB%E5%86%99%E6%96%87%E4%BB%B6/)
