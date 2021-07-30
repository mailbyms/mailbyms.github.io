---
title: mall4j部署说明
date: 2021-07-29 16:51:07
categories:
  - 部署
  - 攻略
tags:
  - 服务器
---

参考官方文档：https://gitee.com/gz-yami/mall4j/wikis



本文假定在服务器 192.168.1.70 上编译运行 Java 接口项目、管理后台网页项目  

在本地 PC 机运行运行微信端、uni-app 端代码



## 一、下载代码

### 主程序代码  

目录里已包含应用接口（yami-shop-api）、管理后台接口（yami-shop-admin）、管理后台前端（mall4v）、小程序（mall4uni）  代码

```
git clone https://gitee.com/gz-yami/mall4j.git
# commit id 5d536f641f1737d4ace00637780744b51bf06bb1
```



### 管理后台代码

目录里的代码，和主程序目录的 mall4v 子目录没什么区别

```
git clone https://gitee.com/gz-yami/mall4v
# commit id b6f56bf4cb13061bf838d13c05511bf5e2f5597d
```



## 二、部署数据库 MySQL

1. 使用 docker-compose 部署即可。
2. 启动前把主程序代码目录里的 `mall4j/db/yami_shop.sql` 放到 initdb 下，第一次启动时会初始化数据库

docker-compose.yml:

```yaml
version: '3'
services:
  mysql:
    restart: always
    image: mysql:5.7.16
    volumes:
      - ./mysql_data:/var/lib/mysql
      - ./my.cnf:/etc/my.cnf
      #  数据库还原目录 可将需要还原的sql文件放在这里
      - ./initdb:/docker-entrypoint-initdb.d
    environment:
      - "MYSQL_ROOT_PASSWORD=root"
      # yami 会自动获得 yami_shops 的权限
      - "MYSQL_DATABASE=yami_shops"
      - "MYSQL_USER=yami"
      - "MYSQL_PASSWORD=yamipasswd"
      - "TZ=Asia/Shanghai"
    ports:
      - 3306:3306
networks:
    default:
        external:
            name: mall4j_network
```

my.cnf

```
[mysqld]
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
lower_case_table_names=1

### optional configurations ###
# enable binlog
#log-bin=mysql-bin
#binlog-format=ROW
# neccessary for MySQL replaction and should be unique
#server_id=1
```



### 遇到的问题

Q: UniAPP 客户端注意账号时数据库报错

> \### Cause: com.mysql.cj.jdbc.exceptions.MysqlDataTruncation: Data truncation: Data too long for column 'login_password' at row 1

A： `tz_user` 表的 `login_password` 只有 50字节，改为 128 字节



## 三、部署 Redis

使用 docker-compose 部署即可。不设置密码

```yaml
# run it with project name "tiac": docker-compose -p tiac up -d
# more instructions from https://hub.docker.com/_/redis?tab=description&page=1&ordering=last_updated
# config file: /usr/local/etc/redis/redis.conf
# 使用自定义的 command 打开数据持久化，注意要映射 /data 目录（镜像默认启动命令是 redis-server），可选：--requirepass "mypassword"

version: '3'
services:
  redis:
    restart: always
    image: redis:6.0
    environment:
      - "TZ=Asia/Shanghai"
    command:  [redis-server, '--appendonly', 'yes']
    volumes:
      - ./data:/data
    ports:
      - 6379:6379
networks:
    default:
        external:
            name: mall4j_network
```



## 四、编译运行接口项目

<font color=red>注意 JDK 版本，官方要求为 JDK 8 。实测 Oracle JDK 1.8.0*221 不可以，OpenJDK 1.8.0*292 可以。</font>


### 修改管理后台接口配置

```
# vim yami-shop-admin/src/main/resources/application-prod.yml
  // 修改数据库连接信息
   datasource:
    ...（省略）
    username: yami
    password: yamipasswd
    
# vim yami-shop-admin/src/main/resources/log4j2_prod.xml
  //（可选）修改日志目录 PROJECT_PATH
```



