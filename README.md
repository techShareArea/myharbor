#### 导引
项目官网: https://goharbor.io/

#### 运行环境准备
> centos7.x     
> docker19.03.2       
> docker-compose1.7.1+      
> 关闭firewalld,selinux等           

#### 配置
参考:https://github.com/goharbor/harbor/blob/master/docs/installation_guide.md

##### 下载安装包
建议使用在线安装包

从[https://github.com/goharbor/harbor/releases](https://note.youdao.com/)选择合适的版本

我以1.7.0版本为例
> wget https://storage.googleapis.com/harbor-releases/release-1.7.0/harbor-online-installer-v1.7.5.tgz        

> mkdir -p /data/deploy/
> tar -zxf harbor-online-installer-v1.7.5.tgz -C /data/deploy     
> cd /data/deploy/harbor/    

##### 准备ssl证书,并把证书放置在部署的路径下
给harbor配置https访问，可参考[https://github.com/goharbor/harbor/blob/master/docs/1.10/install-config/configure-https.md](https://note.youdao.com/)生成自签名的SSL证书，如果在公网上部署可以申请真实的证书           
> openssl genrsa -out ca.key 4096   
```
openssl req -x509 -new -nodes -sha512 -days 36500 \
  -subj "/C=CN/ST=Guangdong/L=Shenzhen/O=QSH/OU=Personal/CN=QSH" \
  -key ca.key \
  -out ca.crt
```  

> mv ca.crt server.crt      
> mv ca.key server.key      
> mkdir -p /data/harbor/cert        
> cp -a server* /data/harbor/cert/      

##### 生成服务器证书
采取的内网自用域名为*.qsh.top
> openssl genrsa -out qsh.top.key 4096      
```
openssl req -sha512 -new \
  -subj "/C=CN/ST=Guangdong/L=Shenzhen/O=QSH/OU=Personal/CN=*.qsh.top" \
  -key qsh.top.key \
  -out qsh.top.csr
```

```
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=*.qsh.top
EOF
```

```
openssl x509 -req -sha512 -days 36500 \
  -extfile v3.ext \
  -CA server.crt -CAkey server.key -CAcreateserial \
  -in qsh.top.csr \
  -out qsh.top.crt
```    

##### 修改配置文件
编辑harbor.cfg文件
```
配置项                  值                              说明
hostname                goog.qsh.top                    对外发布的主机名
ui_url_protocol         https                           使用https协议接入
ssl_cert                /data/harbor/cert/server.crt    ssl证书
ssl_cert_key            /data/harbor/cert/server.key    ssl证书私钥
secretkey_path          /data/harbor                    密钥存储目录，可以按实际需要修改
harbor_admin_password   Harbor12345                     管理员密码，可以保留默认安装好再修改 
```

编辑prepare文件
> 将DATA_VOL="/data"修改为DATA_VOL="/data/harbor"       

编辑docker-compose.yml文件
> cp -a docker-compose.yml docker-compose.yml.bak       #备份
> sed -i 's/ - \/data/ - \/data\/harbor/g' docker-compose.yml       #把挂载到宿主机的目录由/data改为/data/harbor下路径      

#### 部署
> ./install.sh      

#### 配置内网访问
在coredns添加一条dns解析记录
```
cat >> /home/coredns/hosts <<EOF
10.0.5.225       good.qsh.top
EOF
```
注:coredns的搭建，配置及部署，这里不再赘述

#### 测试
内网下，使用浏览器访问:good.qsh.top，并创建demo私密项目

##### 客户端(10.0.0.247)测试流程操作

Ⅰ.配置hosts
> echo '10.0.5.225    good.qsh.top' >> /etc/hosts       

Ⅱ.配置证书
> mkdir -p /etc/docker/certs.d/good.qsh.top   

如下两行命令在服务机上执行      
> scp qsh.top.key 10.0.0.247:/etc/docker/certs.d/good.qsh.top      
> scp qsh.top.crt 10.0.0.247:/etc/docker/certs.d/good.qsh.top/qsh.top.cert       

Ⅲ.docker配置加载可信任域名
```
cat >> /etc/docker/daemon.json <<EOF
{
  "insecure-registries": [
  "good.qsh.top"
  ]
}
EOF
```

Ⅲ.登录harbor仓库
> docker login --username=admin good.qsh.top        
输入密码

Ⅳ.镜像push，run测试
> docker pull redis:4       
> docker tag redis:4 good.qsh.top/demo/redis:4      
> docker push good.qsh.top/demo/redis:4     #镜像推送至demo仓库，并方位web页面查看是否已存在        
> docker image rm good.qsh.top/demo/redis:4     #删除镜像       
> docker run --rm -it good.qsh.top/demo/redis:4     #从苍鹭下拉镜像并测试       

