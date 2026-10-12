# Docker服务器推荐：从 2核2G 起步，自建服务和小项目该怎么挑配置、看流量、省预算

搜「Docker 服务器推荐」的人，通常不是想听一堆虚拟化原理，而是已经撞到墙了：容器起来不到半天就被 OOM Killer 干掉，日志里一行 exit code 137；或者在共享主机上折腾半天，Docker 装上了，但端口映射怎么都不通。

这两件事都不是配置写错了。共享主机把物理内存池给所有租户共用，再用 cgroups 给你划一块上限，容器一超就被内核杀掉。加上它通常不允许容器需要的那些内核命名空间，装得上也跑不稳。结论很直接：**要跑 Docker，就得有一台有独立内核、独立内存配额、有 root 权限的机器**，也就是 KVM 云服务器或独立服务器。

至于配置怎么定、钱花在哪儿，下面按实际会遇到的顺序讲一遍，最后附上能跑容器的服务器套餐对比（含价格和购买入口）。

## Docker 到底吃什么资源，先定个下限

把「跑得起来」和「跑得舒服」分开看，会省很多来回试错的时间。

| 使用场景 | 最低内存 | 建议内存 | 建议 vCPU | 硬盘 |
| --- | --- | --- | --- | --- |
| 单个容器，测试用 | 1GB | 2GB | 1 核 | 20GB |
| Docker Compose（2–4 个服务：应用 + 数据库 + 缓存 + 反代） | 2GB | 4GB | 2 核 | 40GB |
| Compose 5 个服务以上，正式跑 | 4GB | 8GB | 4 核 | 80GB |
| 单节点 K3s 轻量 Kubernetes | 4GB | 8GB | 2 核 | 60GB |

1GB 内存不是完全不能用，但实测能放的只有两三个极轻量容器（Alpine 类镜像）。只要你要跑 MySQL 或 Postgres，2GB 是实际的下限，4GB 才是常见起点。

硬盘比内存更容易被忽略。一个 Node 或 Java 的基础镜像动辄 1GB 以上，几次版本更新后旧镜像堆积，25GB 的盘说满就满。另外日志是隐形杀手：默认的 json-file 日志驱动不轮转，一个话多的容器能自己把磁盘写满。开工第一天就把 `log-opts` 里的 `max-size` 设上，用 `docker system df` 定期看一眼。

## 挑 Docker 服务器，这 5 件事比 CPU 型号更影响体验

**1. 计费方式是限流量还是不限流量。** 这条排第一，因为它决定账单会不会失控。Docker 拉镜像、容器之间互传数据、对外提供 API，都算流量。有些区域是「大带宽 + 限流量」，用完限速不停机；有些是「小带宽 + 不限流量」。前者跑突发任务划算，后者适合长期稳定输出。上下行双向计费的话，别只算下行。

**2. 有没有预装 Docker 环境。** 省的是最恶心的一步。国内机器直连 Docker 官方仓库经常超时，很多时候不是装不上，是拉不动。

**3. 镜像加速怎么配。** 国内机器基本都得改 `/etc/docker/daemon.json`，加 `registry-mirrors`。雨云官方文档里给的做法就是这一步，用的是南京大学镜像源，改完 `systemctl restart docker` 生效。

**4. 磁盘能不能单独扩。** 套餐升一级通常要连 CPU 内存一起加钱，但如果只是镜像放不下，能单独扩盘更划算。

**5. 高防和备案。** 容器一旦对外暴露端口，就等于把你的服务放到了公网。跑在需要备案的大陆机房，域名和常用端口的 HTTP 业务要先备案再过白；不想走这套流程，就选香港、日本、韩国或美国。

顺带两个安全点，跟选商家无关但值得知道：把用户加进 `docker` 组等于给了 root 级权限（能控制 daemon 就能把宿主文件挂进容器），介意的话用 rootless 模式；另外容器发布的端口会绕过 ufw/firewalld 规则，别以为防火墙拦得住，反代之外的端口记得绑到 `127.0.0.1`。

## 雨云跑 Docker：预装环境和面板是什么情况

