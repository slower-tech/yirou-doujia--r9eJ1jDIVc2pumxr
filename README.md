
## 背景


前段时间为公司的产品增加了磁力链、种子下载的能力，测试时发现网上搜到的热门种子，有时好用，有时不好用，不好用主要表现在：没速度、速度慢、速度不稳定、下载一部分后没速度等，严重拖累了测试工作。为此，想到搭建一套内网测试环境，用来保证下载速度的同时，还能自己制作测试种子，方便控制种子的文件数、总大小等，从而进行各种各样的测试。


## 方案


网上参考了一些案例 (附录 2\)，发现大部分都是在公网环境搭建的，比如一些 NAS 设备，本身有公网 IP，可以在内网直接请求。然而我仅有的两台设备，都没有公网 IP，想申请公网 IP 也非常难，其中一台 Linux 测试机还要通过跳板机访问：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241025162507420-249512616.png)
主要的服务部署在 Linux 机上，这样除了我自己，测试人员也能用；下载侧就放在 Mac 上，或者移动设备，目前有一台 Android 和一台 iPhone；需要注意的是，出于安全管控，Linux 机只有 8000\~9000 范围内的端口可以建立侦听，这对后面服务侦听端口的设置有一定影响。
### tracker


主要提供种子的声明、查询 peer 的能力。这里需要搭建内网 tracker 的主要原因是：如果指定公网 tracker，那么它们只能记录公司 gateway 的 IP 作为 peer 地址，通过这个地址是无法找到并连接内网供种机的，即使下载者位于内网。


这里选取的是 opentracker，它本身是开源的，除去性能强大外，安装也比较简单，直接源码编译，这里归纳为以下脚本：



```
#! /bin/sh
if [ ! -f "libowfat.tar.gz" ]; then
    wget --no-check-certificate https://kokodayo.site/usr/uploads/Opentracker/libowfat.tar.gz
fi

if [ ! -d "libowfat" ]; then
    tar xvf libowfat.tar.gz
fi

cd libowfat
# note: add -std=gnu99 for GNUmakefile
# CFLAGS=-std=gnu99 -pipe $(WARN) $(DEFINE) $(OPT_REG)
# CFLAGS_OPT=-std=gnu99 -pipe $(WARN) $(DEFINE) $(OPT_PLUS)
make -j2
cd ..

if [ ! -f "opentracker.tar.gz" ]; then
    wget --no-check-certificate https://kokodayo.site/usr/uploads/Opentracker/opentracker.tar.gz
fi

if [ ! -d "opentracker" ]; then
    tar xvf opentracker.tar.gz
fi

cd opentracker
# note: add -D_GNU_SOURCE for Makefile
# CFLAGS+=-I$(LIBOWFAT_HEADERS) -Wall -pipe -Wextra -D_GNU_SOURCE #-ansi -pedantic
make -j2

if [ ! -f /usr/bin/opentracker ]; then
    sudo cp opentracker /usr/bin/opentracker
fi

cd ..

echo "done!"
```

在 CentOS 7\.9 和 gcc 4\.8\.5 上编译这两个库，存在两个坑，需要修改配置文件解决。


#### 问题 I：libowfat \-std\=gnu99


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241025170353423-431479357.png)


注意是加在 GNUMakefile 而不是 Makefile，后者不生效。没添加之前会报下面的编译错误：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241025170508001-1709681194.png)


#### 问题 II：opentracker \-D\_GNU\_SOURCE


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241025170730877-1761372963.png)


没添加时会报下面的编译错误：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241025170831061-689401102.png)


这两个问题，怀疑是 gcc 版本过低所致。


#### 编译



```
$ make
cc -c -o opentracker.o -I../libowfat -Wall -pipe -Wextra -D_GNU_SOURCE  -O3 -DWANT_FULLSCRAPE opentracker.c
cc -o opentracker opentracker.o trackerlogic.o scan_urlencoded_query.o ot_mutex.o ot_stats.o ot_vector.o ot_clean.o ot_udp.o ot_iovec.o ot_fullscrape.o ot_accesslist.o ot_http.o ot_livesync.o ot_rijndael.o -L../libowfat -lowfat -pthread -lpthread -lz
strip opentracker
$ ls -lh opentracker
-rwxr-xr-x 1 yunhai01 DOORGOD 88K Oct 25 17:10 opentracker
```

编译时 opentracker 无脑找 ../libowfat 目录作为输入库，只要保证两个库同目录即可。


脚本最后将生成的 opentracker 复制到 `/usr/bin` 以便全局生效。


#### 运行


将 opentracker 配置为系统服务，在 CentOS 上就是通过 systemctl，它需要一个配置文件：



```
[Unit]
Description=opentracker server

[Service]
User=yunhai01
ExecStart=/usr/bin/opentracker -p 8888 -P 8888
Restart=on-abort

[Install]
WantedBy=multi-user.target
```

将这个文件放置在：`/usr/lib/systemd/system/opentracker.service`，注意命令带的两个参数 \-p 和 \-P，分别表示侦听的 tcp \& udp 端口，需要选取 8000\~9000 之间的端口，原因这里不再赘述。


之后可以通过下面的命令启动 opentracker：



```
$ sudo systemctl enable opentracker.service
Created symlink from /etc/systemd/system/multi-user.target.wants/opentracker.service to /usr/lib/systemd/system/opentracker.service.
$ sudo systemctl start opentracker.service
$ systemctl status opentracker.service
● opentracker.service - opentracker server
   Loaded: loaded (/usr/lib/systemd/system/opentracker.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2024-10-12 15:59:31 CST; 1 weeks 6 days ago
 Main PID: 107808 (opentracker)
   CGroup: /system.slice/opentracker.service
           └─107808 /usr/bin/opentracker -p 8888 -P 8888
```

第一条 enable 命令，设置了开机自启动；第二条 start 命令启动了服务；第三条 status 命令查看命令状态。举一反三：停止是 stop，开机不启动就是 disable 啦。


除了在 Linux 的命令行查看服务状态，还可以通过 web 查看，地址是：`http://yunhai.bcc-bdbl.baidu.com:8888/stats`：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241025191031082-769604788.png)


其中 `yunhai.bcc-bdbl.baidu.com` 是 Linux 机域名，8888 是刚才在配置文件中指定的端口。


图中输出显示 tracker 中记录了 9 个种子，14 个 peer，12 个供种 peer。想要查看详细信息，可以加上 `?mode=everything` 参数。


最后需要注意的是，如果开启了防火墙，需要添加相应的出入端口以保证服务可以被网络上的其它设备访问。


### 供种侧


主要提供种子制作、上传、做种的能力，这里选取在 Linux 上应用广泛的 Transmission 软件包，它是很多 Linux 发行版的默认 BT 下载工具，对 Linux、Mac、Windows 等 PC 设备支持的比较好。也可以选取其它流行的 BT 工具，这方面没有限制。


#### 安装


使用 Transmission 前需要安装，这里直接通过包管理器安装：



```
$ sudo yum install transmission transmission-daemon transmission-cli
```

其中 daemon 是后台服务，用于供种；cli 是命令行工具，用来和 daemon 交互；工具包中的其它工具可以制作、查看种子文件。


#### 运行


daemon 的配置文件位于：`/var/lib/transmission/.config/transmission-daemon/settings.json`，在启动前需要做一些配置。



