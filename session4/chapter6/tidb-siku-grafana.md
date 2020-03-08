TiDB 作为一款HTAP 数据库，如何与大数据可视化生态中的开源组件进行集成，如何将基于TiDB 构建的实时大数据仓库的数据进行完美展现呢。本文主要介绍TiDB 与大数据可视化领域的Grafana、和Saiku 的集成.
# 1.Grafana 的集成
Grafana 是一个跨平台的开源的度量分析和可视化工具，由于Grafana 已经内置了MySQL 的插件，连接TiDB 可以直接使用MySQL 的客户端。因此可以直接添加TiDB 的数据源，方便高效，充分利用现有生态组件。
 ![Alt](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWRlci5zaGltby5pbS9mLzV2Q1VaWTQ0ZWlFVzFxcWoucG5n?x-oss-process=image/format,png#pic_center)
图1-1
      如图 1-1 所示为Grafana 面板的设计界面。通过该界面可以通过SQL 的方式，将数据通过Grafana 组件进行展示，用于大数据的可视化展现。
  注意：
    1.基于Grafana 的展现，一般在创建TiDB 数据表的时候，需要包含时间列
    2.针对Grafana 对应报表，控制查询范围，另外针对性能要求较高的场景，可以针对性的增加索引      
# 2.Saiku 的集成
 Saiku 提供了一个用户友好的基于Web 的分析解决方案，可让用户快速轻松地分析公司数据以及创建和共享报告。该解决方案可连接到一系列OLAP 服务器，并且可以快速，经济高效地部署，以允许用户实时浏览数据。通过TiDB +Saiku 构架可以搭建成一个BI 分析平台，使业务分析人员可以进行动态多维分析。后期基于TiDB 的列存储，相信TiDB 在OLAP 上的优势会更加突出。
       多维分析平台的搭建分为两大部分：第一部分是Saiku 集群；第二部分TiDB 集群。TiDB 作为Saiku 查询的数据源，通过Saiku 的业务SQL 直接从TiDB 中进行数据查询。
## 2.1 多维报表的Schema 定义
按照Saiku 定义Schema 的通用规则，可以直接定义跑在TiDB 上的Schema 。无需特殊处理。同时可以充分发挥在大数据下的多表关联优势，降低存储和数据维护成本，对关系型数据库常用的星星模型有比较好的支撑。定义的Schema 实例如下

![Alt](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWRlci5zaGltby5pbS9mL2ZyTHBCMFhvcm5ndWRFWlMucG5n?x-oss-process=image/format,png#pic_center)
     图1-2
如上图是一个星星模型的Schema ,维表kms_dt 通过主键tday 和主表的bizdate 进行关联构成星星模型.
   注意: 在配置schema 时,表和字段的大小写要和TiDB 中的表一致,否则定义的schema 在Saiku 中报错.
## 2.2 TiDB 的数据源定义
通过Saiku的管理控制台界面，进行数据源的定义，如图 1-3 所示，相关参数定义如下
   URL:
         TIDB的拦截地址，如果有负载均衡，可以设置负载均衡的的地址
  Schema:
        多维分析报表对应的schema
  jdbcdriver：
        由于TiDB 与MySQL 良好的集成性，可以直接使用MySQL 的驱动 jdbc Driver:com.mysql.jdbc.Drive
 账户和密码：
       TiDB 的数据库访问密码
![Alt](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWRlci5zaGltby5pbS9mL3BrcUh2aU5sVzVjcEpFYXIucG5n?x-oss-process=image/format,png#pic_center)
图1-3

## 2.3 多维报表的运行
 在Saiku 界面新建对应多维分析报表分析,在选择多维数据中找到需要定义的数据源,定义上对应的指标和维度作为默认的多维分析界面.分析人员可以根据需求指定自己的私有报表分析。定义完成的报表后通过报表管理界面直接运行，并进行交互式的分析。 运行界面实例如下：
   ![Alt](https://imgconvert.csdnimg.cn/aHR0cHM6Ly91cGxvYWRlci5zaGltby5pbS9mLzJSSVl3cG8zSHI0a3pia2EucG5n?x-oss-process=image/format,png#pic_center)

图1-4
 注意：
   1.Saiku 的运行由于默认的星星模型的关联方式是内连接，因此在数据的关联上要注意数据的准确性
  2.由于Saiku 的查询SQL 是根据用户自定义的，因此会出现查询SQL 无法直接使用现有数据库表索引或者语法不够优化等情况。可以通过修改 Saiku工程的Mondrian 代码方式，针对TiDB 数据库进行针对性的修改。同时通过分析TiDb 的日志表，针对性的对报表增加索引，提升查询速度。

 

    
   
