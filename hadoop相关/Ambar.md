# Ambari
- 概念： 
> Apache Ambari项目旨在通过开发供应，管理和监控Apache Hadoop集群的软件来简化Hadoop管理。Ambari提供了一个直观，易用的Hadoop管理**Web UI**，该Web UI由其RESTful API支持。
- 功能：

> - 配置Hadoop集群
>     - Ambari提供了在任意数量的主机上安装Hadoop服务的分步向导。
>     - Ambari处理群集的Hadoop服务配置。
> - 管理Hadoop集群
>    - Ambari提供集中管理，以便在整个集群中启动，停止和重新配置Hadoop服务。
> - 监控Hadoop集群
>     - Ambari提供了一个用于监控Hadoop集群运行状况的仪表板。
>     - Ambari利用Ambari指标系统进行指标收集。
>     - Ambari利用Ambari警报框架进行系统警报，并会在需要您的注意时通知您（例如节点停机，剩余磁盘空间不足等）。
Ambari使应用程序开发人员和系统集成人员能够：
>  - 通过Ambari REST API轻松将Hadoop供应，管理和监视功能集成到他们自己的应用程序中。
- 支持操作系统：
**目前只支持Linux系列的64位操作系统**：
>- RHEL（Redhat Enterprise Linux）6和7
>- CentOS 6和7
>- OEL（Oracle Enterprise Linux）6和7
>- SLES（SuSE Linux Enterprise Server）11
>- Ubuntu 12和14
>- Debian 7
## Ambari在Ubuntu 17.10下的安装
1. Maven安装:
    1. apt-get install maven 
    1. 修改/etc/profile文件
       - export M2_HOME=/user/share/maven
       - 在path中添加M2_HOME路径
    1. 检验mvn -v是否正常运转 如下所示
        
      ```ubuntu
        Maven home: /usr/share/maven
    Java version: 1.8.0_171, vendor: Oracle Corporation
    Java home: /jvm/jdk1.8.0_171/jre
    Default locale: zh_CN, platform encoding: UTF-8
    OS name: "linux", version: "4.13.0-43-generic", arch: "amd64", family: "unix"
    ```
    4. 在/user/share/maven下 vim jms-1.1.pom文件
    
    1. 