```
{
    "alt-speed-down": 50,
    "alt-speed-enabled": false,
    "alt-speed-time-begin": 540,
    "alt-speed-time-day": 127,
    "alt-speed-time-enabled": false,
    "alt-speed-time-end": 1020,
    "alt-speed-up": 50,
    "bind-address-ipv4": "0.0.0.0",
    "bind-address-ipv6": "::",
    "blocklist-enabled": false,
    "blocklist-url": "http://www.example.com/blocklist",
    "cache-size-mb": 4,
    "dht-enabled": false,
    "download-dir": "/var/lib/transmission/Downloads",
    "download-queue-enabled": true,
    "download-queue-size": 5,
    "encryption": 2,
    "idle-seeding-limit": 30,
    "idle-seeding-limit-enabled": false,
    "incomplete-dir": "/var/lib/transmission/Downloads",
    "incomplete-dir-enabled": false,
    "lpd-enabled": false,
    "message-level": 1,
    "peer-congestion-algorithm": "",
    "peer-id-ttl-hours": 6,
    "peer-limit-global": 200,
    "peer-limit-per-torrent": 50,
    "peer-port": 8082,
    "peer-port-random-high": 9000,
    "peer-port-random-low": 8000,
    "peer-port-random-on-start": false,
    "peer-socket-tos": "default",
    "pex-enabled": true,
    "port-forwarding-enabled": true,
    "preallocation": 1,
    "prefetch-enabled": true,
    "queue-stalled-enabled": true,
    "queue-stalled-minutes": 30,
    "ratio-limit": 2,
    "ratio-limit-enabled": false,
    "rename-partial-files": true,
    "rpc-authentication-required": true,
    "rpc-bind-address": "0.0.0.0",
    "rpc-enabled": true,
    "rpc-host-whitelist": "",
    "rpc-host-whitelist-enabled": false,
    "rpc-password": "{5a5bfec5cb304bb9018c9fcf985f87eec2053f00joXosjaH",
    "rpc-port": 8081,
    "rpc-url": "/transmission/",
    "rpc-username": "admin",
    "rpc-whitelist": "127.0.0.1",
    "rpc-whitelist-enabled": false,
    "scrape-paused-torrents-enabled": true,
    "script-torrent-done-enabled": false,
    "script-torrent-done-filename": "",
    "seed-queue-enabled": false,
    "seed-queue-size": 10,
    "speed-limit-down": 100,
    "speed-limit-down-enabled": false,
    "speed-limit-up": 100,
    "speed-limit-up-enabled": false,
    "start-added-torrents": true,
    "trash-original-torrent-files": false,
    "umask": 18,
    "upload-slots-per-torrent": 14,
    "utp-enabled": true
}
```

主要修改的字段如下：


* 端口范围
	+ rpc\-port：通过 rpc 访问的端口，设置为 8081
	+ peer\-port：告诉 tracker 下载侧连接的端口，设置为 8082
	+ peer\-port\-random\-low, peer\-port\-random\-high：随机 peer\-port 允许的范围，设置为 8000\~9000
* 用户账户
	+ rpc\-authentication\-required：需要帐密校验，不设置这个直接裸访问会失败
	+ rpc\-username：用户名，设置为 admin
	+ rpc\-password：密码，设置为 abc123


密码在服务重启后会加密，所以看到一串奇怪的数字字符组合不要惊讶。


如果 daemon 已在运行，所有变更均会在退出时被服务用当前配置覆盖，所以要确保服务停止后再修改配置文件。


daemon 的启动、状态查询、开机启动和 opentracker 一致，这里不再赘述。


#### 查询


服务启动后可通过 web 查看，地址是：`http://yunhai.bcc-bdbl.baidu.com:8081/transmission/web/index.html`


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241025174552745-1451944503.png)


注意 8081 端口就是配置文件中的 rpc\-port 字段。


网上有一些美化后的 WebUI (例如 TWC)，可以无缝替换官方简陋的界面：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241025192050778-924703265.png)


安装也不复杂：



```
$ wget https://github.com/ronggang/transmission-web-control/raw/master/release/install-tr-control-cn.sh
$ sudo sh install-tr-control-cn.sh
/bin/whoami

	欢迎使用 Transmission Web Control 中文安装脚本。
	官方帮助文档：https://github.com/ronggang/transmission-web-control/wiki
	安装脚本版本：1.2.5

	1. 安装最新的发布版本（release）；
	2. 安装指定版本，可用于降级；
	3. 恢复到官方UI；
	4. 重新下载安装脚本（install-tr-control-cn.sh）；
	5. 检测 Transmission 是否已启动；
	6. 指定安装目录；
	9. 安装最新代码库中的内容（master）；
	===================
	0. 退出安装；

	请输入对应的数字：1
```

具体可参考附录 4。关于 CUI 命令行的查询方式，在下一节制种中介绍。


#### 制种


基于原文件，transmission\-create 可以一键制种：



```
$ transmission-create -h
Usage: transmission-create [options] 

Options:
 -h --help                    Display this help page and exit
 -p --private                 Allow this torrent to only be used with the
                              specified tracker(s)
 -o --outfile           Save the generated .torrent to this filename
 -s --piecesize in KiB> Set how many KiB each piece should be, overriding
                              the preferred default
 -c --comment        Add a comment
 -t --tracker            Add a tracker's announce URL
 -V --version                 Show version number and exit
```

这里主要使用 \-t 参数：



```
$ ls -lh *.mp4
-rwxr--r-- 1 yunhai01 DOORGOD 1.2G Oct 25 19:59 盲道.mp4
$ transmission-create 盲道.mp4 -t http://yunhai.bcc-bdbl.baidu.com:8888/annouce
Creating torrent "/ext/torrent/movie/盲道.mp4.torrent" ........ done!
$ ls -lh *.torrent
-rw------- 1 yunhai01 DOORGOD 24K Oct 25 20:01 盲道.mp4.torrent
```

\-t 指定 tracker 地址，这里需要输入上面搭建的内网 tracker 地址：`http://yunhai.bcc-bdbl.baidu.com:8888/annouce`，否则在加载种子的时候，Transmission 不知道向哪个 tracker 声明 (announce) 种子，允许指定多个 \-t 参来添加多个 tracker 地址；\-o 可以指定输出的种子文件名，默认文件命名规则为"输入文件.torrent"；\-s 可以指定分片 (piece) 大小，如果不指定，transmission\-create 将根据文件长度智能设定。


分片是数据校验的基础，针对每个分片，种子文件中都会存储它的 md5 以及其它的一些必要信息，方便下载完成后进行对比。如果分片太小，分片数量就会非常多，导致种子文件膨胀。因此大文件的分片通常会大一些，保证整个种子文件大小控制在几百 KB 的尺寸。


transmission\-edit 可以修改种子的 tracker 信息：



```
$ transmission-edit -h
Usage: transmission-edit [options] torrent-file(s)

Options:
 -h --help                Display this help page and exit
 -a --add            Add a tracker's announce URL
 -d --delete         Delete a tracker's announce URL
 -r --replace   Search and replace a substring in the announce URLs
 -V --version             Show version number and exit
```

方便添加遗漏的 tracker 信息，还能删改。


transmission\-show 可以查看种子的详情：



```
$ transmission-show -h
Usage: transmission-show [options] <.torrent file>

Options:
 -h --help     Display this help page and exit
 -m --magnet   Give a magnet link for the specified torrent
 -s --scrape   Ask the torrent's trackers how many peers are in the torrent's
               swarm
 -V --version  Show version number and exit
```

不带参数时显示种子信息；\-m 将种子转换为磁力链接；\-s 查询种子所在的 tracker，获取 peer 信息：



```
$ transmission-show 盲道.mp4.torrent
Name: 盲道.mp4
File: 盲道.mp4.torrent

GENERAL

  Name: 盲道.mp4
  Hash: 7659ead46555d97a98e25a373d8216edc926d78a
  Created by: Transmission/2.94 (d8e60ee44f)
  Created on: Fri Oct 25 20:01:31 2024
  Piece Count: 1200
  Piece Size: 1.00 MiB
  Total Size: 1.26 GB
  Privacy: Public torrent

TRACKERS

  Tier #1
  http://yunhai.bcc-bdbl.baidu.com:8888/annouce

FILES

  盲道.mp4 (1.26 GB)
```

种子信息分三大块：


* 通用
	+ 文件名
	+ 唯一 hash 值
	+ 分片长度
	+ 分片个数
	+ 文件长度
	+ 各种时间
