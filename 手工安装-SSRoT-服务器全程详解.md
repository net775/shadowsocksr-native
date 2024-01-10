对于需要定制化安装 SSRoT 服务器的用户, 本文提供了详细的讲解. 几乎涉及到安装的每个细节.

下面的操作全部在你的远程主机 `VPS` 上进行, 其操作系统最好是 `ubuntu` 18.04+

使用 `ssh` 软件从你电脑上登录远程主机 `VPS`. 
```
ssh root@123.45.67.89 -p 22
```
上面的 `123.45.67.89` 是你 VPS 的 IP 地址, `root` 是远程主机上的账号名, 如果你的 VPS 的 ssh 端口 不是 22, 请替换成你自己的. 然后 盲打 输入 你的 VPS 登录 密码 登入 VPS.

提升操作权限到 `root` 权限.
```
sudo -i
```
预先安装一些必要的工具
```
apt-get update -y
apt-get install make zlib1g zlib1g-dev build-essential autoconf libtool openssl libssl-dev -y
apt install python3 python python-minimal cmake git wget curl -y

```

# 安装 web 服务器 `nginx` 软件

安装 `web` 服务器 [nginx](https://nginx.org) 软件.
命令为
```
apt-get install nginx-extras -y

```
完毕以后, 可以敲入 `nginx -v` 查看 `nginx` 的版本号, 也可以通过命令 `which nginx` 查看 `nginx` 到底安装在哪个地方. 

`nginx` 配置文件是 /etc/nginx/nginx.conf, 文件里有如下两行
```
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```
第一行表明 `/etc/nginx/conf.d/` 文件夹里所有 `*.conf` 都会被包含进配置里面, 目前这个文件夹是空的.

第二行表明 `/etc/nginx/sites-enabled/` 文件夹内所有文件都会包含进配置, 目前只有一个文件 `default`, 因此该文件全路径即为 `/etc/nginx/sites-enabled/default`, 该文件内指明了 `web` 站点首页文件是 `/var/www/html/index.html`.

为了避免这个文件对我们以后的操作造成困惑、干扰. 把这个文件移走了再说
```
mv /etc/nginx/sites-enabled/default /nginx-default

```

创建我们的假站点文件夹 `/fakesite`, 并把样本主页文件复制到这里.
```
mkdir -p /fakesite/.well-known/acme-challenge/
chown -R www-data:www-data /fakesite
cp /var/www/html/*.* /fakesite

```

在 `/etc/nginx/conf.d/` 文件夹内全新创建 子 配置文件 `ssr.conf`, 使用 `cat` 命令完成
```
rm -rf /etc/nginx/conf.d/*
cat > /etc/nginx/conf.d/ssr.conf <<EOF
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name localhost;
        index index.html index.htm index.nginx-debian.html;
        root  /fakesite;
    }
EOF

```
然后使用下列命令让 nginx 重新加载配置使其生效
```
nginx -s stop
nginx

```

# 获取 数字安全证书

[Let's Encrypt](https://letsencrypt.org/) 是免费、自动化、开放的证书签发服务, 它得到了 Mozilla、Cisco、Akamai、Electronic Frontier Foundation 和 Chrome 等众多公司和机构的支持，发展十分迅猛。

申请 Let's Encrypt 证书不但免费，还非常简单，虽然每次只有 90 天的有效期，但可以通过脚本定期更新，配好之后一劳永逸。

以下命令就是配置过程. 注意, 下列命令不能简单地复制粘贴, 请将这些命令复制到文本编辑器里, 将里边的两处 `mygoodsite.com` 字串替换成你自己的 `域名`, 然后才可以复制粘贴到 `ssh` 的命令行终端控制台里.
```
org_pwd=`pwd`

mkdir /fakesite_cert
cd /fakesite_cert
openssl genrsa 4096 > account.key
openssl genrsa 4096 > private_key.pem
openssl req -new -sha256 -key private_key.pem -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:mygoodsite.com,DNS:www.mygoodsite.com")) > domain.csr
curl -L https://raw.githubusercontent.com/diafygi/acme-tiny/master/acme_tiny.py -o acme_tiny.py
python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /fakesite/.well-known/acme-challenge/ > ./signed.crt
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
cat signed.crt intermediate.pem > chained_cert.pem
wget -O - https://letsencrypt.org/certs/isrgrootx1.pem > root.pem
cat intermediate.pem root.pem > full_chained_cert.pem

cd ${org_pwd}

```

> 安装证书失败的绝大部分原因参阅 [这里](./%E5%87%86%E5%A4%87%E5%B7%A5%E4%BD%9C#%E5%B0%86-%E5%9F%9F%E5%90%8D-%E5%92%8C-%E8%99%9A%E6%8B%9F%E4%B8%BB%E6%9C%BA-%E7%9A%84-ip-%E5%85%B3%E8%81%94%E4%B8%8A)

经过这样一通**骚**操作, 我们就已经在 `/fakesite_cert` 文件夹里创建好了数字证书.

## Let's Encrypt证书申请频率的限制
- 同一个主域名一周之内只能申请50个证书
- 每个账号下每个域名每小时申请验证失败的次数为5次
- 每周只能创建5个重复的证书，即使是通过不同的账号进行创建
- 每个账号同一个IP地址每3小时最多可以创建10个证书
- 每个多域名（SAN） SSL证书（不是通配符域名证书）最多只能包含100个子域
- 更新证书没有次数的限制，但是更新证书会受到上述重复证书的限制
- 如果提示证书申请失败，可以尝试更换域名再试（添加或换不同的二级域名，也算是新域名）
- 同一IP地址，在短时间内过于频繁的申请证书，也会被限制，此时更换域名也无法申请成功，只能等待一段时间，或者更换Ip.

参考资料 [Let's Encrypt，免费好用的 HTTPS 证书](https://imququ.com/post/letsencrypt-certificate.html)

# 将 数字安全证书 部署到 web 服务器

将 原 web 服务器 配置文件 删掉, 再用 `vi` 软件重新生成 配置.
```
rm -rf /etc/nginx/conf.d/ssr.conf
vi /etc/nginx/conf.d/ssr.conf

```
通过 `vi` 输入如下内容, 注意 替换里边的 `mygoodsite.com` 字串为您的 `域名`
```
    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        ssl on;
        ssl_certificate       /fakesite_cert/chained_cert.pem;
        ssl_certificate_key   /fakesite_cert/private_key.pem;
        ssl_protocols         TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers           HIGH:!aNULL:!MD5;
        server_name           mygoodsite.com;
        index index.html index.htm index.nginx-debian.html;
        root  /fakesite;
        error_page 400 = /400.html;

        location /5mhk8LPOzXvjlAut/ {
            proxy_redirect off;
            proxy_pass http://127.0.0.1:10000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $http_host;
        }
    }

    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name mygoodsite.com;
        index index.html index.htm index.nginx-debian.html;
        root  /fakesite;

        location /.well-known/acme-challenge/ {
        }

        location / {
            rewrite ^/(.*)$ https://mygoodsite.com/$1 permanent;
        }
    }

```
注意
- 其中的三处 `mygoodsite.com` 字串, 替换成你自己的 `域名`. 
- `5mhk8LPOzXvjlAut` 字串就是 `反向代理` 的入口, 必须替换成你自己随机生成的字串, 不能偷懒, 否则 `GFW` 的网络爬虫会爬取到本网页, 将 `5mhk8LPOzXvjlAut` 加入破解词库, `GFW` 分分钟将你拿下.
- 大家可以看到, 该 `location` 节区有句 `proxy_pass http://127.0.0.1:10000;` 表明, 为 `反代` 提供服务的 `SSR` 服务器必须监听在 `127.0.0.1:10000` 上, 否则不能联动. 这也表明 **`SSR` 服务器可以安装在另一台主机上监听另一端口, 只要把这里的 `proxy_pass` 值设置相应的正确值就没问题, 这就搭建了二级代理, 极大地增加 GFW 破解的难度, 提高安全性.**
- 由于 `vi` 编辑器非常原始, 也不支持鼠标, 编辑文字极其不便, 建议在本地纯文本编辑器里弄好了以后, 一次性复制粘贴到 `vi` 编辑器里, 然后保存退出.

随机字串 的生成很简单, 如下命令足矣, 注意查看生成的字串, 如果含有斜杠 `/` 加号 `+` 或者等号 `=`, 就再次生成, 直到没有为止.
```
head -c 12 /dev/random | base64

```
最后使用下列命令使配置生效
```
nginx -s reload

```
# 证书自动更新 计划任务 设置
用 `cat` 在 `/fakesite_cert` 文件夹 创建 计划任务脚本 `renew_cert.sh`
```
cat > /fakesite_cert/renew_cert.sh <<EOF
#!/bin/bash

cd /fakesite_cert/
python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /fakesite/.well-known/acme-challenge/ > ./signed.crt || exit
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
cat signed.crt intermediate.pem > chained_cert.pem
nginx -s reload

EOF

```
然后给这个文件赋予 可执行 属性
```
chmod +x /fakesite_cert/renew_cert.sh
```
将这个脚本写入 计划任务 crontab 中
```
service cron stop
rm -rf tmp_info
crontab -l > tmp_info
echo "0 0 1 * * /fakesite_cert/renew_cert.sh >/dev/null 2>&1" >> tmp_info && crontab tmp_info && rm -rf tmp_info
service cron start

```

# 安装 SSR 服务器

使用如下命令安装 SSR 服务器.
```
curl -L https://raw.githubusercontent.com/ShadowsocksR-Live/shadowsocksr-native/master/install/ssrn-install.sh -o ssrn-install.sh
chmod +x ssrn-install.sh
./ssrn-install.sh 2>&1 | tee ssr-n.log

```
中间会要求你输入一些参数. 如果你懒, 一路回车也可以. 

安装完毕以后, 用 `vi` 打开 `SSR` 配置文件 `/etc/ssr-native/config.json`
```
vi /etc/ssr-native/config.json

```
改成下面这个样子:

找到 `server_settings` 节区, 将监听端口 `listen_port` 的值替换成 `10000` 端口.

找到 `client_settings` 节区, 将 `server_port` 的值替换成 `HTTPS` web 服务器端口 `443` 端口.

找到 `over_tls_settings` 节区, 将 `enable` 的值改成 `true`, 将 `server_domain` 的值 `mygoodsite.com` 替换成你的 `域名`, 将 `path` 的值 `5mhk8LPOzXvjlAut` 替换成你前边生成的随机字串, 注意前后的斜杠 `/` 必须保留不能删了.
```
    ...

    "server_settings": {
        "listen_address": "0.0.0.0",
        "listen_port": 10000
    },

    "client_settings": {
        "server": "12.34.56.78",
        "server_port": 443,
        "listen_address": "0.0.0.0",
        "listen_port": 1080
    },

    "over_tls_settings": {
        "enable": true,
        "server_domain": "mygoodsite.com",
        "path": "/5mhk8LPOzXvjlAut/",
        "root_cert_file": ""
    }

```
然后再用 `cat /etc/ssr-native/config.json` 命令检查一下, 如果没问题就可以复制粘贴到本地作为客户端的 配置文件了.

最后使用如下命令重启 SSR 服务.
```
systemctl restart ssr-native.service

```

到此, 如果一切顺利, `SSRoT` 安装完成. 可以使用 [`ssr-client` 客户端](./客户端用法) 翻墙了.

当然. 这时候位于 `/fakesite/` 文件夹内的网站内容, 还是一个孤零零的示例网页文件, 下一步您得尽快填补一些看起来有趣的内容, 做法参看 [创建假网站](./创建假网站) 一节. 否则, 等 `GFW` 醒过闷儿来, 就可能封杀这个网站了.

# 成果汇总
|     项目                             |     路径       |
|--------------------------------------|----------------|
| web 服务器 nginx 安装后的可执行文件路径 | /usr/sbin/nginx |
| nginx 配置文件的根文件夹路径           | /etc/nginx/ |
| nginx 针对假网站的配置文件全路径       | /etc/nginx/conf.d/ssr.conf |
| 假网站的根目录                        | /fakesite/ |
| 假网站的安全证书文件存放目录           | /fakesite_cert/ |
| ssl_certificate                     | /fakesite_cert/chained_cert.pem |
| ssl_certificate_key                 | /fakesite_cert/private_key.pem |
| SSR 可执行文件的全路径                | /usr/bin/ssr-server |
| SSR 配置文件全路径                    | /etc/ssr-native/config.json |


> 注意：对于某些开启了 IPv4 / IPv6 双栈的服务器，请求目标网站地址时可能会失败，这时你得 [关掉整个主机的 IPv6 支持](./%E7%A6%81%E7%94%A8-IPv6)。至于 **IPv6-only** 的主机，另有专文讲述，写给专业人士的，小白看不懂。
