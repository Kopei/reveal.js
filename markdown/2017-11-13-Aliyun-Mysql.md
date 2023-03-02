## RDS文档小结
默认部署主备架构且提供了容灾、备份、恢复、监控、迁移等方面的全套解决方案


### 账号模式
- rds账户分为经典模式和高权限模式，5.7只有高权限模式。
- 它们之间的区别是能不能直接通过sql创建账户。
  <img src="http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/26186/cn_zh/1510133360436/%E5%9C%A8%E4%B8%8D%E5%90%8C%E8%B4%A6%E5%8F%B7%E6%A8%A1%E5%BC%8F%E4%BD%BF%E7%94%A8%E5%AF%B9%E6%AF%94.png" alt="Drawing" width="400"/>
- **注意！！** 升级了高权限就不能回滚经典模式了！


### Rds的一些使用限制
- 不提供root或sa账户,修改部分参数限制
- 通过命令行和图形界面进行逻辑备份，控制台和api进行物理备份。还原亦然。
- 使用命令行和图形界面进行逻辑导入，mysql命令行和数据迁移服务进行数据迁移。
- 存储引擎：只支持InnoDb和TokuDB。
- 默认主备复制双节点，无需手动搭建，用户不能访问slave。5.7只有基础版本，没有主备，支持原生Json。
- 只能通过api和控制台进行重启。
- 普通账户不可以自定义授权，5.7不支持创建普通账户权限。


### 高可用服务
- 阿里云的rds高可用采用Detection, Repair, Notice等模块组成， 主要保障数据链路服务的可用性。
- Detection: 通过HA判断主备节点是否正常，能够排除网络抖动，30秒完成异常切换操作。
- Repair: 维护主备节点的复制关系和修复。
- Notice: 负责通知HA主备节点的状态变化。
  <img src="http://p0iombi30.bkt.clouddn.com/rda%20ha.png" alt="pic" width="400" />
- 多可用区可以承受机房级别的故障，多可用采用半同步复制方案，响应时间可能比单可用长。


- 高可用策略：
  - rds有两个高可用策略，RTO(recovery time objective）和RPO(recovery point objective)
  - 三种复制方式：
  1. 异步复制, 主库提交，响应应用，再向slave异步复制。
  2. 强同步， 主备复制完成后返回响应。
  3. 半同步， 正常主备强同步，当主向备复制发生问题时，主退化为异步复制。


### 备份服务
- 通过Backup, Recovery, Storage模块提供数据的离线备份，转储和恢复
- 通过Backup备节点压缩数据到oss
- 通过Recovery可以恢复备份文件到目标节点：
  - 回滚主节点
  - 修复备节点
  - 创建只读实例
- 通过Storage 复制备份文件的上传、转储，下载。



### 从本地迁移数据到云上
- 云上的数据库账号需要和本地一致
- 支持DTC, FTP, mysqldump
- 使用DTS迁移可以实现不停应用，平滑迁移。
- DTS支持结构迁移、全量迁移、增量迁移。迁移过程中有数据变更可以开启增量迁移。[DTS迁移文档](https://help.aliyun.com/document_detail/26132.html)
  - 迁移过程中不支持DDL
  - 结构迁移不支持event迁移
  - 增量迁移本地需要开启binlog，binlog_format为row。如果本地 MySQL 为5.6版本时，它的 binlog_row_image还须设置为full。
- [mysqldump doc](https://help.aliyun.com/document_detail/26133.html)


### 容灾
- RDS 通过数据传输服务（DTS）实现主实例和异地灾备实例之间的实时同步。
  主实例和灾备实例均搭建主备高可用架构，当主实例所在区域发生突发性自然灾害等状况，
  主节点（Master）和备节点（Slave）均无法连接时，可将异地灾备实例切换为主实例，
  在应用端修改数据库链接地址后，即可快速恢复应用的业务访问。
- 多收DTS的费用
- 灾备实例不支持备份设置、备份恢复、数据迁移、数据库管理、申请外网访问地址、修改连接地址功能。


### mysql和oss搭配使用
- 多类数据存储解决方案


### 数据库性能测试
- 使用测压[SysBench0.5](https://github.com/akopytov/sysbench)
- 准备数据
```
sysbench --num-threads=32 --max-time=3600 --max-requests=999999999 --test= oltp.lua --oltp-table-size=10000000 
--oltp-tables-count=64 --db-driver=mysql --mysql-table-engine=innodb 
--mysql-host= XXXX --mysql-port=3306 --mysql-user= XXXX --mysql-password= XXXX prepare
```


- 压测性能
```
sysbench --num-threads=32 --max-time=3600 --max-requests=999999999 --test= oltp.lua --oltp-table-size=10000000
--oltp-tables-count=64 --db-driver=mysql --mysql-table-engine=innodb --mysql-host= XXXX --mysql-port=3306 
--mysql-user= XXXX --mysql-password= XXXX run
```
- 清理环境
```
sysbench --num-threads=32 --max-time=3600 --max-requests=999999999 --test= oltp.lua --oltp-table-size=10000000
--oltp-tables-count=64 --db-driver=mysql --mysql-table-engine=innodb --mysql-host= XXXX --mysql-port=3306
--mysql-user= XXXX --mysql-password= XXXX cleanup
```


### DMS在线数据库管理工具



### 重启的坑！
- 由于数据库重启时，rds会自动更新小版本，如果出现不兼容后果自负！所以重启前必须先购买一个新的实例进行兼容测试。


### 数据压缩
- 5.6使用TokuDB支持存储压缩数据


### 主备切换
- 主备切换会有闪断，应用需要重连。实际体验小于1分钟完成切换， 5.7没有备。


### 数据复制方式
- 复制方式包括： 强同步，半同步，异步
- 强同步需要3台以上节点，一个事务包含同步完成大部分节点
- 半同步在同步出现异常时，退化为异步同步。
- 异步可能会引起数据不一致。


### CloudDBA监控服务
- 实时性能监控
- 会话诊断和终止
- 分析慢sql
- CloudDBA仅适用于公共云华北1、华北2、华东1、华东2、华南1地域的MySQL 5.5和MySQL 5.6版本的实例


### mysql5.6的读写分离
- 读写分离和主实例、读实例的区别，后者单独有连接地址，业务逻辑选择进行连接。读写分离是一个统一的地址，程序自动进行读写分流。
- 用户只需要购买读实例，可以免费试用读写分离
  
  <img src="http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/pic/51073/cn_zh/1505467133787/%E8%AF%BB%E5%86%99%E5%88%86%E7%A6%BB%E5%9C%B0%E5%9D%80.png" alt="Drawing" width="400"/>


### 数据备份和恢复
- 数据备份oss+日志oss总量>实例空间50%时，将收费。
- **误删数据该如何恢复？** 使用克隆实例按 _按备份集_ 或 _按时间点_ 两种方式复制出一个新的实例，进入克隆实例导出sql,再进入主实例导入sql。若数据较多可以使用DTS.
- 不建议使用覆盖性恢复


### 虚机自建mysql和RDS性能对比
- 云数据库是可能比自建数据库慢的。见[对比ECS自建数据库与RDS性能时的注意事项](https://help.aliyun.com/document_detail/55823.html)


### 数据加密
- SSL
- TDE 指定参与加密的数据库或者表。这些数据库或者表中的数据在写入到任何设备（磁盘、SSD、PCIE 卡）或者服务（OSS、OAS）前都会进行加密.


### 其它技术运维问题
- [其他问题](https://help.aliyun.com/knowledge_list_page/41698/1.html)



