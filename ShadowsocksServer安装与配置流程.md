# ShadowsocksServer安装与配置流程

## ssServer安装

```
apt-get update # 更新软件源
apt-get install python3-pip python3-gevent python3-m2crypto python3-wheel python3-setuptools# 安装pip以及包
pip install shadowsocks #安装shadowsocks
apt-get install vim #方便编辑
```

## ssServer配置

### 编写配置文件

```
mkdir /etc/shadowsocks
vim /etc/shadowsocks/config.json
```

### config.json内容如下

```
{
    "server":"::",
    "server_port":8388,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"261018",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": false
}
```
> ::的意思是同时监听IPv4和IPv6，一般可填0.0.0.0

```
#多端口多密码配置如下
{
    "server":"::",
    "port_password": {
        "8388": "your_password1",
        "8389": "your_password2"
    },
    "local_address": "127.0.0.1",
    "local_port":1080,
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": false
}
```

### 开启ssServer

```
ssserver -c /etc/shadowsocks/config.json -d start #执行此行

ssserver -c /etc/shadowsocks/config.json -d stop #停止命令
ssserver -c /etc/shadowsocks/config.json -d restart 
```

#### 如果遇到以下报错：

> ```
> root@ubuntu-singapore:~# ssserver -c /etc/shadowsocks/config.json -d start
> INFO: loading config from /etc/shadowsocks/config.json
> 2021-11-11 07:21:51 INFO     loading libcrypto from libcrypto.so.1.1
> Traceback (most recent call last):
>   File "/usr/local/bin/ssserver", line 8, in <module>
>     sys.exit(main())
>   File "/usr/local/lib/python3.8/dist-packages/shadowsocks/server.py", line 34, in main
>     config = shell.get_config(False)
>   File "/usr/local/lib/python3.8/dist-packages/shadowsocks/shell.py", line 262, in get_config
>     check_config(config, is_local)
>   File "/usr/local/lib/python3.8/dist-packages/shadowsocks/shell.py", line 124, in check_config
>     encrypt.try_cipher(config['password'], config['method'])
>   File "/usr/local/lib/python3.8/dist-packages/shadowsocks/encrypt.py", line 44, in try_cipher
>     Encryptor(key, method)
>   File "/usr/local/lib/python3.8/dist-packages/shadowsocks/encrypt.py", line 82, in __init__
>     self.cipher = self.get_cipher(key, method, 1,
>   File "/usr/local/lib/python3.8/dist-packages/shadowsocks/encrypt.py", line 109, in get_cipher
>     return m[2](method, key, iv, op)
>   File "/usr/local/lib/python3.8/dist-packages/shadowsocks/crypto/openssl.py", line 76, in __init__
>     load_openssl()
>   File "/usr/local/lib/python3.8/dist-packages/shadowsocks/crypto/openssl.py", line 52, in load_openssl
>     libcrypto.EVP_CIPHER_CTX_cleanup.argtypes = (c_void_p,)
>   File "/usr/lib/python3.8/ctypes/__init__.py", line 386, in __getattr__
>     func = self.__getitem__(name)
>   File "/usr/lib/python3.8/ctypes/__init__.py", line 391, in __getitem__
>     func = self._FuncPtr((name_or_ordinal, self))
> AttributeError: /lib/x86_64-linux-gnu/libcrypto.so.1.1: undefined symbol: EVP_CIPHER_CTX_cleanup
> ```

> 编辑/usr/local/lib/python3.8/dist-packages/shadowsocks/crypto/openssl.py
>
> `EVP_CIPHER_CTX_cleanup`替换为`EVP_CIPHER_CTX_reset`即可解决问题。

## BBR加速

### 修改系统变量

```
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
```

### 保存生效，配置内核

```
sysctl   -p
```

###  查看内核是否已开启BBR

```
sysctl net.ipv4.tcp_available_congestion_control

sysctl net.ipv4.tcp_congestion_control
```

### 验证BBR是否已经启动

```
lsmod | grep bbr
```

## 设置ssServer自动启动

### 创建shadowsocks.servic文件，填入一下内容：

```
[Unit]
Description=Shadowsocks
After=network.target

[Service]
Type=forking
PIDFile=/run/shadowsocks/server.pid
PermissionsStartOnly=true
ExecStartPre=/bin/mkdir -p /run/shadowsocks
ExecStartPre=/bin/chown root:root /run/shadowsocks
ExecStart=/usr/local/bin/ssserver --pid-file /var/run/shadowsocks/server.pid -c /etc/shadowsocks/config.json -d start
Restart=on-abort
User=root
Group=root
UMask=0027

[Install]
WantedBy=multi-user.target
```

### 设置文件权限

```
chmod 755 /etc/systemd/system/shadowsocks.service
```

### 启动服务

```
systemctl enable shadowsocks
systemctl start shadowsocks
```



----------------

参考：

[被校园网限速限流的日子 | 路由代理ipv6访问的操作手册 - leizhao - 博客园 (cnblogs.com)](https://www.cnblogs.com/hizhaolei/p/11452303.html)

[ShenHongFei/ssdut-free-ipv6-network-guide: 大工软院 寝室免费高速IPv6网络 配置指南(A configuration guide for SSDUT(Software School of Dalian University of Technology) students to surf the internet freely in their dormitories.) (github.com)](https://github.com/ShenHongFei/ssdut-free-ipv6-network-guide)

[dlut-ipv6/SS科学上网使用BBR加速-校园网ipv6免流.md at master · fuujiro/dlut-ipv6 (github.com)](https://github.com/fuujiro/dlut-ipv6/blob/master/SS科学上网使用BBR加速-校园网ipv6免流.md)

[Ubuntu 18.04 LTS搭建Shadowsocks科学上网 (pyxis.me)](https://pyxis.me/ubuntu-18-04-ltsan-zhuang-shadowsocks/)

[只要三步轻松搭建自己的VPN——Ubuntu+SS （VPN傻瓜教程） - 代码天地 (codetd.com)](https://www.codetd.com/article/1418936)

