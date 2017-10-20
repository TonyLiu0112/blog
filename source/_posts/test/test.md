# 端午节活动前端部署说明
> 此版本为demo演示版本，提供游戏效果体验，游戏排名、用户对接等功能，因涉及到如下问题后续再定
> 1. 对接客户id代码优化
> 2. 排名列表用户信息接口调用 


## Tomcat

* 版本 7.0+

* 安装

    解压下载的压缩包到指定目录即可，例如linux安装包tomcat7.tar.gz
    
    `tar zxvf tomcat7.tar.gz`
   
## 部署


> $tomcat_home 代表你部署的tomcat根目录

* 创建文件目录

    `mkdir -p /$tomcat_home/webapps/ROOT/act/`

* 解压demo演示项目

    `tar zxvf festival0505.tar.gz`
    
* 将演示项目复制到容器

    `mv festival0505 /$tomcat_home/webapps/ROOT/act/`

* 启动服务

    `./$tomcat_home/bin/startup.sh`
    
* 访问
   
   在手机端访问 *http://ip:port/act/festival0505/index.html*
   
   *eg*: `http://192.168.0.200:8080`