* tracker：列表
* 文件：列表



```
$ transmission-show 盲道.mp4.torrent -m
magnet:?xt=urn:btih:7659ead46555d97a98e25a373d8216edc926d78a&dn=%E7%9B%B2%E9%81%93.mp4&tr=http%3A%2F%2Fyunhai.bcc-bdbl.baidu.com%3A8888%2Fannouce
$ transmission-show movie/月光宝盒.mkv.torrent -s
Name: 月光宝盒.mkv
File: movie/月光宝盒.mkv.torrent

http://1337.abcvg.info:80/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... error: unexpected response 502 "Bad Gateway"
http://bvarf.tracker.sh:2086/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... no match
http://ipv6.rer.lol:6969/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... error: Couldn't connect to server
http://jvavav.com:80/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... error: Couldn't resolve host name
http://retracker.x2k.ru:80/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... no match
http://yunhai.bcc-bdbl.baidu.com:8888/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... 1 seeders, 1 leechers
```

\-m 转为磁力链方便在设备间传递；\-s 查询 peer 信息，新制作的种子因为还未声明到 tracker，是查不到 peer 信息的，这里使用另外一个添加好的种子做个展示，它有 6 个 tracker，返回的信息中：有的解析不了、有的连不上、有的返回了错误、有的不包含这个种子、还有的正确返回了 peer 和 seeder 的数量。


如果所有原文件都位于一个目录，例如 movie，那么可以使用脚本一键生成所有种子：



```
for file in $(ls *.mkv); do
	echo "create $file.torrent"
	param="-t http://yunhai.bcc-bdbl.baidu.com:8888/announce"
	transmission-create ${param} $PWD/$file
done
```

#### 添加


有两种方式可以将种子添加到 Transmission：WebUI 与 CUI 命令行，先说比较简单的 WebUI 方式：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241025200822170-2119887187.png)


这种方式需要事先将种子传递到启动 WebUI 的设备，并选取种子。注意这里的保存目录，是指启动 Transmission 服务的设备存放源文件的地方，如果设置不对，Transmission 无法进行数据检验，就会认为是一个下载任务而非做种任务。


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241025201509333-651691736.png)


刚添加的种子会进行检验，这个过程结束后就可以正常做种了。


另外一种方式是通过 transmission\-remote，它除了添加种子，还可以列出、查看、删改种子，这个命令功能强大选项多，这里就不一一罗列了，只讲一下日常使用的几个子命令：



```
$ transmission-remote 8081 -n admin:abc123 -si
VERSION
  Daemon version: 2.94 (d8e60ee44f)
  RPC version: 15
  RPC minimum version: 1

CONFIG
  Configuration directory: /var/lib/transmission/.config/transmission-daemon
  Download directory: /var/lib/transmission/Downloads
  Listenport: 8082
  Portforwarding enabled: Yes
  uTP enabled: Yes
  Distributed hash table enabled: No
  Local peer discovery enabled: No
  Peer exchange allowed: Yes
  Encryption: required
  Maximum memory cache size: 4.00 MiB

LIMITS
  Peer limit: 200
  Default seed ratio limit: Unlimited
  Upload speed limit: Unlimited (Disabled limit: 100 kB/s; Disabled turtle limit: 50 kB/s)
  Download speed limit: Unlimited (Disabled limit: 100 kB/s; Disabled turtle limit: 50 kB/s)

MISC
  Autostart added torrents: No
  Delete automatically added torrents: No
```

\-si 展示 session 信息。



```
$ transmission-remote 8081 -n admin:abc123 -st

CURRENT SESSION
  Uploaded:   115.6 GB
  Downloaded: None
  Ratio:      Inf
  Duration:   13 days (1139476 seconds)

TOTAL
  Started 1 times
  Uploaded:   115.6 GB
  Downloaded: None
  Ratio:      Inf
  Duration:   13 days (1139476 seconds)
```

\-st 展示 session 状态。



```
$ transmission-remote -l
ID     Done       Have  ETA           Up    Down  Ratio  Status       Name
   4   100%    6.23 GB  Done         0.0     0.0    1.4  Idle         EP01-06.mkv
   5   100%    7.39 GB  Done         0.0     0.0    1.0  Idle         EP07-13.mkv
   6   100%    7.32 GB  Done         0.0     0.0    1.0  Idle         EP14-20.mkv
Sum:          20.94 GB               0.0     0.0
$ transmission-remote 8081 -n admin:abc123 -l
ID     Done       Have  ETA           Up    Down  Ratio  Status       Name
   7   100%    6.23 GB  Done         0.0     0.0    0.5  Idle         EP01-06.mkv
   8   100%    7.39 GB  Done         0.0     0.0    0.0  Idle         EP07-13.mkv
   9   100%    7.32 GB  Done         0.0     0.0    0.0  Idle         EP14-20.mkv
 163   100%    1.01 GB  Done         0.0     0.0   17.3  Idle         月光宝盒.mkv
 164   100%    1.51 GB  Done         0.0     0.0   10.9  Idle         西游伏魔篇.rmvb
 165   100%    4.61 GB  Done         0.0     0.0    6.0  Idle         大开眼界.mkv
 166   100%    1.65 GB  Done         0.0     0.0    5.5  Idle         逆转时空.mp4
 167   100%   647.2 MB  Done         0.0     0.0    8.7  Idle         大圣娶亲.rmvb
 168   100%    2.94 GB  Done         0.0     0.0    6.0  Idle         龙马精神.mp4
Sum:          33.32 GB               0.0     0.0
```

\-l 列出所有种子。如果不指定账号密码，则只能查看本地添加的种子，通过 WebUI 添加的看不到。其中 ID 字段很重要，后面索引文件时会使用。



```
$ transmission-remote 8081 -n admin:abc123 -t 168 -i
NAME
  Id: 168
  Name: 龙马精神.mp4
  Hash: b46c1fa9b7e18b2fd91bdb3f6d7130a966865d99
  Magnet: magnet:?xt=urn:btih:b46c1fa9b7e18b2fd91bdb3f6d7130a966865d99&dn=%E9%BE%99%E9%A9%AC%E7%B2%BE%E7%A5%9E.mp4&tr=http%3A%2F%2F1337.abcvg.info%3A80%2Fannounce&tr=http%3A%2F%2Fbvarf.tracker.sh%3A2086%2Fannounce&tr=http%3A%2F%2Fipv6.rer.lol%3A6969%2Fannounce&tr=http%3A%2F%2Fjvavav.com%3A80%2Fannounce&tr=http%3A%2F%2Fretracker.x2k.ru%3A80%2Fannounce&tr=http%3A%2F%2Fyunhai.bcc-bdbl.baidu.com%3A8888%2Fannounce

TRANSFER
  State: Idle
  Location: /ext/torrent/movie
  Percent Done: 100%
  ETA: 0 seconds (0 seconds)
  Download Speed: 0 kB/s
  Upload Speed: 0 kB/s
  Have: 2.94 GB (2.94 GB verified)
  Availability: 100%
  Total size: 2.94 GB (2.94 GB wanted)
  Downloaded: None
  Uploaded: 17.68 GB
  Ratio: Inf
  Corrupt DL: None
  Peers: connected to 1, uploading to 0, downloading from 0

HISTORY
  Date added:       Mon Oct 14 14:35:37 2024
  Date started:     Mon Oct 14 14:36:08 2024
  Latest activity:  Fri Oct 25 11:41:15 2024
  Seeding Time:     11 days (972122 seconds)

ORIGINS
  Date created: Mon Oct 14 14:31:42 2024
  Public torrent: Yes
  Creator: Transmission/2.94 (d8e60ee44f)
  Piece Count: 1404
  Piece Size: 2.00 MiB

LIMITS & BANDWIDTH
  Download Limit: Unlimited
  Upload Limit: Unlimited
  Ratio Limit: Default
  Honors Session Limits: Yes
  Peer limit: 50
  Bandwidth Priority: Normal
```

