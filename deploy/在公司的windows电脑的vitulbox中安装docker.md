## 在公司的windows电脑的vitulbox中安装docker

我的环境是mint

### 安装docker

1. 根据以下设置仓库

   ```shell
      sudo apt-get update
      
      sudo apt-get install \
       apt-transport-https \
       ca-certificates \
       curl \
       gnupg-agent \
       software-properties-common
       
       curl -v -x http_proxy://web-proxy.tencent.com:8080 -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
       (上面的若不行，可以尝试：curl -v -x http_proxy://web-proxy.tencent.com:8080 -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -)
       
       apt-key fingerprint 0EBFCD88
       
       sudo add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/debian jessie stable"
   ```

2. 安装docker

   ```
   sudo apt-get update
   
   sudo apt-get install docker-ce docker-ce-cli containerd.io
   
   apt-cache madison docker-ce
   
   apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
   ```

3. 设置代理

   /etc/systemd/system/docker.service.d/http-proxy.conf中添加

   ```shell
   [Service]
   Environment=HTTP_PROXY=web-proxy.tencent.com:8080 NO_PROXY=localhost,127.0.0.1
   ```

4. 更新配置

   ```shell
   systemctl daemon-reload
   ```

5. 重启docker

   ```shell
   systemctl restart docker
   ```

6. 