# MCSM一键安装怎么做：那条命令支持哪些系统？23333 打不开、节点连不上又该怎么救（附端口清单与手动安装回退路径）

MCSM 一键安装在 Linux 上就是这一行命令：

bash
sudo su -c "wget -qO- https://script.mcsmanager.com/setup.sh | bash"


脚本只认 Ubuntu、CentOS、Debian、Archlinux 四个发行版，跑完面板落在 `/opt/mcsmanager/` 目录，浏览器打开 `http://服务器IP:23333/` 就是登录页。面板不需要数据库，只要 Node.js 16.20.2 或更高版本。下面把装之前要确认的三件事、装完必须放行的端口、脚本失败后的手动回退、以及开服阶段最常卡住的几个点讲清楚，最后再说机器该买多大。

> 自己装 MCSM 的人，机器基本落在 4 核 8G 这一档。Java 服务端同时吃单核性能和内存，核心数堆上去、主频拉胯，TPS 照样掉。如果机器还没买，可以先看游戏云里有没有现成的 MCSM 面板服——有的话连脚本都不用跑。
>
> 👉 [看看雨云游戏云的 MCSM 面板服与云服务器配置](https://bit.ly/RaiNyun)

## 装之前先确认三件事

**系统在支持列表里。** 官方脚本只适用 Ubuntu、CentOS、Debian、Archlinux。Alma、Rocky、openSUSE 这类不在列表内的系统，脚本跑到一半报错是常事，直接走手动安装那条路更省时间。

**你有 root 权限。** 命令本身带了 `sudo su -`，但登录账号得能提权，否则第一条就卡住。

**Node.js 版本够。** 一键脚本会连运行环境一起装，你什么都不用准备。只有手动安装才需要自己先装 Node.js，版本下限是 16.20.2，官方建议直接用最新 LTS，兼容性和稳定性最好。

顺带说一句浏览器：Chrome、Firefox、Safari、Opera 都能用，IE 已经不支持了。用 IE 打开看到白屏，不是面板坏了。

## Linux 一键安装：从一行命令到 23333 面板

命令执行完之后，面板代码和运行环境都在 `/opt/mcsmanager/`。日常启停用 systemd：

bash
systemctl start mcsm-{web,daemon}   # 启动面板与守护进程
systemctl stop mcsm-{web,daemon}    # 停止


想让它开机自启，补两条 `systemctl enable mcsm-web.service` 和 `systemctl enable mcsm-daemon.service`。服务名带 `.service` 后缀，终端里敲 `mcsm` 按 Tab 能补全出来。

国内网络拉 `setup.sh` 慢或者中途断掉的情况不少，社区教程里常用的是国内镜像脚本 `setup_cn.sh`，同样要求 root 权限运行，装完还是 `mcsm-web` / `mcsm-daemon` 这套 systemd 服务。这个脚本不在官方 README 里，属于第三方提供，用之前心里有数就行。

面板起来后访问 `http://服务器IP:23333/`。如果机器没有独立公网 IP，这个地址要换成映射后的公网端口，具体见下面的端口一节。

## 脚本跑失败了，走手动安装这条回退路

官方 README 留了完整的备用方案，顺序是：装 Node.js（示例用 v20.11.0 的 linux-x64 压缩包，解压后把 `node` 和 `npm` 软链到 `/usr/bin`）→ 建 `/opt/mcsmanager/` 目录 → 下载 `mcsmanager_linux_release.tar.gz` → 解压进目录 → `chmod 775 install.sh && ./install.sh` → 开两个终端分别执行 `./start-daemon.sh` 和 `./start-web.sh`。

这里有一句官方写得很明确、但很多教程会漏掉的话：**这套手动步骤不会把面板注册成系统服务**。SSH 窗口一关，进程跟着没，表现出来就是“装好了，过一会儿又打不开了”。要常驻得用 `screen` 或 `tmux` 挂着跑；想让 systemd 托管，得另外照官方文档写 unit 文件。

## Windows、macOS、Docker 三条支线

**Windows** 是官方提供的整合版，下载 `mcsmanager_windows_release.zip`，双击 `start.bat`，Web 面板和守护进程一起起来。想在本地先把整套流程跑通、再决定要不要买服务器的人，用这个最省事。

**macOS** 先 `brew install node`，然后 `curl -L` 下载的还是那个 `mcsmanager_linux_release.tar.gz`（官方就是这么写的，不是笔误），解压、`./install.sh`、两个终端分别起 daemon 和 web，访问 `http://localhost:23333/`。Web 端一般会自动识别并连上本地守护进程。

**Docker** 用官方给的 `docker-compose.yml`，镜像为 `githubyumao/mcsmanager-web:latest` 和 `githubyumao/mcsmanager-daemon:latest`，动手前要把文件里所有 `<CHANGE_ME_TO_INSTALL_PATH>` 换成你的实际安装目录，daemon 还要挂 `/var/run/docker.sock` 并设置 `MCSM_DOCKER_WORKSPACE_PATH`，最后 `docker compose pull && docker compose up -d`。

Docker 这条路有个官方明说的坑：**装完之后 Web 端可能连不上 Daemon，进面板会看到报错**。这不是容器起失败了，需要你在面板里新建一个节点，把 Web 和 Daemon 两头接上。

如果宿主机的 Docker 跑在 rootless 模式，socket 不在 `/var/run/docker.sock`，通常在 `/run/user/<uid>/docker.sock`。挂载路径要换成这个，同时给 daemon 加上 `DOCKER_HOST=unix:///run/user/1000/docker.sock`，其中的 1000 换成 `id -u` 查到的实际 UID。

## 装完必须放行的三个端口

这三个端口管的是三件不同的事，缺一个都会出问题：

| 端口 | 作用 | 什么时候需要 |
| --- | --- | --- |
| 23333 | Web 面板 | 装完就要开 |
| 24444 | Daemon 守护进程 | 装完就要开，节点连接靠它 |
| 25565 | Minecraft 服务端默认端口 | 开服才需要 |

有独立公网 IP 的机器，防火墙和云平台安全组放行就行。没有独立公网 IP 的 NAT 型机器，得去控制台做端口映射，把内网的 23333 和 24444 各映射到一个公网端口，游戏端口同理。

**节点配置这一栏最容易填错**：远程节点 IP 填服务器的公网地址，远程节点端口填 24444 **映射出去的那个公网端口**，不是 24444 本身。填成 24444，典型症状就是面板能打开、新建实例一直转圈、节点状态显示离线。

防火墙可以直接关，也可以只放行要用的端口，后者更稳妥，关防火墙只是图快。

## 机器该买多大：MCSM 面板服还是自己的 VPS

开服圈子里比较一致的配置经验是 4 核 8G 起步，再低就很勉强。CPU 优先看单核主频，14900K、7950X 这一类高频型号比核心多但主频低的机器更适合跑 Java 服务端。系统选 Debian 12，没有就 Debian 11；Java 用 21。跑大型整合包或者同时挂几个实例，内存往上加。

雨云这边有两条路可以走，区别不小：

| 产品线 | 适合谁 | 和 MCSM 的关系 | 入口 |
| --- | --- | --- | --- |
| 游戏云 · MCSM 面板服 | 没开过服、只开一个原版或 Paper 服 | 面板和实例平台上现成，本文的安装脚本完全不用跑 | [查看 MCSM 面板服](https://bit.ly/RaiNyun) |
| 游戏云 · VPS | 要开群组服、要挂 MCDR 或机器人框架 | 按本文一键安装，面板自己管，自由度高 | [查看游戏云 VPS](https://bit.ly/RaiNyun) |
| 云服务器 | 面板之外还想跑别的服务 | 同上，系统镜像自己选 | [查看云服务器](https://bit.ly/RaiNyun) |
| 裸金属独立机 | 大型整合包、几十人同时在线的服 | 同上，配置上限最高 | [查看独立机](https://bit.ly/RaiNyun) |

雨云的产品按配置计价，同一产品线里选不同的核心数、内存、带宽和硬盘，价格都不一样，下单页会按你的选择显示实价，所以这里不给固定套餐价，照着你的目标配置点进去看更准。计费方式有动态和固定两种，长期开服建议选固定计费，机器性能全程在线，不会因为省那点钱把 TPS 拖下去。配置拿不准可以先试用。

**面板服和 VPS 怎么取舍**：面板服便宜、开箱即用，缺点是开不了群组服，也挂不了机器人框架；VPS 什么都能干，代价是本文这套安装、端口、节点配置你得自己走一遍。第一次开服、只打算开一个服，面板服就够了。

## 开服阶段最容易卡住的五个地方

**EULA 没同意。** 启动后终端刷出 `You need to agree to the EULA in order to run the server`，去文件管理里编辑 `eula.txt`，把 `false` 改成 `true` 保存，回控制台重新点开启。

**Java 版本对不上。** 1.21 的服务端要 Java 21，装完敲 `java -version` 确认一下，显示的不是 21 就去装 JDK 21 再继续。

# -Xmx4G` 的注释去掉并改成你要的数值，再把启动命令换成 `bash run.sh` 重启。1.16.5 及以下的 Forge 没这么麻烦，直接跑 jar 就行。

**`-Xmx` 超过了物理内存。** 给 JVM 的上限必须小于机器实际内存，系统本身还要留一截。4G 内存的机器写 `-Xmx7G`，服务端一定起不来。

**面板能开、节点是红的。** 按顺序查三样：daemon 有没有真的在跑（`systemctl status mcsm-daemon`）、24444 有没有放行或映射、节点端口填的是不是映射后的公网端口。用 Docker 装的，先去面板里手动建一个节点。

给个能直接用的启动命令模板，参数含义照着改：

bash
java -server -Xms128M -XX:MaxRAMPercentage=95.0 -Dfile.encoding=UTF-8 -Duser.country=CN -jar paper-1.21-50.jar


`-Xms128M` 是初始堆内存；`-XX:MaxRAMPercentage=95.0` 限制 JVM 最多用掉系统可用内存的 95%；`-Dfile.encoding=UTF-8` 防中文乱码；`-Duser.country=CN` 让部分服务端自动切中文、走国内镜像下载资源。末尾的 `paper-1.21-50.jar` 只是教程时点的示例文件名，你下的是哪个版本就换成哪个。

## 按你的情况选

只跑一个原版或 Paper 服、朋友几个人玩，4 核 8G 就是最小可用配置，再低会难受。

要上 Forge 模组、跑大型整合包，或者打算一个面板挂好几个实例，内存往 12G 以上走，同时重点看单核主频，别被核心数迷惑。

只想先验证一下这东西好不好用，Windows 整合版在本地双击 `start.bat` 就能跑，跑通了再买机器，一分钱不浪费。

下单前确认两件事：机器有没有独立公网 IP（没有就要做端口映射，节点端口填映射后的那个），以及你要开的是 Java 版还是基岩版（端口和协议不一样）。如果要开群组服或者挂机器人框架，直接选 VPS 那档，别买面板服再回头折腾。

👉 [从这里进雨云，按你的配置看实价并下单](https://bit.ly/RaiNyun)