\-t 指定查看的种子，\-i 展示详情，注意下面的命令都要搭配 \-t 使用。



```
$ transmission-remote 8081 -n admin:abc123 -t 168 -if
龙马精神.mp4 (1 files):
  #  Done Priority Get      Size  Name
  0: 100% Normal   Yes   2.94 GB  龙马精神.mp4
```

\-if 展示种子文件列表详情。



```
$ transmission-remote 8081 -n admin:abc123 -t 168 -it

  Tracker 0: http://1337.abcvg.info:80
  Active in tier 0
  Got a list of 1 peers 4 hours (17784 seconds) ago
  Asking for more peers in 4 hours (17362 seconds)
  Tracker had 0 seeders and 0 leechers 4 hours (14768 seconds) ago
  Asking for peer counts in 6 hours (24449 seconds)

  Tracker 1: http://bvarf.tracker.sh:2086
  Active in tier 1
  Got a list of 1 peers 15 minutes (948 seconds) ago
  Asking for more peers in 1 hour, 44 minutes (6268 seconds)
  Tracker had 1 seeders and 0 leechers 15 minutes (948 seconds) ago
  Asking for peer counts in 14 minutes (859 seconds)

  Tracker 2: http://ipv6.rer.lol:6969
  Active in tier 2
  Got an error "Could not connect to tracker" 13 minutes (780 seconds) ago
  Asking for more peers in 1 hour, 47 minutes (6430 seconds)
  Got a scrape error "Could not connect to tracker" 1 hour, 16 minutes (4619 seconds) ago
  Asking for peer counts in 43 minutes (2599 seconds)

  Tracker 3: http://jvavav.com:80
  Active in tier 3
  Got an error "Could not connect to tracker" 7 minutes (454 seconds) ago
  Asking for more peers in 1 hour, 53 minutes (6789 seconds)
  Got a scrape error "Could not connect to tracker" 1 hour, 18 minutes (4721 seconds) ago
  Asking for peer counts in 41 minutes (2499 seconds)

  Tracker 4: http://retracker.x2k.ru:80
  Active in tier 4
  Got a list of 2 peers 18 minutes (1101 seconds) ago
  Asking for more peers in 21 minutes (1299 seconds)
  Tracker had 1 seeders and 1 leechers 18 minutes (1101 seconds) ago
  Asking for peer counts in 11 minutes (699 seconds)

  Tracker 5: http://yunhai.bcc-bdbl.baidu.com:8888
  Active in tier 5
  Got a list of 2 peers 18 minutes (1114 seconds) ago
  Asking for more peers in 13 minutes (795 seconds)
  Tracker had 1 seeders and 1 leechers 18 minutes (1114 seconds) ago
  Asking for peer counts in 11 minutes (689 seconds)
```

\-it 展示种子 tracker 列表。



```
$ transmission-remote 8081 -n admin:abc123 -t 168 -ic
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 11111111
  11111111 11111111 11111111 11111111 11111111 11111111 11111111 1111
```

\-ic 展示种子分片下载详情，注意这里展示的下载块位图，上传没有位图可显示，因为 peer 可能有多个，且每个可能只申请一部分数据。



```
$ transmission-remote 8081 -n admin:abc123 -t 168 -ip
Address               Flags         Done  Down    Up      Client
```

\-ip 展示种子 peer 详情。



```
$ transmission-remote 8081 -n admin:abc123 -t 168 -S
localhost:8081/transmission/rpc/ responded: "success"
$ transmission-remote 8081 -n admin:abc123 -t 168 -l
ID     Done       Have  ETA           Up    Down  Ratio  Status       Name
 168   100%    2.94 GB  Done         0.0     0.0    6.0  Stopped      龙马精神.mp4
Sum:           2.94 GB               0.0     0.0
$ transmission-remote 8081 -n admin:abc123 -t 168 -s
localhost:8081/transmission/rpc/ responded: "success"
$ transmission-remote 8081 -n admin:abc123 -t 168 -l
ID     Done       Have  ETA           Up    Down  Ratio  Status       Name
 168   100%    2.94 GB  Done         0.0     0.0    6.0  Idle         龙马精神.mp4
Sum:           2.94 GB               0.0     0.0
```

\-S 停止种子任务；\-s 启动种子任务。以此类推：\-a 添加种子，\-r 删除种子，下面重点看下如何借助 transmission\-reomte 添加本地种子资源：



```
$ transmission-remote 8081 -n admin:abc123 -a 盲道.mp4.torrent -w /ext/torrent/movie/
localhost:8081/transmission/rpc/ responded: "success"
$ transmission-remote 8081 -n admin:abc123  -l
ID     Done       Have  ETA           Up    Down  Ratio  Status       Name
   7   100%    6.23 GB  Done         0.0     0.0    0.5  Idle         EP01-06.mkv
   8   100%    7.39 GB  Done         0.0     0.0    0.0  Idle         EP07-13.mkv
   9   100%    7.32 GB  Done         0.0     0.0    0.0  Idle         EP14-20.mkv
 163   100%    1.01 GB  Done         0.0     0.0   17.3  Idle         月光宝盒.mkv
 164   100%    1.51 GB  Done         0.0     0.0   10.9  Idle         西游伏魔篇.rmvb
 165   100%    4.61 GB  Done         0.0     0.0    6.0  Idle         大开眼界.mkv
 166   100%    1.65 GB  Done         0.0     0.0    5.5  Idle         逆转时空.mp4
 167   100%   647.2 MB  Done         0.0     0.0    8.7  Idle         大圣娶亲.rmvb
 168   100%    2.94 GB  Done         0.0     0.0    6.0  Idle         龙马精神.mp4
 173    27%   340.8 MB  Unknown      0.0     0.0    0.0  Verifying    盲道.mp4
Sum:          33.66 GB               0.0     0.0
```

要领就是通过 \-w 参数指定原文件所在路径，这与 WebUI 中的"保存目录"如出一辙。下面是 \-w 选项的 man 说明：



```
 -w   --download-dir                 When used in conjunction with --add,
                                           set the new torrent's download
                                           folder. Otherwise, set the default
                                           download folder
```

单独使用是指定全局的下载目录了。也可以不指定 \-w，直接使用全局的默认下载目录，这需要修改上面介绍过的 setting.json 文件，字段名为 donwload\-dir。


注意观察上面的输出，添加后也有一个 Verify 状态。


### 下载侧


环境搭好了，下面在 mac 端测一下吧\~


这里选取 Transmission mac 客户端做演示，首先将种子加入到 Transmission 中：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028111948619-40698062.png)


种子中记录的 tracker 信息会让 Transmission 连接内网 tracker 进行查询。


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028112019242-20748328.png)


在接下来的界面中选择：


* 要下载的文件
* 本地保存路径
* 任务分组
* 任务优先级
* ....


一会儿就能看到有速度了：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028114525545-1757731290.png)


刷新 tracker 统计，看到 peer 数会增加 1：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028112238455-3592213.png)


Linux 机的 Transmission 也能观察到上传速度了：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028112306935-227509880.png)


在新版 WebUI 我没找到上传进度，老版通过 peer 信息是可以看到的：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028112353712-183852321.png)


Transmission 下载侧也能看到 peer 信息，也是没有进度：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028112614698-522541250.png)


另外还能展示块的信息：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028112815442-1180985786.png)


除了下载完成的块，还能查看已分配正在下载中的：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028112829926-269659520.png)


不得不说很技术范儿\~ 


下载完成后，mac Transmission 也会加入供种行列，查看 tracker 状态：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028113105651-1695692673.png)


发现供种数也增 1。


如果不方便传递种子文件，也可以通过磁力链接下载：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028113852728-26694771.png)


磁力链未解析前很多信息 Transmission 拿不到，所以只能选择存储位置：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028113923236-23655546.png)


