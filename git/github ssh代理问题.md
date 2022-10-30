## 环境：

window10 + wsl2



## wsl2设置代理：

```bash
curl -v ip:port # 验证代理地址可行
```

### export

export http_proxy=`ip`:`port`

export https_proxy=`ip`:`port`

### proxychains

```bash
vim /etc/proxychain4.conf
socket5 ip:port
```



## 设置git proxy

```bash
# linux
vim ~/.ssh/config
# windows 新建config到C:\Users\user_name\.ssh下
```

找到windows下git安装路径`PATH\Git`



config:

```sh
# Windows
ProxyCommand "PATH\Git\mingw64\bin\connect" -S ip:port -a none %h %p
# linux
ProxyCommand nc -v -x ip:port %h %p
```

```sh
Host github.com
  User git
  Port 22
  Hostname github.com
  # 注意修改路径为你的路径
  IdentityFile "PATH\.ssh\id_rsa"
  TCPKeepAlive yes

Host ssh.github.com
  User git
  Port 443
  Hostname ssh.github.com
  # 注意修改路径为你的路径
  IdentityFile "PATH\.ssh\id_rsa"
  TCPKeepAlive yes
```



## wsl ip的查找与固定