雨云（RainYun）是 2018 年成立的国产云服务商，持有工信部颁发的 ISP、IDC、CDN 证，产品线包括云服务器、游戏云、裸金属物理机、显卡云、对象存储、CDN 等。对跑容器这件事，有几个点比较实际：

- 下单时可以选预安装应用，列表里有 **Docker 环境**，也可以选宝塔、1Panel、MCSManager、Java 环境，能跟系统一起装好，省掉自己装 Docker 那步
- 系统镜像可选 Debian 12 / Debian 11 / Ubuntu 22.04 等；官方文档里的示例场景（2核2G + Debian 12 + 预装 Docker 环境）就是「一台机器跑一个容器服务」的典型配置
- 官方文档有完整教程：《雨云服务器使用 Docker 搭建个人网盘 NextCloud 并接入雨云对象存储 ROS》，从选配置、装 Docker、配镜像源到跑 compose 都有，新手上手可以照着走
- 后台能看到云服务器状态，支持重装/切换系统、升级配置

它的产品线里还有虚拟主机类产品，但那是给纯建站用的，需要独立内核和 root 权限的场景不该往那上面考虑。

👉 [先看一眼当前开放的区域和可选配置](https://www.rainyun.com/MzA2MTc2_)

## 免备案区域套餐对比（香港、日本、韩国、美国）

价格均为各区域公开月付价，同一区域常按带宽和流量档位拆成多套方案，下表列的是常见档位。

**美国（AMD EPYC 7702 · 流量不限型）**

| 套餐 | 配置 | 带宽 | 月付 | 购买 |
| --- | --- | --- | --- | --- |
| 入门版 | 1核 / 1GB / 30–2000GB | 上行20M 下行30M | ¥25 | [选这个入门档](https://www.rainyun.com/MzA2MTc2_) |
| 标准版 | 2核 / 2GB | 上行30M 下行50M | ¥34 | [查看标准版](https://www.rainyun.com/MzA2MTc2_) |
| 中配版 | 2核 / 4GB | 上行40M 下行60M | ¥44 | [看中配版配置](https://www.rainyun.com/MzA2MTc2_) |
| 进阶版 | 4核 / 4GB | 上行60M 下行80M | ¥60 | [进阶版价格](https://www.rainyun.com/MzA2MTc2_) |
| 高配版 | 4核 / 8GB | 上行80M 下行100M | ¥113 | [高配版详情](https://www.rainyun.com/MzA2MTc2_) |
| 旗舰版 | 8核 / 16GB | 上行100M 下行150M | ¥197 | [旗舰版方案](https://www.rainyun.com/MzA2MTc2_) |

**中国香港（Xeon Gold · 流量叠加型）**

| 套餐 | 配置 | 带宽 | 月付 | 购买 |
| --- | --- | --- | --- | --- |
| 入门版 | 1核 / 1GB | 10M 双向 | ¥28 | [香港入门版](https://www.rainyun.com/MzA2MTc2_) |
| 标配版 | 2核 / 2GB | 20M 双向 | ¥35 | [香港标配版](https://www.rainyun.com/MzA2MTc2_) |
| 中配版 | 2核 / 4GB | 20M 双向 | ¥48 | [香港中配版](https://www.rainyun.com/MzA2MTc2_) |
| 高配版 | 4核 / 4GB | 30M 双向 | ¥68 | [香港高配版](https://www.rainyun.com/MzA2MTc2_) |
| 顶配版 | 4核 / 8GB | 30M 双向 | ¥108 | [香港顶配版](https://www.rainyun.com/MzA2MTc2_) |
| 旗舰版 | 8核 / 16GB | 50M 双向 | ¥188 | [香港旗舰版](https://www.rainyun.com/MzA2MTc2_) |

**中国香港五区（EPYC 7K62/7702 · 国际大带宽）**

| 套餐 | 配置 | 流量 · 带宽 | 月付 | 购买 |
| --- | --- | --- | --- | --- |
| 入门版 | 1核 / 1GB / 30GB | 1TB · 200Mbps | ¥23 | [香港五区入门档](https://www.rainyun.com/MzA2MTc2_) |
| 标准版 | 2核 / 2GB / 30GB | 1TB · 500Mbps | ¥30 | [香港五区标准档](https://www.rainyun.com/MzA2MTc2_) |
| 中配版 | 2核 / 4GB / 30GB | 1TB · 500Mbps | ¥43 | [香港五区中配档](https://www.rainyun.com/MzA2MTc2_) |
| 进阶版 | 4核 / 4GB / 30GB | 1TB · 1Gbps | ¥63 | [香港五区进阶档](https://www.rainyun.com/MzA2MTc2_) |
| 高级版 | 4核 / 8GB / 30GB | 1TB · 1Gbps | ¥103 | [香港五区高级档](https://www.rainyun.com/MzA2MTc2_) |
| 旗舰版 | 8核 / 16GB / 30GB | 1TB · 1Gbps | ¥183 | [香港五区旗舰档](https://www.rainyun.com/MzA2MTc2_) |

**日本东京（EPYC 7K62 · 软银大带宽）**

| 套餐 | 配置 | 流量 · 带宽 | 月付 | 购买 |
| --- | --- | --- | --- | --- |
| 入门版 | 1核 / 1GB / 30GB | 200GB · 50Mbps 双向 | ¥25 | [日本入门版](https://www.rainyun.com/MzA2MTc2_) |
| 标配版 | 2核 / 2GB / 30GB | 500GB · 100Mbps 双向 | ¥32 | [日本标配版](https://www.rainyun.com/MzA2MTc2_) |
| 中配版 | 2核 / 4GB / 30GB | 500GB · 100Mbps 双向 | ¥42 | [日本中配版](https://www.rainyun.com/MzA2MTc2_) |
| 进阶版 | 4核 / 4GB / 30GB | 500GB · 100M↑200M↓ | ¥64 | [日本进阶版](https://www.rainyun.com/MzA2MTc2_) |
| 高配版 | 4核 / 8GB / 30GB | 500GB · 150M↑200M↓ | ¥128 | [日本高配版](https://www.rainyun.com/MzA2MTc2_) |
| 高配版+ | 8核 / 8GB / 30GB | 500GB · 200Mbps 双向 | ¥168 | [日本高配版+](https://www.rainyun.com/MzA2MTc2_) |
| 旗舰版 | 8核 / 16GB / 30GB | 1TB · 300Mbps 双向 | ¥192 | [日本旗舰版](https://www.rainyun.com/MzA2MTc2_) |

日本二区另有软银大带宽方案，2核4G / 30GB / 500GB 流量 / 100Mbps 标价 ¥47/月。

**韩国首尔一区（AMD EPYC 7A23 · 不含 IP 费用）**

| 套餐 | 配置 | 流量 · 带宽 | 月付 | 购买 |
| --- | --- | --- | --- | --- |
| 标配版 | 2核 / 2GB / 40GB | 30GB · 20M↑40M↓ | ¥28 | [韩国标配版](https://www.rainyun.com/MzA2MTc2_) |
| 标配版+ | 2核 / 2GB / 40GB | 50GB · 30M↑50M↓ | ¥36 | [韩国标配版+](https://www.rainyun.com/MzA2MTc2_) |
| 中配版 | 2核 / 4GB / 40GB | 50GB · 40M↑80M↓ | ¥45 | [韩国中配版](https://www.rainyun.com/MzA2MTc2_) |
| 高配版 | 4核 / 4GB / 40GB | 80GB · 50M↑100M↓ | ¥72 | [韩国高配版](https://www.rainyun.com/MzA2MTc2_) |
| 顶配版 | 4核 / 8GB / 40GB | 80GB · 75M↑100M↓ | ¥150 | [韩国顶配版](https://www.rainyun.com/MzA2MTc2_) |
| 顶配版+ | 8核 / 8GB / 40GB | 100GB · 100Mbps 双向 | ¥188 | [韩国顶配版+](https://www.rainyun.com/MzA2MTc2_) |
| 旗舰版 | 8核 / 16GB / 40GB | 150GB · 100Mbps 双向 | ¥220 | [韩国旗舰版](https://www.rainyun.com/MzA2MTc2_) |

韩国这套支持原生 IP 和住宅 IP，普通 IP 5 元/个/月，原生 IP 10 元/个/月，住宅 IP 25 元/个/月。

## 国内区域套餐对比

**湖北十堰（三线 BGP · Platinum 8272CL/8171M · 限流量 · 200G 防御）**

| 套餐 | 配置 | 流量 · 带宽 | 月付 | 购买 |
| --- | --- | --- | --- | --- |
| 标配版 | 2核 / 2GB / 30GB | 200GB · 50M 双向 | ¥10 | [十堰标配版](https://www.rainyun.com/MzA2MTc2_) |
| 中配版 | 2核 / 4GB / 30GB | 300GB · 50M 双向 | ¥19 | [十堰中配版](https://www.rainyun.com/MzA2MTc2_) |
| 进阶版 | 4核 / 4GB / 30GB | 500GB · 50M↑100M↓ | ¥30 | [十堰进阶版](https://www.rainyun.com/MzA2MTc2_) |
| 高配版 | 4核 / 8GB / 30GB | 500GB · 50M↑100M↓ | ¥58 | [十堰高配版](https://www.rainyun.com/MzA2MTc2_) |
| 顶配版 | 8核 / 8GB / 30GB | 500GB · 100M 双向 | ¥90 | [十堰顶配版](https://www.rainyun.com/MzA2MTc2_) |
| 旗舰版 | 8核 / 16GB / 30GB | 1TB · 100M 双向 | ¥128 | [十堰旗舰版](https://www.rainyun.com/MzA2MTc2_) |
| 企业版 | 12核 / 24GB / 30GB | 1TB · 100M 双向 | ¥288 | [十堰企业版](https://www.rainyun.com/MzA2MTc2_) |

**湖北襄阳（Platinum · 限流量 · 200G 防御）**

| 套餐 | 配置 | 流量 · 带宽 | 月付 | 购买 |
| --- | --- | --- | --- | --- |
| 标配版 | 2核 / 2GB / 30GB | 500GB · 50M↑100M↓ | ¥50 | [襄阳标配版](https://www.rainyun.com/MzA2MTc2_) |
| 中配版 | 2核 / 4GB / 30GB | 500GB · 50M↑100M↓ | ¥58 | [襄阳中配版](https://www.rainyun.com/MzA2MTc2_) |
| 进阶版 | 4核 / 4GB / 30GB | 500GB · 100M 双向 | ¥72 | [襄阳进阶版](https://www.rainyun.com/MzA2MTc2_) |
| 高配版 | 4核 / 8GB / 30GB | 500GB · 100M↑200M↓ | ¥98 | [襄阳高配版](https://www.rainyun.com/MzA2MTc2_) |
| 高配版+ | 8核 / 8GB / 30GB | 500GB · 100M↑200M↓ | ¥130 | [襄阳高配版+](https://www.rainyun.com/MzA2MTc2_) |
| 旗舰版 | 8核 / 16GB / 30GB | 1TB · 200M↑500M↓ | ¥168 | [襄阳旗舰版](https://www.rainyun.com/MzA2MTc2_) |
| 企业版 | 12核 / 24GB / 30GB | 1TB · 200M↑500M↓ | ¥298 | [襄阳企业版](https://www.rainyun.com/MzA2MTc2_) |

**广东深圳（Platinum 8259CL · 限流量 · 20G+ 防御）**

| 套餐 | 配置 | 流量 · 带宽 | 月付 | 购买 |
| --- | --- | --- | --- | --- |
| 标配版 | 2核 / 2GB / 30GB | 300GB · 50M↑100M↓ | ¥48 | [深圳标配版](https://www.rainyun.com/MzA2MTc2_) |
| 中配版 | 2核 / 4GB / 30GB | 300GB · 50M↑100M↓ | ¥58 | [深圳中配版](https://www.rainyun.com/MzA2MTc2_) |
| 进阶版 | 4核 / 4GB / 30GB | 300GB · 100M 双向 | ¥72 | [深圳进阶版](https://www.rainyun.com/MzA2MTc2_) |
| 高配版 | 4核 / 8GB / 30GB | 500GB · 100M↑200M↓ | ¥98 | [深圳高配版](https://www.rainyun.com/MzA2MTc2_) |
| 高配版+ | 8核 / 8GB / 30GB | 500GB · 100M↑200M↓ | ¥130 | [深圳高配版+](https://www.rainyun.com/MzA2MTc2_) |
| 旗舰版 | 8核 / 16GB / 30GB | 1TB · 200M 双向 | ¥168 | [深圳旗舰版](https://www.rainyun.com/MzA2MTc2_) |
| 企业版 | 12核 / 24GB / 30GB | 1TB · 200M 双向 | ¥298 | [深圳企业版](https://www.rainyun.com/MzA2MTc2_) |

**广东广州番禺（Platinum · 不限流量 · 200Mbps · 阿里 BGP 150G 防御）**

| 套餐 | 配置 | 月付 | 购买 |
| --- | --- | --- | --- |
| 标配版 | 2核 / 2GB / 30GB | ¥80 | [番禺标配版](https://www.rainyun.com/MzA2MTc2_) |
| 中配版 | 2核 / 4GB / 30GB | ¥88 | [番禺中配版](https://www.rainyun.com/MzA2MTc2_) |
| 进阶版 | 4核 / 4GB / 30GB | ¥98 | [番禺进阶版](https://www.rainyun.com/MzA2MTc2_) |
| 高配版 | 4核 / 8GB / 30GB | ¥115 | [番禺高配版](https://www.rainyun.com/MzA2MTc2_) |
| 顶配版 | 8核 / 8GB / 30GB | ¥168 | [番禺顶配版](https://www.rainyun.com/MzA2MTc2_) |
| 旗舰版 | 8核 / 16GB / 30GB | ¥208 | [番禺旗舰版](https://www.rainyun.com/MzA2MTc2_) |
| 企业版 | 12核 / 24GB / 30GB | ¥355 | [番禺企业版](https://www.rainyun.com/MzA2MTc2_) |

**江苏宿迁（E5v3 · 不限流量 · NAT 共享端口）**

| 套餐 | 配置 | 带宽 | 月付 | 购买 |
| --- | --- | --- | --- | --- |
| 入门版 | 1核 / 1GB / 30GB | 10Mbps | ¥14 | [宿迁入门版](https://www.rainyun.com/MzA2MTc2_) |
| 标配版 | 2核 / 2GB / 30GB | 15Mbps | ¥22 | [宿迁标配版](https://www.rainyun.com/MzA2MTc2_) |
| 高配版 | 4核 / 4GB / 30GB | 25Mbps | ¥42 | [宿迁高配版](https://www.rainyun.com/MzA2MTc2_) |
| 顶配版 | 4核 / 8GB / 30GB | 35Mbps | ¥80 | [宿迁顶配版](https://www.rainyun.com/MzA2MTc2_) |
| 旗舰版 | 8核 / 16GB / 30GB | 35Mbps | ¥138 | [宿迁旗舰版](https://www.rainyun.com/MzA2MTc2_) |

宿迁这套是 NAT 共享端口模式，只能手动映射端口对外访问，独立 IPv4 需要额外 150 元。拿它跑容器做内网穿透、代码调试、自动化脚本都没问题，但别指望它直接对外开 80/443 建站。

**浙江温州（100G 高防大带宽）**：2核2G / 100Mbps / 100G 高防，年付 7 折 ¥823.2/年（约 ¥68.6/月）；4核8G / 100Mbps / 100G 高防，年付 7 折 ¥1243.2/年（约 ¥103.6/月）。这个区域另有不限流量版本，带宽降到 25Mbps 上行，月付便宜约 20 元。👉 [查看温州高防配置](https://www.rainyun.com/MzA2MTc2_)

## 游戏云和显卡云：什么时候才需要考虑

大部分 Docker 场景用不上这两条线，但有两类例外值得知道。

**游戏云**用的是单核高频 CPU（i9-14900K、Ryzen 7950X 这类），适合对单核敏感、又不需要多核吞吐的容器负载。雨云论坛的选购帖给的单核排序是 i9-14900K > Ryzen 7950X > Ryzen 5900X，E5 系列单核较弱，高版本应用慎选。游戏云起价按 CPU 和配置区分：

| CPU 平台 | 起始配置 | 带宽 | 起价/月 | 购买 |
| --- | --- | --- | --- | --- |
| E5 2666v3 | 2核 / 4GB | 10M | ¥65 | [游戏云 E5 方案](https://www.rainyun.com/MzA2MTc2_) |
| Xeon Gold 6146 | 2核 / 4GB | 10M | ¥80 | [游戏云 Gold 6146](https://www.rainyun.com/MzA2MTc2_) |
| Ryzen 5900X | 1核 / 2GB | 5M | ¥125 | [游戏云 5900X](https://www.rainyun.com/MzA2MTc2_) |
| Ryzen 7950X | 1核 / 2GB | 5M | ¥139 | [游戏云 7950X](https://www.rainyun.com/MzA2MTc2_) |
| i9 13900KF | 1核 / 2GB | 5M | ¥145 | [游戏云 i9 13900KF](https://www.rainyun.com/MzA2MTc2_) |
| i9 14900KF | 1核 / 2GB | 5M | ¥145 | [游戏云 i9 14900KF](https://www.rainyun.com/MzA2MTc2_) |

**显卡云**适合要跑 CUDA 的容器——本地大模型、语音推理、渲染这类。宁波用的是 Tesla P40，深圳已经换成 Tesla V100（独立显卡，支持 CUDA）：

| 区域 · 方案 | 配置 | 带宽 | 起价/月 | 购买 |
| --- | --- | --- | --- | --- |
| 宁波 P40 挂机入门版 | 2核 / 4GB / 1GB 显存 | 10M↑50M↓ | ¥40 | [P40 入门方案](https://www.rainyun.com/MzA2MTc2_) |
| 宁波 P40 挂机标配版 | 4核 / 4GB / 1GB 显存 | 15M↑50M↓ | ¥48 | [P40 标配方案](https://www.rainyun.com/MzA2MTc2_) |
| 宁波 P40 小型工作 | 8核 / 8GB / 2GB 显存 | 25M↑100M↓ | ¥68 | [P40 小型工作方案](https://www.rainyun.com/MzA2MTc2_) |
| 宁波 P40 小型工作+ | 8核 / 8GB / 4GB 显存 | 25M↑100M↓ | ¥135 | [P40 工作+ 方案](https://www.rainyun.com/MzA2MTc2_) |
| 宁波 P40 高配 | 8核 / 16GB / 4GB 显存 | 25M↑150M↓ | ¥168 | [P40 高配方案](https://www.rainyun.com/MzA2MTc2_) |
| 宁波 P40 旗舰 | 16核 / 16GB / 8GB 显存 | 35M↑150M↓ | ¥258 | [P40 旗舰方案](https://www.rainyun.com/MzA2MTc2_) |
| 深圳 V100 日常网吧型 | 16核 / 24GB / 16GB 显存 | 50M↑200M↓ | ¥28（每月赠 5 小时，加时 2.5 元/小时） | [V100 网吧型](https://www.rainyun.com/MzA2MTc2_) |
| 深圳 V100 算力长租型 | 16核 / 24GB / 16GB 显存 | 50M↑200M↓ | ¥638（不限时长） | [V100 长租型](https://www.rainyun.com/MzA2MTc2_) |

宁波显卡云默认已装 NVIDIA 驱动并启用 CUDA，系统可选 Ubuntu 22.04 LTS 或 Windows 2022。深圳 V100 的网吧型按小时计费，关机不计时，适合偶尔跑一次推理的人；真的长期占用算力，长租型反而更便宜。

再往上就是裸金属物理机，比如宁波 R7-8745HS（8核16线程 / 32G / 500GB / 500M / 100G 高防，标价 ¥798/月），适合要独占资源和自定义配置的场景。这个量级的负载已经超出「跑 Docker」的讨论范围了。

## 按预算选：三套能直接抄的组合

**预算卡在 20 元以内，先跑通再说**
湖北十堰标配版，2核2G / 30GB / 200GB 流量，¥10/月。装 Debian 12 + 预装 Docker 环境，跑一两个容器完全够。缺点是流量限 500GB 以内的小档位（这套是 200GB），用完限速到 5Mbps，需要的话按 300G 20 元、1024G 50 元补流量包。

**免备案 + 要稳定对外提供服务**
香港五区标准版，2核2G / 1TB 流量 / 500Mbps，¥30/月。流量给得足，带宽高，拉镜像和对外响应都快；不需要走备案流程。要跑数据库就加一档到中配版 2核4G ¥43/月。

**正经跑一套 Compose 或单节点 K3s**
美国进阶版 4核4G ¥60/月，或者湖北十堰进阶版 4核4G / 500GB 流量 ¥30/月（国内访问快，但建站要备案）。8GB 档位的话，美国高配版 ¥113/月，十堰顶配版 8核8G ¥90/月。

## 优惠、试用和退款怎么用

这几条都是能在下单页验证的：

- 通过优惠入口注册账号并绑定微信，会赠送一张**首月 5 折优惠券**，没有额外限制条件
- **年付 7 折**，可以和上面那张券组合，实际支出会比按月付低不少
- 云服务器和游戏云产品支持**1 元试用 1 天**（每款限 1 次，配置越高试用价越高，部分产品 20 元/天）
- 大部分产品支持**7 天内无理由退款**（每个账号限 3 次）

先用 1 元试用跑一遍你的 compose，把端口、反代、数据卷都验证过，再决定买哪档配置，这个顺序比直接年付划算得多。

👉 [从优惠入口注册，领首月 5 折券](https://www.rainyun.com/MzA2MTc2_)

## 下单前容易踩的几个坑

**流量是双向算的。** 流量计费模式下上行和下行都计，容器之间或者对外提供文件服务时容易低估。看不清某个套餐是限流量还是不限流量，看购买页有没有「**G」字样——没有就是不限流量。

**大陆区域建站要备案。** 十堰、襄阳、深圳、番禺、温州这些国内机房，域名和常用端口的 HTTP 业务要先备案再过白，不备案就只能用 IP 加非标端口访问。

**IP 是单独计费的。** 多数区域默认含 1 个 IPv4（5 元/个/月），韩国是明确不含的。高防 IP 价格差得挺大：香港移动高防 IP 40 元/月，日本高防 IP 30 元/月，十堰三线 BGP 高防 IP 200 元/月。

**硬盘扩容价各区域不同。** 十堰、襄阳、深圳是高速固态 0.4 元/GB/月（上限 250GB）、机械 0.1 元/GB/月（上限 2TB）；韩国是 SSD 0.3 元/GB/月（上限 200GB）；香港五区按 0.12 元/10GB/天 计，折合约 3.6 元/10GB/月。

**重装系统会清空数据。** Docker 默认把数据存在本地磁盘，重装或换系统镜像前先把 volume 和目录备份出去。

## 常见问题

**2核2G 能跑几个容器？**
两到四个轻量服务（反代 + 一个小应用 + 一个 SQLite 类数据库）比较现实。跑 MySQL 或 Postgres 建议直接上 4GB。

**1GB 内存的套餐值得买吗？**
只放一两个极轻量容器、且不跑数据库的话可以。拿它当生产环境省下的钱，会被排查 OOM 的时间吃掉。

**国内机器拉不动镜像怎么办？**
改 `/etc/docker/daemon.json` 加 `registry-mirrors`，或者下单时直接选预装的 Docker 环境镜像，省掉安装这步。

**NAT 机型和独立 IP 机型差在哪？**
NAT 共用公网 IP、靠端口映射对外，宿迁那套就是；独立 IP 能直接暴露端口、跑网站。选哪种取决于你的容器要不要对外提供服务。

**先买一个月还是一年？**
不确定配置就按月付，或者先 1 元试用。确认配置合适了再用年付 7 折，这是最省事也最省钱的做法。