### 修改应用接口配置

```
# vim yami-shop-api/src/main/resources/application-prod.yml
  //修改数据库连接信息
   datasource:
    ...（省略）
    username: yami
    password: yamipasswd

# vim yami-shop-api/src/main/resources/log4j2_prod.xml
  //（可选）修改日志目录 PROJECT_PATH
```



### 修改微信小程序的 appid 和 secret

```
# vim yami-shop-mp/src/main/resources/ma.properties
ma.appid=
ma.secret=
```

<font color=red>注意，即使运行 uniapp，不是调试小程序代码，也要填这个信息（查看本文 “小程序” ）</font> 



### 编译项目

```
mvn clean package -DskipTests
```



### 运行管理后台接口项目

```
java -jar -Dspring.profiles.active=prod,quartz yami-shop-admin/target/yami-shop-admin-0.0.1-SNAPSHOT.jar
# 打开项目自带的 swagger 查验
curl http://localhost:8111/doc.html
```

### 运行应用接口项目

```
java -jar -Dspring.profiles.active=prod yami-shop-api/target/yami-shop-api-0.0.1-SNAPSHOT.jar 
# 打开项目自带的 swagger 查验
curl http://localhost:8112/doc.html
```



## 五、编译管理台网站前端

进入 mall4v 项目

```
npm config set registry https://registry.npm.taobao.org
npm install 
npm run build
```

编译后的文件在 dist 子目录下，修改配置文件

```
# vim dist/config/index.js
// api接口请求地址
  window.SITE_CONFIG['baseUrl'] = 'http://192.168.1.70/apis'
```

在 nginx 部署前端页面，并配置接口地址转发到前面的 `管理后台接口` 项目：

```
server {
        listen       80;
        server_name  localhost;
		root /data/mall4j/mall4v/dist;

	location /apis {
		rewrite  ^/apis/(.*)$ /$1 break;
		proxy_pass   http://127.0.0.1:8111;
        }
}
```

管理后台的网站即 http://192.168.1.70



## 六、小程序项目

使用微信官方的开发工具 [wechat_devtools](https://dldir1.qq.com/WechatWebDev/release/p-ae42ee2cde4d42ee80ac60b35f183a99/wechat_devtools_1.05.2107090_x64.exe) 导入项目（mall4m 目录），界面会提示设置小程序的 appid，使用自动生成的 "xxx测试" 号即可

<font color=red>注意，此小程序的 appid，要填到接口项目的配置文件</font> 

```
# 修改 mall4m/utils/config.js
 domain = "http://192.168.1.70:81"; //统一接口域名，测试环境
```



## 七、uni-app 项目

使用官方的开发工具 [HBuilder X](https://download1.dcloud.net.cn/download/HBuilderX.3.1.22.20210709.full.zip) 导入项目 mall4juni  

### 修改配置文件

```
# 修改 utils/config.js
var mpAppId = '你的微信app id，与后台接口参数填的一致'
var domain = "http://192.168.1.70:8112"; //统一接口域名，测试环境
var wsDomain = "ws://192.168.1.70:8112"; //统一接口域名，测试环境
```

### 修改工具类代码

```
#修改 http.js ，原代码第33行，Authorization 的注释换掉，最终效果如下：
  wx.request({
    ...
    header: {
      // 'content-type': params.method == "GET" ? 'application/x-www-form-urlencoded' : 'application/json;charset=utf-8',
       'Authorization': params.login ? undefined : uni.getStorageSync('token')
	  // 'Authorization': !needToken ? undefined : uni.getStorageSync('token') || uni.getStorageSync('tempToken'),
    },
```

### 运行

开发工具菜单栏 -> 运行 -> 运行到内置浏览器

说明：HBuilder X 的内置浏览器，放开了跨域的限制，便于调试







