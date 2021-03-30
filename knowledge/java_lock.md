# Java lock

## heavyweight lock
   * OS mutex  
   * 用户态 内核态切换  
   * 线程阻塞  开销最大

## lightweight lock
   * 每次CAS 去设置object header word 中的 线程指针  开销小
   * 无用户态 内核态 切换 


## bias lock
   * 第一次抢锁会用CAS去设置biaslock thread ID， 但之后如果还是同一线程则不会再有CAS操作了(locking and unlocking can be performed by that thread without using atomic operations)
   * 无内核态 切换  无CAS（除了第一次）
   * bias lock 开不开 要看实际分析结果
   * -XX:+PrintGCApplicationStoppedTime
   * -XX:+PrintSafepointStatistics 
   * -XX: PrintSafepointStatisticsCount=1
   * -XX:+PrintGCApplicationStoppedTime 
   * -XX:+PrintSafepointStatistics 
   * -XX:PrintSafepointStatisticsCount=1    
   * -XX:+UseBiasedLocking 
   * -XX:BiasedLockingStartupDelay=0


## 一些相关的JAVA 源码

```
bias UT  https://github.com/openjdk/jdk/blob/9308d1858044f5b361850af40b1421e47a5da955/test/hotspot/gtest/oops/test_markWord.cpp


执行sync 方法时 jvm的操作  https://github.com/openjdk/jdk/blob/7cae6c35648c0f7ccfd7b2955e82c8aa1fcb1184/src/hotspot/share/interpreter/bytecodeInterpreter.cpp#L655-L751

markword的cas 操作  https://github.com/openjdk/jdk/blob/1fc67ab002225b1096a4d2239ab3fd115868828d/src/hotspot/share/oops/oop.inline.hpp#L75-L78

https://github.com/openjdk/jdk/blob/5e9d3fdc9ccfba6b51e340f31030905238703764/src/hotspot/share/runtime/biasedLocking.cpp

```

## 锁优化

 * 减少锁持有时间
 * 锁粒度
 * 读写分离
 * 锁消除 （编译）

