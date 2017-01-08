由于之前在[Comodo](https://ssl.comodo.com/free-ssl-certificate.php)申请的SSl证书马上就要到期，刚好看到现在很多网站已经用上了[Let's Encrypt](https://letsencrypt.org/)的证书，了解了下配置和更新都十分简单，因此决定本站也转向Let's Encrypt。简单记录一下配置过程（环境：CentOS 7 + nginx）。

## 1.开启EPEL

使用Certbot配置证书，首先要开启EPEL(Extra Packages for Enterprise Linux)。

    sudo yum install epel-release

## 2.安装Certbot

    sudo yum install certbot

## 3.生成及配置证书

### 预先工作

在证书生成过程中，Certbot会在网站根目录生成一个随机文件（`.well-known/acme-challenge/some_random.html`），Certbot的服务器会访问地址`imliyan.com/.well-known/acme-challenge/some_random.html`用于验证，但由于网站是使用 Django 开发并用 uwsgi + nginx 部署的，除静态文件外的 url 都被转到 Django 处理，因此这一步会返回 404，解决办法可以在 Django 的 urls.py 中进行配置或配置 nginx，此处选择配置 nginx。

    server {
        location /.well-known/acme-challenge {
            alias /path/to/imliyan.com/.well-known/acme-challenge;
        }
    }

### 生成证书

    certbot certonly --webroot -w /path/to/imliyan.com -d imliyan.com -d www.imliyan.com

上述命令会为`imliyan.com`和`www.imliyan.com`生成一个单独的证书。证书存储于 /etc/letsencrypt/live/imliyan.com/fullchain.pem

### nginx配置

    server {
        listen 443 ssl;
        server_name www.imliyan.com imliyan.com;
        
        ssl_certificate /etc/letsencrypt/live/imliyan.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/imliyan.com/privkey.pem;
        ssl_trusted_certificate /etc/letsencrypt/live/imliyan.com/chain.pem;
    }

### 重启 nginx 

    nginx -s reload

到此配置完成。整个过程相比 Comodo 手工生成证书签名请求（CSR）、生成证书、上传证书、配置要简单不少，关键是到期更新的话也十分简单。
