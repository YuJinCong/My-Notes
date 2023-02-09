

## Brew

### Brew安装

如果电脑没有Homebrew，打开终端输入以下命令：

`/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"`

### Brew安装软件目录

Homebrew安装的软件默认在`/usr/local/Cellar`路径下

### Brew安装redis

安装后的配置文件`redis.conf`存放在`/usr/local/etc`路径下

**启动redis服务：**

```bash
#方式1
brew services start redis

#方式2
redis-server /usr/local/etc/redis.conf

```

**查看redis服务进程：**

通过命令`ps axu | grep redis`查看redis是否正在运行

**redis-cli连接redis服务**

redis默认端口6379，默认auth为空，输入以下命令即可连接：`redis-cli -h 127.0.0.1 -9 6379`

**启动redis客户端，打开终端并输入命令redis-cli，该命令会连接本地的redis服务**

```bash
$redis-cli
redis 127.0.0.1:6379>
redis 127.0.0.1:6379> PING
PONG
```

**关闭redis**

```bash
#正常关闭
redis-cli shutdown

#强行终止redis
sudo pkill redis-server
```

**redis.conf配置文件详解**

redis默认前台启动，想以守护进程方式运行（后台运行），可以在**redis.conf**中将`daemonize no`修改为`yes`即可