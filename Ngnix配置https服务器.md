## Nginx配置https服务器

本文简单总结一下Nginx如何配置https服务器，给自己的站点加上绿锁。

Nginx配置https并不复杂，主要分为两个步骤：

- 申请CA证书
- Nginx配置https

#### 1. 申请CA证书

本文以腾讯云免费的DV SSL证书为例，申请CA证书。

- 打开腾讯云SSL证书-证书管理控制台

  ![](https://tva1.sinaimg.cn/large/00831rSTly1gcgkw4vkivj32ck0qwjzt.jpg)

- 点击申请免费证书，弹出确认证书类型对话框

  ![](https://tva1.sinaimg.cn/large/00831rSTly1gcgky32c3ej312k0oen0c.jpg)

- 选择免费版DVSSL证书，填写域名和申请邮箱

  ![](https://tva1.sinaimg.cn/large/00831rSTly1gcgl3oc017j310k0s2tbj.jpg)

- 选择自动DNS验证，点击确认申请。

  ![](https://tva1.sinaimg.cn/large/00831rSTly1gcgl32zwmkj315y0oywhj.jpg)

- 等待审核，提示需要一个自然工作日进行审核，但是其实很快，我这里大概等待了5分钟审核通过了。

  ![](https://tva1.sinaimg.cn/large/00831rSTly1gcgl80xxauj32by0n2n53.jpg)

- 下载证书和私钥，解压后

  ![](https://tva1.sinaimg.cn/large/00831rSTly1gcglfbt9jsj30pi05wmxq.jpg)

- 压缩包内包含申请域名的证书请求文件和不同web服务的证书和私钥文件，这里我们保存Nginx的证书和私钥。至此，申请CA证书完毕。

  ![](https://tva1.sinaimg.cn/large/00831rSTly1gcglho7titj30ns02o74n.jpg)

需要特别说明的一点是：

- 免费的证书安全认证级别一般比较低，不显示单位名称，不能证明网站的真实身份，仅起到加密传输信息的作用，适合个人网站或非电商网站。

- 由于此类只验证域名所有权的低端 SSL 证书已经被国外各种欺诈网站滥用，如果条件允许的话，强烈推荐部署验证单位信息并显示单位名称的 OV SSL 证书或申请最高信任级别的、显示绿色地址栏、直接在地址栏显示单位名称的 EV SSL 证书。

#### 2. Nginx配置https

- 拷贝`1_www.66951735.com_bundle.crt` 和 `2_www.66951735.com.key`拷贝到Nginx 服务器的 `/usr/local/nginx/conf` 目录（此处为默认安装目录，请根据实际情况操作）下。

- 编辑`/usr/local/nginx/conf/nginx.conf`文件

  ```
  server {
       #SSL 访问端口号为 443
       listen 443 ssl; 
       #填写绑定证书的域名
       server_name www.66951735.com; 
       #证书文件名称
       ssl_certificate 1_www.66951735.com_bundle.crt; 
       #私钥文件名称
       ssl_certificate_key 2_www.66951735.com.key; 
       ssl_session_timeout 5m;
       #请按照以下协议配置
       ssl_protocols TLSv1 TLSv1.1 TLSv1.2; 
       #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
       ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
       ssl_prefer_server_ciphers on;
       location / {
          #网站主页路径。此路径仅供参考，具体请您按照实际目录操作。
           root /var/www/www.domain.com; 
           index  index.html index.htm;
       }
   }
  ```

  

- 在 Nginx 根目录下，通过执行以下命令验证配置文件问题。

  ```
  ./sbin/nginx -t
  ```

  若存在问题，根据提示修改问题即可。

- 重启Nginx，即可使用https://www.66951735c.om进行访问

  ```
  ./sbin/nginx -s reload
  ```

  ![](https://tva1.sinaimg.cn/large/00831rSTly1gcgq4adkvtj31co0u0jv0.jpg)

#### 3. 遇到的问题

- 执行`./sbin/nginx -t`，出现如下错误：

  ![](https://tva1.sinaimg.cn/large/00831rSTly1gcgpyhb4yyj31s804cq5m.jpg)

- 很容易理解，Nginx缺少http_ssl_module，需要重新配置ssl模块，步骤如下：

  - 进入Nginx源码包目录，执行如下命令，配置ssl模块

    ```
    ./configure --prefix=/usr/local/nginx --with-http_ssl_module
    ```

    若执行的时候出现如下错误：

    ![](https://tva1.sinaimg.cn/large/00831rSTly1gcgqb75dddj31c806idiv.jpg)

    请执行如下命令（CentOS）：

    ```
    $ yum -y install openssl openssl-devel
    ```

  - 运行make命令

    ```
    make
    ```

    注意：此处不要执行make install，否则就是覆盖安装

  - 将刚刚编译好的nginx覆盖掉原来的nginx

    ```
    cp ./objs/nginx /usr/local/nginx/sbin/
    ```

  - 执行nginx -V查看ssl模块是否安装成功

    ```
    /usr/local/nginx/sbin/nginx -V
    ```

    ![](https://tva1.sinaimg.cn/large/00831rSTly1gcgqiy6jkfj31q808a42h.jpg)

#### 4. HTTP 自动跳转 HTTPS

- 编辑`/usr/local/nginx/conf/nginx.conf`文件，添加如下配置

  ```
  server {
      listen 80;
      #填写绑定证书的域名
      server_name www.66951735.com; 
      #把http的域名请求转成https
      return 301 https://$host$request_uri; 
  }
  ```

- 重启nginx，即可实现http自动跳转到https

