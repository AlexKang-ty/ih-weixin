# 微信服务
用于互联网医院多个公众平台的管理，包括access_token的管理，公众平台管理，在互联网医院的绑定管理，事件回调，支付信息维护等。
## 技术框架
    - sprintboot:
    - mybatis
    - postgresql
    - 
## 特殊依赖
    - mybatis-generator-lombok-plugin
        - 克隆依赖源码：git clone https://github.com/GuoGuiRong/mybatis-generator-lombok-plugin
        - 到目录中：mvn clean install
        - 添加配置文件：generatorConfig.xml
    ```
        <dependency>
            <groupId>com.chrm</groupId>
            <artifactId>mybatis-generator-lombok-plugin</artifactId>
            <version>${mybatis-generator-lombok-plugin-version}</version>
        </dependency>
    ```
## 首先创建数据库函数
```
CREATE SEQUENCE table_id_seq increment by 1 maxvalue 99999999 minvalue 1 start 1 cycle;

CREATE OR REPLACE FUNCTION snow_next_id(OUT result bigint) AS $$
DECLARE
our_epoch bigint := 1314226531721;
seq_id bigint;
now_millis bigint;
shard_id int := 5;
BEGIN
seq_id := nextval('table_id_seq') % 1024;
SELECT FLOOR(EXTRACT(EPOCH FROM clock_timestamp()) * 1000) INTO now_millis;
result := (now_millis - our_epoch) << 23;
result := result | (shard_id << 10);
result := result | (seq_id);
END;
$$ LANGUAGE PLPGSQL;
```
## accessToken
小程序以及公众号的accessToken都以String类型存放在redis中 K:appId V:accessToken

## 部署

### 基础环境
* jdk11
* pgsql>12.0
* redis3.2以上版本 部署版本5.0.7
* maven3.5以上版本

### 编译命令
```
    mvn clean package -DskipTests=true  -Denforcer.skip=true
```

### 创建目录和脚本
* sudo adduser quege
* mkdir -p /home/quege/ih-weixin
* mkdir -p /home/quege/bin
* sudo chmod 777 /home/quege/bin/start-ih-weixin.sh
* sudo chmod 777 /home/quege/bin/stop-ih-weixin.sh
* sudo chmod 777 /home/quege/bin/upgrade-ih-weixin.sh
* /home/quege/bin/start-ih-weixin.sh
```
#!/usr/bin/env bash
/bin/systemctl start ih-weixin
```
* /home/quege/bin/stop-ih-weixin.sh
```
#!/usr/bin/env bash
/bin/systemctl stop ih-weixin
```
* /home/quege/bin/upgrade-ih-weixin.sh
```
#!/usr/bin/env bash

VERSION=$1
APP_HOME=$2
SASD=ih-weixin-$VERSION
KEEP_OLD_CONF=ih-weixin-conf

function error_exit
{
	echo "$1" 1>&2
	exit 1
}

# stop old instance
echo "stopping old instance ..."
sudo /home/quege/bin/stop-ih-weixin.sh || error_exit "unable to stop ih weixin"

# because use same version info, need keep old config and remove old folder first
# step 1 keep the corrent config
echo "Upgrading ih weixin now ..."
cd $APP_HOME
if [ -d ${KEEP_OLD_CONF} ]; then
  rm -rf ${KEEP_OLD_CONF}/
  mkdir ${KEEP_OLD_CONF}
else
  mkdir ${KEEP_OLD_CONF}
fi
if [ -d ./current ]; then
	OLD=`/bin/readlink -f ./current`
	cp -rf current/conf ${KEEP_OLD_CONF}/
	rm -f current
fi
# step 2 remove old folders
if [ -d $OLD ]; then
	echo "deleting old directory $OLD"
	rm -rf $OLD
fi

#cp ./target/$IHAPID.tar.gz $APP_HOME
tar -zxf $SASD.tar.gz

ln -s $APP_HOME/$SASD $APP_HOME/current
# copy old conf back to current
cp -rf ${KEEP_OLD_CONF}/conf current/

echo "starting new instance ..."
sudo /home/quege/bin/start-ih-weixin.sh || error_exit "unable to start ih weixin"

# do some cleanup
rm $SASD.tar.gz
```

### 安装
* cp ih-weixin-[version].tar.gz /home/quege/ih-weixin
* tar -zxvf ih-weixin-[version].tar.gz
* ln -s /home/quege/ih-weixin/ih-weixin-[version] /home/quege/ih-weixin/current
* sudo touch /etc/default/ih-weixin
* sudo systemctl daemon-reload

### 配置
/home/quege/ih-weixin/current/conf/application.yml
```
server:
  port: 8082
  servlet:
    context-path: /
spring:
  datasource:
    username: [usr]
    password: [pwd]
    url: jdbc:postgresql://[ip]:5432/ih-weixin?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai&stringtype=unspecified
    driver-class-name: org.postgresql.Driver
  profiles:
    active: dev
  redis:
    host: [ip]
    port: 6379
    database: 5
  mvc:
    pathmatch:
      matching-strategy: ant_path_matcher
mybatis:
  mapper-locations: classpath:com/sunhealth/ihweixin/dao/*.xml
  type-aliases-package: com/sunhealth/ihweixin/model/entity
  configuration:
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
#定时任务
task1_cron: 0 */10 * * * ? # 项目启动每隔10分钟run一次
#本地文件存储地址
filePath: /home/quege/ih-weixin/current/file/qrcode
```

### 建立Service

* 在/etc/systemd/system目录下建立文件，比如说：ih-weixin.service
* 运行命令 `sudo systemctl daemon-reload`
* 运行命令 `sudo systemctl start ih-weixin`
```
[Unit]
Description=ih api Weixin
After=network.target

[Service]
EnvironmentFile=/etc/default/ih-weixin
ExecStart=/home/quege/ih-weixin/current/bin/ih-weixin --start
PIDFile=/home/quege/ih-weixin/current/run/ih-weixin.pid
LimitNOFILE=49152
KillMode=process
Restart=on-failure
User=quege
Group=quege

[Install]
WantedBy=multi-user.target
```

### nginx 配置
```
server {
    server_name gzh.taihealth.cn;
    underscores_in_headers on;
    location / {
        proxy_pass http://localhost:8082;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    listen 80; # managed by Certbot
    client_max_body_size 200m;

    listen 443 ssl;
    ssl_certificate /etc/nginx/ssh/8297620_gzh.taihealth.cn_nginx/8297620_gzh.taihealth.cn.pem;
    ssl_certificate_key /etc/nginx/ssh/8297620_gzh.taihealth.cn_nginx/8297620_gzh.taihealth.cn.key;
}
```

### 升级
我们可以通过QuickBuild (Continuous Integration)来升级系统，也可以手工来升级。基础是使用如下命令：
```bash
cp ih-weixin-${newVersion}.tar.gz to /home/quege/ih-weixin
/home/quege/bin/upgrade-ih-weixin.sh ${newVersion} /home/quege/ih-weixin
```
### 关于Too Many Open Files
由于Systemd忽略常规的设置，所以我们通过修改/etc/systemd/system/<serviceName>.service来增加：

```
LimitNOFILE=65536
```

详细的说明参见[stackexchange](https://unix.stackexchange.com/questions/345595/how-to-set-ulimits-on-service-with-systemd)

## Swagger.json
路径：target/classes/com/sunhealth/ihweixin/controller/swagger.json