及任务分组与优先级：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028114019538-728989460.png)


在查询到种子信息之前，任务没有进度显示：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241028114138445-998875701.png)


拉到种子文件后，就一样了。


### 二次开发


其实公司的产品也是基于 Transmission 库做的二次开发，库名为 btsdk。一般而言，含有内网 tracker 的种子文件和磁力链，是可以直接“喂”给 btsdk，然而实际使用中，由于各种技术原因，产品传递给 sdk 的是只包含 hash 值的磁力链，一些额外的信息被“过滤”掉了，以上面的例子来说，完整的磁力链是：



```
magnet:?xt=urn:btih:7659ead46555d97a98e25a373d8216edc926d78a&dn=%E7%9B%B2%E9%81%93.mp4&tr=http%3A%2F%2Fyunhai.bcc-bdbl.baidu.com%3A8888%2Fannouce
```

传递给 btsdk 的是：



```
magnet:?xt=urn:btih:7659ead46555d97a98e25a373d8216edc926d78a
```

导致 libtransmission 不知道要去内网 tracker 请求种子信息，从而下载超时。


为了解决这个问题，需要在 debug 版本为种子任务加入本地 tracker，下面是摸索过程。


#### tr\_torrentSetTrackerList


这个接口可以在种子任务的类型上直接设置 tracker 列表，简单粗暴但有效，总的流程是：


* tr\_torrentNew 创建种子任务
* tr\_torrentSetTrackerList 添加种子内网 tracker
* 设置超时定时器
* ....


刚开始用是没问题的，然而隔了一天再试，就全超时了。仔细分析后发现，由于目前启用了创建就启动的选项，导致这里有线程竞争：位于调用者线程的 tr\_torrentSetTrackerList，和位于 libtransmission 工作线程中使用 tracker 列表进行请求的地方 (通过 tr\_torrentNew 触发) 存在竞争，后者可能先于前者被系统调度，这里插入的 tracker 就不会被用到，进而导致超时。


这个问题也有简单的解决方案，就是统一设置创建种子后不自启动，等到任务准备好后，手动调用 tr\_torrentStart 再启动。


不过我没有尝试这种方案，而是寻求在全局记录额外的 tracker 信息，当 libtransmission 启动任务时，会从全局记录的信息中将额外的 tracker 添加到种子任务，从而实现内网 tracker 的访问。


#### TR\_KEY\_trackerList


在 key 枚举中搜索，得到了这个标识：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241029204452808-2020560597.png)


设置一下试试：



```
tr_variant settings = {};
tr_variantInitDict(&settings, 0);
...
tr_variantDictAddStr(&settings, TR_KEY_trackerList, BT_DEBUG_TRACKER_URLS); 
...
tr_sessionLoadSettings(&settings, param.config_dir, "btsdk");
tr_session* session = tr_sessionInit(param.config_dir, true, &settings);
...
tr_ctor* ctor = tr_ctorNew(session);
int n = tr_sessionLoadTorrents(session, ctor);
tr_ctorFree(ctor);
```

这段代码是初始化的骨架，主要流程是：


* tr\_variantDictAddXXX 添加设置项
* tr\_sessionLoadSettings 加载设置，上面添加的设置会覆盖本地读取的设置
* tr\_sessionInit 创建会话，一个 transmission 会话可以包含多个种子文件。设置由这里传入
* tr\_ctorNew / tr\_ctorFree / tr\_sessionLoadTorrents 加载本地种子，用于种子持久化后的恢复


按理说 session 通过 settings 可以拿到 tracker 设置，然而实际跑了一下，新代码没有起作用，内网种子仍然超时。


#### TR\_KEY\_default\_trackers


继续搜索，发现了这个，感觉要靠谱不少：


