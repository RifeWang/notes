# systemd

参考：http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html

系统启动和服务器守护进程管理器，涉及到系统管理的方方面面。

## 核心概念 unit

unit 表示不同类型的 systemd 对象，通过配置文件进行标识和配置；文件中主要包含了系统服务、监听 socket、保存的系统快照以及其它与 init 相关的信息。

涉及配置文件目录（优先级从高到低）：
- /etc/systemd/system : 管理员创建的执行脚本
- /run/systemd/system : 系统执行过程中产生的服务脚本
- /usr/lib/systemd/system : 每个服务最主要的启动脚本设置

unit 类型（ `systemctl -t help` ) :
- `service` (重点关注) ：文件扩展名为 .service ，用于定义系统服务
- `socket` ：用于标识进程间通信用的 socket 文件
- `busname`
- `target` ：文件扩展名为 .target ，多个 unit 构成的一个组
- `snapshot` ：管理系统快照
- `device` ：用于定于内核识别的设备
- `mount` ：定义文件系统挂载点
- `automount` ：文件系统的自动挂载点
- `swap` ：用于标识 swap 设备
- `timer` ：定时器
- `path` ：用于定义文件系统中的一个文件或目录，常用于当文件系统变化时延迟激活服务
- `slice` ：进程组
- `scope` ：不是由 systemd 启动的外部进程

### service unit file

由三个部分组成 `Unit`  `Service`  `Install` :
- [Unit] : 定义与 unit 的描述信息、行为及依赖关系等
    - `Description` : 描述信息
    - `After` : 定义 unit 的启动次序，当前 unit 应该晚于哪些 unit 启动
    - `Before` : 定义 unit 的启动次序
    - `Requires` : 依赖到的其它 unit ，强依赖，被依赖的 unit 无法激活时，当前 unit 也无法激活
    - `Wants` : 依赖到的其它 unit ，弱依赖
    - `Conflicts` : 定义 unit 间的冲突关系

- [Service] : 特定类型相关的专用选项，这里是 service 类型
    - `User` : 启动服务的用户，注意权限问题。
    - `Group` : 启动服务的用户组。
    - `PIDFile` : 服务启动后的 pid 进程号输出文件，例如 `/var/run/nginx.pid`
    - `ExecStart` : 指明启动 unit 要运行的命令或脚本的绝对路径
    - `ExecStartPre` : ExecStart 前运行
    - `ExecStartPost` : ExecStart 后运行
    - `ExecStop` : 指明停止 unit 要运行的命令或脚本，例如 `/bin/kill -s TERM $MAINPID`
    - `Restart` : 设定为 always 时表示意外终止后自动重启此服务
    - `Type` : 定义影响 ExecStart 及相关参数的功能的 unit 进程启动类型
        - `simple` : 默认值，这个 daemon 主要由 ExecStart 的指令串来启动，启动后常驻于内存中。
        - `forking` : 由 ExecStart 启动的程序作为父进程，延伸出其它子进程来作为此 daemon 的主要服务，原生父进程在启动结束后就会终止。
        - `oneshot` : 与 simple 类似，不过这个程序再工作完毕后就结束了，不会常驻在内存中。
        - `dbus` : 与 simple 类似，但这个 daemon 必须要在取得一个 D-Bus 的名称后才会继续运作，因此通常也要同时设定 Busname 才行，与 socket 编程有关系。
        - `notify` : 在启动完成后会发送一个通知消息。
        - `idle` : 与 simple 类似，要执行这个 daemon 必须所有的工作都顺利执行完毕后才会执行，这类 daemon 通常是开机到最后才执行。
    - `Environment` : 设置环境变量，可以设置多次，例如 "ENV_1=abcd" 。
    - `EnvironmentFile` : 环境变量配置文件。
    - `WorkingDirectory` : 设置工作目录，程序中的相对路径就是相对于此路径。
    - `RuntimeDirectory` : 对应一个 /run 系统路径下的运行时目录。
    - `StandardOutput` : 标准输出，例如 file:/var/log/std.log
    - `StandardError` : 标准错误

- [Install] : 定义由 systemctl enable | disable 命令在实现服务启用或禁用时用到的一些选项
    - `Alias` : 别名，可使用 systemctl command Alias.service
    - `RequireBy` : 被哪些 unit 所依赖，强依赖
    - `WantedBy` : 被哪些 unit 所依赖，弱依赖，一般情况下指定这个服务在哪个 target 目标环境下运行，比如 `multi-user.target`
    - `Also` : 安装本服务的时候还要安装别的服务

创建或修改了 unit 文件要通知 systemd 重载此配置文件：`systemctl daemon-reload` 。


编写 .service 文件示例：
```
[Unit]
Description=My service

[Service]
User=root
Group=root
ExecStart=/usr/bin/clickhouse-server --config /etc/clickhouse-server/config.xml
Type=simple

[Install]
WantedBy=multi-user.target
```

### 管理服务 command

`systemctl [COMMAND] name.service` :

                        centOS 7                                   | centOS 6

- 启动：        `systemctl start name.service`                      | service name start
- 停止：        `systemctl stop name.service`                       | service name stop
- 重启：        `systemctl restart name.service`                    | service name restart
- 状态：        `systemctl status name.service`                     | service name status
- 开机启动：     `systemctl enable name.service`                     | chkconfig name on
- 禁止开机启动：  `systemctl disable name.service`                    | chkconfig name off
- 查看服务开机启动状态： `systemctl is-enabled name.service`
- 查看所有服务开机启动状态： `systemctl list-unit-files --type service`  | chkconfig --list


- 条件式重启（已启动才重启，否则不做操作）：service name condrestart | systemctl try-restart name.service
- 重载或重启服务（先加载，再启动）：systemctl reload-or-restart name.service
- 重载或条件式重启服务：systemctl reload-or-try-restart name.service
- 禁止自动和手动启动：systemctl mask name.service
- 取消禁止：systemctl unmask name.service


查看服务：`systemctl list-units --type service --all`




-----------

1. systemctl

```shell
systemctl reboot
systemctl poweroff
systemctl halt
systemctl rescue
```

2. systemd-analyze

```shell
systemd-analyze  查看启动耗时
systemd-analyze blame  查看每个服务的启动耗时
systemd-analyze critical-chain nginx.service 显示指定服务的启动流
```

3. hostnamectl

```shell
hostnamectl     显示当前主机的信息
hostnamectl set-hostname Name  设置主机名
```

4. localectl

```shell
localectl       查看本地化设置
localectl set-locale LANG=en_GB.utf8
localectl set-keymap en_GB
```

5. timedatectl

```shell
timedatectl     查看当前时区设置
timedatectl list-timezones  显示所有可用的时区

timedatectl set-timezone America/New_York
timedatectl set-time YYYY-MM-DD
timedatectl set-time HH:MM:SS
```

6. loginctl

```shell
loginctl list-sessions  列出当前session
loginctl list-users     列出当前登录用户
loginctl show-user root  显示指定用户的信息
```
