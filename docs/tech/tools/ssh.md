# SSH Notes

### Install on Debian12
```bash
sudo apt-get update
sudo apt-get install openssh-server
sudo systemctl start ssh
```

### 基本使用

**生成密钥对**
```bash
ssh-keygen -t rsa
```
使用时需要将生成的`id_rsa.pub`文件拷贝到远程服务器的`~/.ssh/authorized_keys`文件中。

**检查本机.ssh的属性**
+ 本地 .ssh 目录和 id_rsa 文件权限要求：
  + .ssh 目录的属性是 700
  + .ssh 目录下的 id_rsa 私有密钥的属性是 600
  + .ssh 目录下的 id_rsa.pub 公用密钥的属性是 644
+ 远程 .ssh 目录和 authorized_keys 文件权限要求：
  + .ssh 目录的属性是 700
  + .ssh 目录下的 authorized_keys 私有密钥的属性是 600

**修改sshd_config**
```sh
#sshd_config
PasswordAuthentication no 
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile     .ssh/authorized_keys
```

### ssh命令参数
+ `-o`: 指定配置选项
    `StrictHostKeyChecking=no`: 忽略服务器Key验证，可以让第一次访问不用确认服务器Key
    `UserKnownHostsFile=/dev/null`: 可以不让服务器的key保存在本地的known_hosts文件中，适用于不能保存和修改本地文件的情况
    `IdentityFile=/path/to/my_private_key`: 指定密钥文件
```sh
ssh -o StrictHostKeyChecking=no username@hostname
```
+ `-N -f`: 不执行远程命令，将ssh挂载在后台。适用于只需要打开端口，不用执行任何远程命令的清空
+ `-L`: 设置本地端口转发，当ssh到192.168.7.23时，将本地的25443端口转发到192.168.6.254的443端口。
```sh
ssh -L 25443:192.168.6.254:443 Squarehuang@192.168.7.23
```
+ `-R`: 设置远程端口转发。若要绑定到本地的所有接口，需要在`sshd_config`中修改为`GatewayPorts yes`
```sh
ssh -R 20080:localhost:20080 Squarehuang@192.168.7.23
```
+ `-D`: 设置动态转发端口，当本地客户端链接到2008端口，连接就会被转发到远程主机，然后被转发成为ssh服务器的动态端口。
```sh
ssh -D 2008 Squarehuang@192.168.7.23
```
+ `-o "ProxyCommand ssh -W %h:%p <jump_server>"`: 通过跳板机连接目标主机 
```sh
ssh Squarehuang@192.168.6.253 -o "ProxyCommand ssh -W %h:%p 10.10.1.200"
```
+ `-J`: ProxyJump选项，简化了通过跳板机连接目标主机的命令
```sh
ssh -J user@host1:port1,user2@host2:port2 user3@host3
```

### .ssh/config配置
**基本配置**
```sh
# .ssh/config
Host jetson
    HostName 192.168.7.23
    LocalForward 25443 192.168.6.254:443
    LocalForward 33062 db2:3306
    User Squarehuang
    Port 22
```

**ssh Multiplexing加速**：克服了TCP连接的消耗，即对于多次连接会话，只会使用一个TCP连接。OpenSSH使用现有的TCP连接来实现多个SSH会话，这种方式降低了新建TCP连接的负载。
> tips:OpenSSH 5.6以上版本支持
> 加速SSH连接，减少TCP连接的消耗配置所有主机登陆激活ssh multiplexing,压缩以及不检查服务器SSH key
```sh
#.ssh/config
Host *
  ServerAliveInterval 60
  ControlMaster auto/10
  ControlPath ~/.ssh/%h-%p-%r
  ControlPersist yes/10m
  StrictHostKeyChecking no
  Compression yes
```

**ssh 配置跳板服务器**:
+ 使用ProxyCommand:
```sh
Host z-dev
   HostName 192.168.6.253
   User Squarehuang
   ProxyCommand ssh -W %h:%p 10.10.1.200
```

+ 使用ProxyJump:
```sh
Host pixel
    HostName 192.168.10.9
    Port 8022
    User u0_a299
    LocalForward 8443 127.0.0.1:8443

Host zcloud
    HostName 192.168.7.200
    ProxyJump pixel
    LocalForward 8443 192.168.6.25:8443

Host z-dev
    HostName 192.168.6.253
    ProxyJump zcloud
    LocalForward 8443 127.0.0.1:443
```

+ 关闭服务器的ProxyJump:
```sh
# /etc/sshd_config
AllowTcpForwarding no
```

### ssh-agent
`ssh-agent`是一个长时间运行的守护进程（daemon），用途是对解密的专用密钥进行高速缓存。
```sh
eval `ssh-agent` #启动ssh-agent
ssh-add ~/my_key #不指定目录，默认加载.ssh/id_rsa
```
当在图形环境中（也就是多个用户），每个登陆会话都会启动一个新的ssh-agent副本，为了解决为每个shell窗口都运行一个代理的问题：
> 下面代码会维护一个 ~/.agent.env 文件以及指向当前运行的 ssh-agent 的环境。如果代理失败，将会自动打开一个新的终端窗口并添加密钥，所有后续的终端窗口都可以共享这个窗口
```sh
#~/.profile 添加ssh-agent启动脚本
if [ -f ~/.agent.env ]; then
  . ~/.agent.env -s > /dev/null

  if ! kill -0 $SSH_AGENT_PID > /dev/null 2>&1; then
    echo
    echo "Stale agent file found.  Spawning new agent..."
    eval `ssh-agent -s | tee ~/.agent.env`
    ssh-add
  fi
else
  echo "Starting ssh-agent..."
  eval `ssh-agent -s | tee ~/.agent.env`
  ssh-add
fi
```

### ssh实际应用

+ `ssh -R`配置反向代理，假设现在你在某个远程集群上，该集群没有外网，那么该如何通过ssh上网呢。一种解决方法是，在远程服务器本地的某个端口上设置http代理，在ssh的时候将那个端口代理到本地的代理端口，那么流量便可以通过本地的代理从而访问外网。