![](https://img2024.cnblogs.com/blog/1707550/202410/1707550-20241029205625591-963836065.png)


替换上面代码中的 key 后，内网种子果然有速度啦，这两个 key 的差别就需要分析源码了，具体的就不在这里展开分析了，毕竟这只是一篇环境搭建的文章，搞的太深奥了会劝退一部分读者，哈哈\~


## 后记


复盘一下整个探索过程，发现最困难的是确认 Transmission 是否声明 (announce) peer 信息到 tracker 这一步。以 opentracker 这简陋的输出，不要说看种子下面有哪些 peer，就连有哪些种子都看不到，打开繁琐输出 (`mode=everything`) 也是一样的抽象：



```
xml version="1.0" encoding="UTF-8"?
<stats>
  <tracker_id>121091771tracker_id>
  <version>
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
$Source$: $Revision$
  version>
  <uptime>1557624uptime>
  <torrents>
    <count_mutex>10count_mutex>
    <count_iterator>10count_iterator>
  torrents>
  <peers>
    <count>15count>
  peers>
  <seeds>
    <count>13count>
  seeds>
  <completed>
    <count>105count>
  completed>
  <connections>
    <tcp>
      <accept>37716accept>
      <announce>24538announce>
      <scrape>12655scrape>
    tcp>
    <udp>
      <overall>0overall>
      <connect>0connect>
      <announce>0announce>
      <scrape>0scrape>
      <missmatch>0missmatch>
    udp>
    <livesync>
      <count>0count>
    livesync>
  connections>
  <debug>
    <renew>
      <count interval="00">20count>
      <count interval="01">475count>
      <count interval="02">4count>
      <count interval="03">5count>
      <count interval="04">8count>
      <count interval="05">5count>
      <count interval="06">8count>
      <count interval="07">5count>
      <count interval="08">3count>
      <count interval="09">8count>
      <count interval="10">5count>
      <count interval="11">12count>
      <count interval="12">3count>
      <count interval="13">12count>
      <count interval="14">10count>
      <count interval="15">2count>
      <count interval="16">6count>
      <count interval="17">4count>
      <count interval="18">2count>
      <count interval="19">3count>
      <count interval="20">1count>
      <count interval="21">0count>
      <count interval="22">0count>
      <count interval="23">0count>
      <count interval="24">0count>
      <count interval="25">0count>
      <count interval="26">2count>
      <count interval="27">1957count>
      <count interval="28">3920count>
      <count interval="29">3833count>
      <count interval="30">3799count>
      <count interval="31">3716count>
      <count interval="32">3888count>
      <count interval="33">1875count>
      <count interval="34">0count>
      <count interval="35">0count>
      <count interval="36">0count>
      <count interval="37">1count>
      <count interval="38">0count>
      <count interval="39">0count>
      <count interval="40">0count>
      <count interval="41">0count>
      <count interval="42">0count>
      <count interval="43">0count>
      <count interval="44">1count>
    renew>
    <http_error>
      <count code="302 Redirect">0count>
      <count code="400 Parse Error">22count>
      <count code="400 Invalid Parameter">1count>
      <count code="400 Invalid Parameter (compact=0)">0count>
      <count code="400 Not Modest">0count>
      <count code="402 Payment Required">0count>
      <count code="403 Access Denied">0count>
      <count code="404 Not found">152count>
      <count code="500 Internal Server Error">1count>
    http_error>
    <mutex_stall>
      <count>0count>
    mutex_stall>
  debug>
stats>
```

带来的问题就是无法区分连不通的原因：是 Transmisstion 没上报？还是上报了地址不行？


### 方案 I：libtransmission 增加日志输出


早期的一个解决方案，是在 libtransmission 源码中增加了日志，再通过 btsdk 将 libtransmission 的日志打印到文件中查看，效果如下：



```
2024-11-13 14:36:26 -- tracker knows of 1 seeders and 1 leechers and gave a list of 2 peers.
2024-11-13 14:36:26 -- pex 0: [10.127.82.16]:51413
2024-11-13 14:36:26 -- pex 1: [10.138.62.136]:8082
2024-11-13 14:36:26 -- peer counts: 1 seeders, 1 leechers.
2024-11-13 14:36:26 -- tracker knows of 1 seeders and 1 leechers and gave a list of 2 peers.
2024-11-13 14:36:26 -- pex 0: [10.127.82.16]:51413
2024-11-13 14:36:26 -- pex 1: [10.138.62.136]:8082
2024-11-13 14:36:26 -- peer counts: 1 seeders, 1 leechers.
```

tracker 返回了两条记录：


* 51413 端口这条记录就是 sdk 自身
* 8082 端口就是 Linux 机上的 Transmission 服务，IP 是能对的上的


增加的日志代码如下：



```
void publishPeersPex(tr_tier* tier, int seeders, int leechers, std::vector const& pex)
{
    if (tier->tor->torrent_announcer->callback != nullptr)
    {
        auto e = tr_tracker_event{};
        e.type = tr_tracker_event::Type::Peers;
        e.seeders = seeders;
        e.leechers = leechers;
        e.pex = pex;
        tr_logAddDebugTier(
            tier,
            fmt::format(
                "tracker knows of {} seeders and {} leechers and gave a list of {} peers.",
                seeders,
                leechers,
                std::size(pex)));

        for (auto i=0; i"pex {}: {}",
                        i, 
                        pex[i].display_name())); 
        }


        tier->tor->torrent_announcer->callback(*tier->tor, &e);
    }
}
```

for 循环中的 tr\_logAddDebugTier 就是了。在请求 tracker 响应中会调用这个 publishPeersPex，具体调用链也没深究，主要是根据上面这条日志顺藤摸瓜而来：



```
tracker knows of 1 seeders and 1 leechers and gave a list of 2 peers.
```

这种方式需要修改源码、编译，非常不便。


### 方案 II：模拟请求 tracker


另一种解决方案是直接向 tracker 发送模拟请求，这一点是受 transmission\-show \-s 命令的启发：



```
$ transmission-show -s 月光宝盒.mkv.torrent
Name: 月光宝盒.mkv
File: 月光宝盒.mkv.torrent

http://1337.abcvg.info:80/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... 0 seeders, 0 leechers
http://bvarf.tracker.sh:2086/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... no match
http://ipv6.rer.lol:6969/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... error: Couldn't connect to server
http://jvavav.com:80/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... error: Couldn't resolve host name
http://retracker.x2k.ru:80/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... no match
http://yunhai.bcc-bdbl.baidu.com:8888/scrape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19 ... 1 seeders, 0 leechers
```

opentracker 支持 scrape 查询操作，这个接口返回的信息中包含了 peer 数量与供种数量，有可能会包含 peer 列表。查询了一些公开的资料，它的主要参数有下面几个：


* info\_hash：sha1 效验码，共 20 比特
* peer\_id：BT 客户端的唯一标识，在客户端启动时产生，共 20 比特
* ip：可选，不提供时服务端会自己找到
* port：监听端口
* uploaded/downloaded：上传/下载的字节数
* left：还需下载的字节数
* numwant：可选，客户端希望从 Tracker 服务器得到的 peer 数量
* key：可选，一个扩展的唯一性标识，即使改变了IP地址，也可以使用该字段标识该 BT 客户机
* compact：压缩标志。如果值为 1 表示接受压缩格式的 peer 列表，即用 6 字节表示一个 peer 地址 (前 4 字节表示 IP 地址，后 2 字节表示端口号)；值为 0 表示不接受


下面是一个请求示例：



```
$ curl "http://yunhai.bcc-bdbl.baidu.com:8888/scape?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19&peer_id=-TR2940-yhp3i52s0fyz&port=8088&uploaded=0&downloaded=0&left=1007978447&compact=0&numwant=10&event=started"
d5:filesd20:�lG�;;��l��rC�d�d8:completei1e10:downloadedi16e10:incompletei0eeee
```

由于返回内容是 bencode 编码的，一些二进制内容在 Console 窗口中显示为乱码。


结果解析放下不表，先看如何自动构造这个请求，后面通过脚本可以对任意一个种子文件发起请求。


#### 请求组装


请求中可选的字段一律不填，剩下的字段都填写默认值：


* tracker list 地址列表
* info\_hash SHA1 填写为 torrent 中记录的值，注意是 hex 直接 url encode
* peer\_id 唯一 ID 固定设置一个随机值即可，需要 url encode
* port 侦听端口随便设置，因为后面不请求数据不会用到，这里为 8088
* downloaded 下载量设置为 0，表示从头下载
* uploaded 上传量不设置
* left 文件大小可设置为 0
* compact 压缩标志设置为 0，方便后面解析
* numwant 想要的 peer 数量可设置为 10，这里应该只有一两个 peer


这里必填的信息有 tracker 地址、info\_hash 字段，都可以使用 transmission\-show 获取到：



```
$ transmission-show 月光宝盒.mkv.torrent
Name: 月光宝盒.mkv
File: 月光宝盒.mkv.torrent

GENERAL

  Name: 月光宝盒.mkv
  Hash: db6c47953b3bddfdbc6c8cf0bf7243ba641ed519
  Created by: Transmission/2.94 (d8e60ee44f)
  Created on: Mon Oct 14 14:29:13 2024
  Piece Count: 1923
  Piece Size: 512.0 KiB
  Total Size: 1.01 GB
  Privacy: Public torrent

TRACKERS

  Tier #1
  http://1337.abcvg.info:80/announce

  Tier #2
  http://bvarf.tracker.sh:2086/announce

  Tier #3
  http://ipv6.rer.lol:6969/announce

  Tier #4
  http://jvavav.com:80/announce

  Tier #5
  http://retracker.x2k.ru:80/announce

  Tier #6
  http://yunhai.bcc-bdbl.baidu.com:8888/announce

FILES

  月光宝盒.mkv (1.01 GB)
```

下面在脚本中提取它们：



```
#! /bin/sh

# @brief: url encode string
# @param: text
# @return: encoded-text
function url_encode()
{
    local input=$1
    local output=""
    # yum install gridsite-clients
    # type "urlencode" > /dev/null 2>&1
    # if [ $? -ne 0 ]; then
        # output=$(echo "${input}" | tr -d '\n' | xxd -plain | tr -d '\n' | sed 's/\(..\)/%\1/g')
        local n=0
        while [ $n -lt ${#input} ];
        do
            case ${input:$n:1} in
                [a-z]|[A-Z]|[0-9]|.|-|_)
                    # regular urlencode only replace aboving characters
                    output="${output}${input:$n:1}"
                    # echo "${input:$n:1}" >> "${BASEDIR}/raw.data"
                    ;;
                *)
                    # for chinese charactor, more than one chars are replaced
                    output="${output}$(echo "${input:$n:1}" | tr -d '\n' | xxd -plain | tr -d '\n' | sed 's/\(..\)/%\1/g')"
                    # echo "${input:$n:1}" >> "${BASEDIR}/raw.data"
                    ;;
            esac
            n=$((n+1))
        done
    # linux urlencode is problemly while handling chinese characters
    # else
    #     output=$(urlencode "${input}")
    # fi

    # echo "${input} after urlencode: ${output}" >> "${BASEDIR}/raw.data"
    echo "${output}"
}

function hexstr2urlenc()
{
    bin=$(echo $1 | xxd -r -p)
    enc=$(url_encode "${bin}")
    #echo "after url encode: ${enc}"
    echo "${enc}"
}

function main()
{
    local file=$1
    infohash=$(transmission-show "${file}"  | grep Hash | awk '{print $2}')
    peerid='2d5452323934302d79687033693532733066797a' # hard coded
    port=8088
    # size=$(stat -c"%s" "${file}")
    size=0
    echo "infohash: ${infohash}, peerid ${peerid}, size ${size}"

    infohash_enc=$(hexstr2urlenc "${infohash}")
    peerid_enc=$(hexstr2urlenc "${peerid}")
    echo "after url encode: ${infohash_enc}, ${peerid_enc}"

    transmission-show "${file}" | sed -n '/TRACKERS/,/FILES/p' | sed '1d;$d;/^$/d;/Tier/d;s/announce/scrape/' > ${file}.tracker
    while read tracker; do
        echo "consult tracker $tracker"
        # echo "${tracker}?info_hash=${infohash_enc}&peer_id=${peerid_enc}&port=$port&uploaded=0&downloaded=0&left=$size&compact=0&numwant=10"
        resp=$(curl -s "${tracker}?info_hash=${infohash_enc}&peer_id=${peerid_enc}&port=$port&uploaded=0&downloaded=0&left=$size&compact=0&numwant=10&event=started")
        if [ -z "${resp}" ]; then
            echo "no data"
            continue
        fi

        echo "${resp}"
        echo
    done < ${file}.tracker
    rm "${file}.tracker"
}

main $@
```

需要注意的是无论是 info\_hash 还是 peer\_id，得到的已经是 hex string，需要先将它们转回为十六进制，再进行 url\_encode，否则会请求失败。


下面是脚本的运行输出：



```
$ sh consult_tracker.sh 月光宝盒.mkv.torrent
infohash: db6c47953b3bddfdbc6c8cf0bf7243ba641ed519, peerid 2d5452323934302d79687033693532733066797a, size 0
after url encode: %dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19, -TR2940-yhp3i52s0fyz
consult tracker http://1337.abcvg.info:80/scrape
no data
consult tracker http://bvarf.tracker.sh:2086/scrape
d5:filesdee

consult tracker http://ipv6.rer.lol:6969/scrape
no data
consult tracker http://jvavav.com:80/scrape
no data
consult tracker http://retracker.x2k.ru:80/scrape
d5:filesd40:db6c47953b3bddfdbc6c8cf0bf7243ba641ed519d8:completei1e10:downloadedi1e10:incompletei0eee5:flagsd20:min_request_intervali300eee

consult tracker http://yunhai.bcc-bdbl.baidu.com:8888/scrape
d5:filesd20:�lG�;;��l��rC�d�d8:completei1e10:downloadedi16e10:incompletei0eeee
```

能请求通。过程中会生成 tracker.txt，可以看到脚本处理的中间结果：



```
$ cat 月光宝盒.mkv.torrent.tracker
  http://1337.abcvg.info:80/scrape
  http://bvarf.tracker.sh:2086/scrape
  http://ipv6.rer.lol:6969/scrape
  http://jvavav.com:80/scrape
  http://retracker.x2k.ru:80/scrape
  http://yunhai.bcc-bdbl.baidu.com:8888/scrape
```

注意已经将 announce 接口替换为了 scrape。


#### 结果解析


opentracker 返回的信息是经过 bencode 的，bencode 是一种 BitTorrent 专用的传输格式，主要上的是减少文本的空间占用，感兴趣的可以查看附录 7。


如果想查看文本形式的内容，还需要转换一下。经过一番搜索，发现有基于 python 的解析器：bencodepy



```
$ pip3 install bencode.py
Collecting bencode.py
  Downloading https://files.pythonhosted.org/packages/15/9f/eabbc8c8a16db698d9c4bd24953763df2594b054237b89afe1ec56d3965e/bencode.py-4.0.0-py2.py3-none-any.whl
Installing collected packages: bencode.py
Successfully installed bencode.py-4.0.0
```

bencodepy 是在 Python 环境中调用的，在 shell 中使用还得封装一下：



```
#! /usr/bin/python3
import bencodepy
import sys

json = bencodepy.decode(sys.argv[1]) 
print ('%s' % json)
```

文件命名为 bdecode.py 放在脚本同名目录，待解析的内容放在第一个参数，就可以这样调用了：



```
$ python3 bdecode.py 'd5:filesd40:db6c47953b3bddfdbc6c8cf0bf7243ba641ed519d8:completei2e10:downloadedi2e10:incompletei0eee5:flagsd20:min_request_intervali300eee' 
{b'files': {b'db6c47953b3bddfdbc6c8cf0bf7243ba641ed519': {b'complete': 2, b'downloaded': 2, b'incomplete': 0}}, b'flags': {b'min_request_interval': 300}}
```

能成功解析。在脚本中增加解析代码再次运行上面的示例：



```
$ sh consult_tracker.sh 月光宝盒.mkv.torrent
infohash: db6c47953b3bddfdbc6c8cf0bf7243ba641ed519, peerid 2d5452323934302d79687033693532733066797a, size 0
after url encode: %dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19, -TR2940-yhp3i52s0fyz
consult tracker http://1337.abcvg.info:80/scrape
d5:filesd20:�lG�;;��l��rC�d�d8:completei1e10:downloadedi1e10:incompletei0eee5:flagsd20:min_request_intervali41354eee

Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/bencodepy/decoder.py", line 84, in decode
    value = to_binary(value)
  File "/usr/local/lib/python3.6/site-packages/bencodepy/compat.py", line 28, in to_binary
    return s.encode('utf-8', 'strict')
UnicodeEncodeError: 'utf-8' codec can't encode character '\udcdb' in position 12: surrogates not allowed

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "./bdecode.py", line 5, in 
    json = bencodepy.decode(sys.argv[1])
  File "/usr/local/lib/python3.6/site-packages/bencodepy/__init__.py", line 155, in bdecode
    return DEFAULT.decode(value)
  File "/usr/local/lib/python3.6/site-packages/bencodepy/__init__.py", line 72, in decode
    return self.decoder.decode(value)
  File "/usr/local/lib/python3.6/site-packages/bencodepy/decoder.py", line 87, in decode
    raise BencodeDecodeError("not a valid bencoded string")
bencodepy.exceptions.BencodeDecodeError: not a valid bencoded string

consult tracker http://bvarf.tracker.sh:2086/scrape
no data
consult tracker http://ipv6.rer.lol:6969/scrape
no data
consult tracker http://jvavav.com:80/scrape
no data
consult tracker http://retracker.x2k.ru:80/scrape
d5:filesd40:db6c47953b3bddfdbc6c8cf0bf7243ba641ed519d8:completei1e10:downloadedi1e10:incompletei0eee5:flagsd20:min_request_intervali300eee

{b'files': {b'db6c47953b3bddfdbc6c8cf0bf7243ba641ed519': {b'complete': 1, b'downloaded': 1, b'incomplete': 0}}, b'flags': {b'min_request_interval': 300}}
consult tracker http://yunhai.bcc-bdbl.baidu.com:8888/scrape
d5:filesd20:�lG�;;��l��rC�d�d8:completei1e10:downloadedi16e10:incompletei0eeee

Traceback (most recent call last):
  File "/usr/local/lib/python3.6/site-packages/bencodepy/decoder.py", line 84, in decode
    value = to_binary(value)
  File "/usr/local/lib/python3.6/site-packages/bencodepy/compat.py", line 28, in to_binary
    return s.encode('utf-8', 'strict')
UnicodeEncodeError: 'utf-8' codec can't encode character '\udcdb' in position 12: surrogates not allowed

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "./bdecode.py", line 5, in 
    json = bencodepy.decode(sys.argv[1])
  File "/usr/local/lib/python3.6/site-packages/bencodepy/__init__.py", line 155, in bdecode
    return DEFAULT.decode(value)
  File "/usr/local/lib/python3.6/site-packages/bencodepy/__init__.py", line 72, in decode
    return self.decoder.decode(value)
  File "/usr/local/lib/python3.6/site-packages/bencodepy/decoder.py", line 87, in decode
    raise BencodeDecodeError("not a valid bencoded string")
bencodepy.exceptions.BencodeDecodeError: not a valid bencoded string
```

看起来效果一般，可能是 info\_hash 等字段解析后是二进制，python 无法直接打印。通过 xxd 转为二进制查看会好一些：



```
    while read tracker; do
        echo "consult tracker $tracker"
        resp=$(curl -s "${tracker}?info_hash=${infohash_enc}&peer_id=${peerid_enc}&port=$port&uploaded=0&downloaded=0&left=$size&compact=0&numwant=10")
        if [ -z "${resp}" ]; then
            echo "no data"
            continue
        fi

        echo "${resp}"
        echo
        echo "${resp}" | xxd -u -g 1
        #echo $(python3 ./bdecode.py "$resp")
    done < ${file}.tracker
```

下面是新的输出：



```
$ sh consult_tracker.sh 月光宝盒.mkv.torrent
infohash: db6c47953b3bddfdbc6c8cf0bf7243ba641ed519, peerid 2d5452323934302d79687033693532733066797a, size 0
after url encode: %dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19, -TR2940-yhp3i52s0fyz
consult tracker http://1337.abcvg.info:80/scrape
d5:filesd20:�lG�;;��l��rC�d�d8:completei0e10:downloadedi0e10:incompletei0eee5:flagsd20:min_request_intervali34648eee

0000000: 64 35 3A 66 69 6C 65 73 64 32 30 3A DB 6C 47 95  d5:filesd20:.lG.
0000010: 3B 3B DD FD BC 6C 8C F0 BF 72 43 BA 64 1E D5 19  ;;...l...rC.d...
0000020: 64 38 3A 63 6F 6D 70 6C 65 74 65 69 30 65 31 30  d8:completei0e10
0000030: 3A 64 6F 77 6E 6C 6F 61 64 65 64 69 30 65 31 30  :downloadedi0e10
0000040: 3A 69 6E 63 6F 6D 70 6C 65 74 65 69 30 65 65 65  :incompletei0eee
0000050: 35 3A 66 6C 61 67 73 64 32 30 3A 6D 69 6E 5F 72  5:flagsd20:min_r
0000060: 65 71 75 65 73 74 5F 69 6E 74 65 72 76 61 6C 69  equest_intervali
0000070: 33 34 36 34 38 65 65 65 0A                       34648eee.
consult tracker http://bvarf.tracker.sh:2086/scrape
no data
consult tracker http://ipv6.rer.lol:6969/scrape
no data
consult tracker http://jvavav.com:80/scrape
no data
consult tracker http://retracker.x2k.ru:80/scrape
d5:filesd40:db6c47953b3bddfdbc6c8cf0bf7243ba641ed519d8:completei1e10:downloadedi1e10:incompletei0eee5:flagsd20:min_request_intervali300eee

0000000: 64 35 3A 66 69 6C 65 73 64 34 30 3A 64 62 36 63  d5:filesd40:db6c
0000010: 34 37 39 35 33 62 33 62 64 64 66 64 62 63 36 63  47953b3bddfdbc6c
0000020: 38 63 66 30 62 66 37 32 34 33 62 61 36 34 31 65  8cf0bf7243ba641e
0000030: 64 35 31 39 64 38 3A 63 6F 6D 70 6C 65 74 65 69  d519d8:completei
0000040: 31 65 31 30 3A 64 6F 77 6E 6C 6F 61 64 65 64 69  1e10:downloadedi
0000050: 31 65 31 30 3A 69 6E 63 6F 6D 70 6C 65 74 65 69  1e10:incompletei
0000060: 30 65 65 65 35 3A 66 6C 61 67 73 64 32 30 3A 6D  0eee5:flagsd20:m
0000070: 69 6E 5F 72 65 71 75 65 73 74 5F 69 6E 74 65 72  in_request_inter
0000080: 76 61 6C 69 33 30 30 65 65 65 0A                 vali300eee.
consult tracker http://yunhai.bcc-bdbl.baidu.com:8888/scrape
d5:filesd20:�lG�;;��l��rC�d�d8:completei1e10:downloadedi16e10:incompletei0eeee

0000000: 64 35 3A 66 69 6C 65 73 64 32 30 3A DB 6C 47 95  d5:filesd20:.lG.
0000010: 3B 3B DD FD BC 6C 8C F0 BF 72 43 BA 64 1E D5 19  ;;...l...rC.d...
0000020: 64 38 3A 63 6F 6D 70 6C 65 74 65 69 31 65 31 30  d8:completei1e10
0000030: 3A 64 6F 77 6E 6C 6F 61 64 65 64 69 31 36 65 31  :downloadedi16e1
0000040: 30 3A 69 6E 63 6F 6D 70 6C 65 74 65 69 30 65 65  0:incompletei0ee
0000050: 65 65 0A
```

最后一个请求的输出，可以看到二进制 info\_hash 的存在：db6c47953b3bddfdbc6c8cf0bf7243ba641ed519，从 0000000 行后半部分一直延伸到 0000010。


经过一番确认，这里仍没有返回 peer 的 IP \& port 信息，想要查询它们，还得请求 announce 接口，它的请求参数与 scrape 基本相同，仅增加：


* event：可选参数，可能取值为 started、completed、stopped，可以分别在下载开始、下载完成和停止下载时发送，这里设置为 started


因为是可选参数，就没添加，直接将 scrape 接口改为 annouce 进行请求，下面是一个示例：



```
$ curl http://1337.abcvg.info:80/announce?info_hash=%dblG%95%3b%3b%dd%fd%bcl%8c%f0%bfrC%bad%1e%d5%19&peer_id=-TR2940-yhp3i52s0fyz&port=8088&uploaded=0&downloaded=0&left=0&compact=0&numwant=10
d8:intervali39012e5:peersld2:ip13:10.138.62.1367:peer id25:-TR2940-yhp3i52s0fyz4:porti8088eeee
```

约摸能看到 ip 字段，值为：10\.138\.62\.136，与 Linux 机地址也能对得上。


不过上面的请求内网自建的 tracker 会返回错误：



```
Invalid Request
```

怀疑是字段没设置对，感兴趣的读者可以排查下原因。


## 总结


本文记录了 BitTorrent 内网测试环境的搭建过程，特别是没有公网设备的场景。


如果已经有公网设备，可以直接使用国内一些活跃的公共 tracker，具体请参考附录 9，作者会不定时更新。


需要注意的是，这里面一部分是 PT 站点，没有身份是不能上报种子的，PT 是 BT 的深化，即私有种子。用户注册站点时会分配一个 passkey，之后使用这个 key 做种子的上传下载。基于种子身份，站点可以做供种时长的统计，对于只下载不上传的“吸血”用户，可以进行有效治理，以提升社区的健康度。这种站点一般不公开注册，需要邀请码才能进入，但分享的资源也有速度保证。


## 参考


\[1]. [CentOS7上OpenTracker的搭建](https://github.com)


\[2]. [opentracker 搭建自己的 BT Tracker 服务器](https://github.com)


\[3]. [搭建自己的 BT Tracker](https://github.com)


\[4]. [Transmission 搭建记录](https://github.com)


\[5]. [BT（带中心Tracker）通信协议的分析](https://github.com)


\[6]. [BitTorrent Tracker 协议详解](https://github.com)


\[7]. [BT 协议规范文档中文版](https://github.com)


\[8]. [bencode.py · PyPI](https://github.com)


\[9]. [XIU2/TrackersListCollection](https://github.com)


\[10]. [BT Tracker的原理及.Net Core简单实现Tracker Server](https://github.com/yangzhili/p/10092675.html "发布于 2018-12-09 19:39"):[楚门加速器](https://shexiangshi.org)


\[11]. [Linux \| 如何挂PT：CentOS 7安装配置美化Transmission及制作种子](https://github.com)


\[12]. [PT站种子制作发布新手全攻略](https://github.com)


\[13]. [制作BT(BitTorrent)种子和磁力链接教程通过BT分享文件](https://github.com)


\[14]. [如何用 Transmission 做种](https://github.com)


\[15]. [PT作弊与反作弊](https://github.com)


\[16]. [实现DHT网络上的种子爬虫](https://github.com)


\[17]. [杂谈网络协议之种子与P2P](https://github.com) 


\[18]. [一次对BT种子的追踪小记](https://github.com)


 


