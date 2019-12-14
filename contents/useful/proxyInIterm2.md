### 在 iterm2 中设置代理
使用 SS 访问互联网的话，设置的系统代理是 Socks5，在 iterm 中访问 HTTPS 是无法使用的。有一个办法就是使用 Privoxy 将 socks5 代理转换成http代理。

- 安装 Privoxy
```shell
-> brew install privoxy
==> Downloading https://homebrew.bintray.com/bottles/privoxy-3.0.26.sierra.bottl
######################################################################## 100.0%
==> Pouring privoxy-3.0.26.sierra.bottle.1.tar.gz
==> Caveats
To have launchd start privoxy now and restart at login:
  brew services start privoxy
Or, if you don't want/need a background service you can just run:
  privoxy /usr/local/etc/privoxy/config
==> Summary
?  /usr/local/Cellar/privoxy/3.0.26: 52 files, 1.8MB
```

- 配置文件
```shell
listen-address 127.0.0.1:8087
forward-socks5 / 127.0.0.1:1086 .
forward 192.168.*.*/ .
forward 10.*.*.*/ .
forward 127.*.*.*/
```  
forward-socks5 后的端口是你自己代理的端口  

- 启动 provixy 服务
    - 可以使用 `brew services start privoxy`
    - 临时 `privoxy /usr/local/etc/privoxy/config`  

  

- 配置 HTTP 代理
```shell
export http_proxy=http://127.0.0.1:8087
export https_proxy=$http_proxy
```

- 设置开关
```shell
alias proxy='export http_proxy=http://127.0.0.1:8087 https_proxy=http://127.0.0.1:8087'
alias unproxy='unset http_proxy https_proxy'
```
