---
title: Docker Registry--私有docker站
author: smona
published: true
date: '2017-03-03 12:55:31'
layout: post
---


# 前言  
  
　原来一直用的是docker hub来push和pull自己的镜像，可是国内pull/push到dockerhub速度实在不敢恭维，而且经常出现handshake timeout的问题，所以思索着能不能有国内的镜像源选择，daocloud是不错的国内选择，可提供pull的镜像也挺多的，但是pull可以，push得另外收费(200一个月)。我自己有三个私有的镜像需要push/pull的，所以思考着干脆自己搭建一个私有的docker registry，国内的服务器上/下载速度怎么都会比直接到docker hub拉取和推送镜像快，但是在搭建过程中遇到了各种各样的琐碎的问题，这里就做一个梳理.

　前提：一台有公网IP的主机,最好还有域名[我用的是腾讯云主机，域名是Godaddy买的，域名解析也在Godaddy~]


----------
　
# **Docker Registry**

　首先我们确保我们安装了docker engine，这是我们的前提，然后我们先拉取docker registry的镜像
```
sudo docker pull registry  
```
　**Tips** 这个默认是从docker hub拉取的，不是很大，可以挂着慢慢等，如果还是嫌慢的话，还有个办法来自[daocloud官网给出的配置 Docker 加速器](https://www.daocloud.io/mirror#accelerator-doc),提取出来就是下面这一行的linux命令：
```
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://0835afe2.m.daocloud.io
```
　其实就是下载一个set_mirror.sh脚本并运行它，脚本好像会在/etc/docker/目录下创建一个daemon.json的文件，至少我看到是这样的，具体其他做了什么操作可以仔细看看它给的set_mirror.sh脚本　
**[官方对set_mirror.sh说明：该脚本可以将 --registry-mirror 加入到你的 Docker 配置文件 /etc/default/docker 中(不是/etc/docker/?)]**  　
再去把registry  pull回来，可以感觉明显快很多


## **获取证书**

  　docker  push/pull  操作不知道从哪个版本开始需要https了，所以我们得给我们的主机弄一个https先.
  　我的主机是挂了域名的(用domain.com代表吧)，而从[[官网Get a certificate](https://docs.docker.com/registry/deploying/#get-a-certificate)]可以看出，docker registry如果要https，则需要两个东西，一个是.crt文件,一个是.key文件，有了这两个就可以让我们的docker registry走https，验证ssl了
  　其实这两个文件都可以自签署，详细办法后面再说，是使用openssl生成的，这种办法比较麻烦，而且docker还不一定认这种自签署的证书[Get https://domain.com:5000/v1/_ping: x509: certificate signed by unknown authority问题]
  　所以为了提高一次性成功率，我决定先折腾DV颁发的权威机构签署的证书（还有OV,EV，相关CA证书说明可以看[这篇](http://www.barretlee.com/blog/2016/04/24/detail-about-ca-and-certs/)）,因为正好腾讯云自己有提供DV SSL免费一年的证书[你要说我穷，我也没办法]，之前没搞过https和证书这些东西，借这个免费一年的DV SSL先上上手,具体怎么操作我就是按照腾讯云自己给出的文档，为我的域名提交了申请，然后添加了CNAME，然后耐心等待验证通过，验证通过以后，你就可以在上面下载到一个zip压缩包，解压后Nginx/那个文件夹里面有两个文件，**刚好就是我们所需要的，一个是.crt，一个是.key，所以所谓权威机构颁发的证书一般就是指这两个东西，准确来说是.crt文件**

  
## **配置证书**

  　我们从权威CA机构拿到crt和key以后(假设一个名字是domain.crt ,一个是domain.key,以方便后面说明)，在当前目录(下面的`pwd`就是指当前目录),比如是/root新建一个certs/目录，然后把domain.crt和domain.key丢进去，就可以准备启动docker registry了：

```
①https  
docker run -d -p 5000:5000 --restart=always --name registry \
  -v `pwd`/certs:/certs \  #或者直接-v /root/certs:/certs
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:latest
  
```


## **配置密码**

　在使用docker hub的过程中，一般在push的时候都会要你先login，登陆的好处就不用我说了吧，不然不加登陆验证的话，到时候什么人都可以往你的Registry塞镜像，你还不知道谁push来的

```
＃在刚才的放certs/同一层目录下新建一个auth/目录
mkdir auth 

＃生成docker registry用户名和密码，username就是用户名，password就是密码，而registry:latest是你用到的registry镜像
docker run --entrypoint htpasswd registry:latest -Bbn username password > auth/htpasswd
```
　然后启动容器的命令增长了(提醒一下，如果刚才不带密码启动过docekr registry，在执行下面命令时候，先停止并删除之前启动registry的容器)：　　
```
②https＋密码
docker run -d -p 5000:5000 --restart=always --name registry \
  -v `pwd`/certs:/certs \ #或者直接-v /root/certs:/certs
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -v `pwd`/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:latest
```

　直接浏览器访问 https://domain.com:5000/v2/察看有什么镜像，刚新生成的恐怕就只有一对花括号而已，所以这时候看到只有一对花括号说明部署成功
　或者直接验证一下：
```
#先登陆
docker login domain.com:5000

#tag现在本地的一个镜像,比如ubuntu
docker tag ubuntu:latest　domain.com:5000/ubuntu:haha

#push试试～
#[docker是根据前面的域名区分你要push哪里的，如果不加前面domain.com:5000/的话就是push去docker hub了]
docker push domain.com:5000/ubuntu:haha

#是不是快很多，当然如果你主机的带宽越给力，就越快
#然而我的小水管带宽只有1Mbps,所以快的非常不明显，1Mbps/8=256Kbps,即256Kb/s的下载速度，如果要1Mb/s,那带宽得8Mbps ：/
```


## **docker-compose!**

　不感觉刚才那一大串命令，第一容易打错，打错就GG了，第二，不容易复用么？
　所以我们用docker-compose,懒人必备！
```
#安装docker-compose,官网给出好多种办法，我推荐是“镜像--容器”的办法，因为还是依赖回docker本身，这样对系统的关系就不大了
#要知道一些老旧linux系统的软件版本依赖问题会搞到你想哭...－－

#其他安装docker-compose办法请自行check这个link==>https://docs.docker.com/compose/install/#install-as-a-container

#注意确保/usr/local/bin/在你的系统环境变量PATH包含有
curl -L https://github.com/docker/compose/releases/download/1.11.1/run.sh > /usr/local/bin/docker-compose

＃修改权限，不然无法直接命令行docker-compose这样来使用
chmod +x /usr/local/bin/docker-compose

#编写yaml配置文件
#[注意只能命名为docker-compose.yaml或者docker-compose.yml,其他的名字docker-compose都会装作看不见～]
#给个样例吧，至于每一项什么意思可以到官网上面去找==>https://docs.docker.com/compose/compose-file/#environment


registry:
  restart: always
  image: registry:latest
  ports:
    - 5000:5000
  environment:
    REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
    REGISTRY_HTTP_TLS_KEY: /certs/domain.key
    REGISTRY_AUTH: htpasswd
    REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
    REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
  volumes:
    - /path/data:/var/lib/registry
    - /root/certs:/certs
    - /root/auth:/auth



#启动[会自动找当前目录下的docker-compose.yaml/docker-compose.yml文件]
sudo docker-compose up -d 
```

----------


# **HTTPS化基于Flask的网站**
　　
　既然现在有domain.crt和domain.key,为何不顺便也把自己的小博客换成https？我的博客是用flask写的，因为本身不是做web开发，所以写起来也经历了一段艰难的过程，详细的写在这里[Web之个人小站](http://blog.csdn.net/qq_29245097/article/details/51440667)，我们现在就来说说flask怎么配置https吧
　google了很多博文，有直接用import OpenSSL的，有用import ssl的,最后发现其实flask本身包含的Werkzeug不知道什么版本开始就自带了openssl这玩意，所以不用另外import什么东西了，可以直接用，比如：
```
if __name__ == '__main__':
    context = ('/root/domain.crt', '/root/domain.key')
    app.run(host='0.0.0.0',port=4777,ssl_context=context)
    #5000给了刚才docker了，只能另外找个4777端口给网站了
```

# **Nginx统一管理**

　待补充


----------


# **自签署证书[不推荐]**

　其实自签署证书可以理解为之前由权威机构签署的过程变成了我们自己来签署，然而这个并不具备公信力的，而且一般的浏览器都不会认可你这个证书，浏览器一般内置的都是一些比较出名的权威CA的公钥，这种出来的https是会有个小绿锁的～所以一般都不推荐自签署证书，但是不是说不能做，做起来 ：/

## **For Docker**  
```
#服务器端

#生成私钥
  openssl genrsa -des3 -out domain.key 2048
#生成证书请求文件，用于和server.key一并提交到CA机构签署
  openssl req -new -key domain.key -out domain.csr
#自己充当CA机构，-x509参数,给自己签署一份crt～
  openssl req -new -x509 -key domain.key -out domain.crt -days 365  
#然后用新生成的domainself.crt和domainself.key替换掉我们原来的domain.crt和domain.key
```

```
#客户端[要给docker植入我们刚才新生成的自签署CA证书domain.crt]

#不然docker在你push的时候不给你push[Get https://domain.com:5000/v1/_ping: x509: certificate signed by unknown authority问题]
mkdir -p /etc/docker/certs.d/domain.com:5000/
scp　/root/certs/domainself.crt　/etc/docker/certs.d/domainself.com:5000/domain.crt

#重启docker进程服务
service docker restart
```

## **For Chrome** 

　这个时候虽然docker是通过了，但是浏览器却还是不识别，直接浏览器访问 https://domain.com:5000/v2/不会变成小绿锁.
我们可以往浏览器里面导入我们刚才domain.crt文件

　用chrome浏览器为栗子,地址栏输入chrome://settings/certificates==>授权中心(Authorities)==>导入==>把我们的domain.crt文件导入

![](http://img.blog.csdn.net/20170306222531215?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMjkyNDUwOTc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
