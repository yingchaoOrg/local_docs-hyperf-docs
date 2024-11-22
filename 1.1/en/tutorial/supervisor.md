# Supervisor 部署应用

[Supervisor](http://www.supervisord.org/) 是 `Linux/Unix` 系统下的一个进程管理工具。可以很方便的监听、启动、停止和重启一个或多个进程。通过 [Supervisor](http://www.supervisord.org/) 管理的进程，当进程意外被 `Kill` 时，[Supervisor](http://www.supervisord.org/) 会自动将它重启，可以很方便地做到进程自动恢复的目的，而无需自己编写 `shell` 脚本来管理进程。

## Installation Supervisor

这里仅举例 `CentOS` 系统下的安装方式：

```bash
# 安装 epel 源，如果此前安装过，此步骤跳过
yum install -y epel-release
yum install -y supervisor  
```

## 创建一个配置文件

```bash
cp /etc/supervisord.conf /etc/supervisord.d/supervisord.conf
```

编辑新复制出来的配置文件 `/etc/supervisord.d/supervisord.conf`，并在文件结尾处添加以下内容后保存文件：

```ini
# 新建一个应用并设置一个名称，这里设置为 hyperf
[program:hyperf]
# 这里为您要管理的项目的启动命令，对应您的项目的真实路径
command=php /var/www/hyperf/bin/hyperf.php start
# 以哪个用户来运行该进程
user=root
# supervisor 启动时自动该应用
autostart=true
# 进程退出后自动重启进程
autorestart=true
# 重试间隔秒数
startsecs=5
# 重试次数
startretries=3
# stderr 日志输出位置
stderr_logfile=/var/www/hyperf/runtime/stderr.log
# stdout 日志输出位置
stdout_logfile=/var/www/hyperf/runtime/stdout.log
```

## 启动 Supervisor

运行下面的命令基于配置文件启动 Supervisor 程序：

```bash
supervisord -c /etc/supervisord.d/supervisord.conf
```

## 使用 supervisorctl 管理项目

```bash
# 启动 hyperf 应用
supervisorctl start hyperf
# 重启 hyperf 应用
supervisorctl restart hyperf
# 停止 hyperf 应用
supervisorctl stop hyperf  
# 查看所有被管理项目运行状态
supervisorctl status
```
