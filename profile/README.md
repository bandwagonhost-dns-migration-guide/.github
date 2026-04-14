
你有没有遇到过这种情况：搬瓦工 VPS 买回来，跑个 curl 或者 wget 慢得要命，ping 谷歌延迟也比预期高出一截——明明买的是 CN2 GIA 高端线路，却总感觉差点意思？

绝大多数人第一反应是检查路由、换机房、换套餐。但很可能真正的问题就藏在一个你从来没注意过的地方：**你的 VPS 还在用搬瓦工默认的 172.31 开头的 DNS 服务器。**

这篇文章就来专门聊聊**搬瓦工 DNS 服务器**这件事：它默认是什么、为什么会拖慢你的速度、怎么一步步迁移到更好的 DNS，以及迁移之后还有哪些坑要注意。

---

## 一、为什么你要考虑换掉搬瓦工自带 DNS

搬瓦工（BandwagonHost）所有服务器开箱预装的 DNS 服务器，通常是一个 `172.31` 开头的内网地址。最初设置这个 DNS 的原因是：搬瓦工官方通过自家 DNS 为用户提供 ChatGPT、Netflix 等流媒体的解锁服务。

听起来不错，但问题是：

**这个 DNS 本身解析速度并不快。** 用户实测在解析谷歌、YouTube、GitHub 等常见域名时，这套官方 DNS 往往比 8.8.8.8 或 1.1.1.1 多出 40~60ms 的额外延迟。本来美西机房到国内就有 150ms 左右的基础延迟，再加上 DNS 解析阶段额外损耗，体感差距相当明显。

而随着搬瓦工已经在多个机房实现了 ChatGPT、Netflix 等的原生 IP 解锁，默认 DNS 的流媒体解锁价值大幅下降，但 DNS 解析慢的问题却依然存在。

所以，换掉默认 DNS 服务器这件事，现在的性价比其实相当高。

---

## 二、迁移前需要准备什么

在动手之前，先把几件事情想清楚：

**1. 你的系统是什么？**

目前搬瓦工支持 Debian、Ubuntu、CentOS 等多种 Linux 发行版，KiwiVM 面板已支持一键安装 Ubuntu 24.04、Debian 13 等最新版本。不同系统修改 DNS 的方式略有不同，尤其是 Ubuntu 20.04+ 引入了 `systemd-resolved`，处理方式和旧版 Debian 有差异。

**2. 你打算换到哪个 DNS？**

常见选择有三个：
- **Google DNS**：`8.8.8.8` / `8.8.4.4`，稳定、通用
- **Cloudflare DNS**：`1.1.1.1` / `1.0.0.1`，速度通常更快
- **114 DNS**（如果你有国内解析需求）：`114.114.114.114`

搬瓦工 VPS 机房在美西、日本、香港等地，推荐优先使用 Cloudflare 的 `1.1.1.1`，实测解析延迟更低。

**3. 你想要临时修改还是永久生效？**

这是坑最多的地方，后面详细说。

---

## 三、分步迁移教程

### 第一步：确认当前 DNS 配置

SSH 登录搬瓦工 VPS 后，运行：

bash
cat /etc/resolv.conf


如果你看到类似这样的输出：


nameserver 172.31.0.2
nameserver 172.31.0.1


那就是搬瓦工默认的 DNS。接下来要做的事就是把它们替换掉。

---

### 第二步：判断你的 DNS 管理方式

直接修改 `/etc/resolv.conf` 是最直觉的做法，但在很多系统上重启后会被还原——因为这个文件可能只是一个软链接，由 `systemd-resolved`、`NetworkManager` 或 `resolvconf` 包动态生成。

先确认一下它是不是软链接：

bash
ls -la /etc/resolv.conf


如果输出里有 `->` 指向其他路径，说明它被系统接管了，直接编辑无效。

---

### 第三步：永久修改 DNS（针对不同系统）

**方案 A：直接锁定 resolv.conf（简单粗暴，适用于 CentOS）**

bash
# 先删除软链接或备份旧文件
mv /etc/resolv.conf /etc/resolv.conf.bak

# 创建新的真实文件
echo "nameserver 1.1.1.1" > /etc/resolv.conf
echo "nameserver 8.8.8.8" >> /etc/resolv.conf

# 加锁，防止被系统覆盖
chattr +i /etc/resolv.conf


如果以后需要修改，先解锁：

bash
chattr -i /etc/resolv.conf


**方案 B：通过 resolvconf 管理（适用于 Debian/Ubuntu）**

bash
nano /etc/resolvconf/resolv.conf.d/base


在文件中写入：


nameserver 1.1.1.1
nameserver 8.8.8.8


保存后执行：

bash
resolvconf -u


**方案 C：卸载 resolvconf 再直接编辑（Debian/Ubuntu 的彻底方案）**

bash
apt remove resolvconf
reboot


重启后 `/etc/resolv.conf` 不会再被自动覆盖，可以直接编辑保存。

**方案 D：通过网卡配置文件指定 DNS（最彻底的方案）**

如果上面几种方案都失效，说明 DNS 被网卡配置接管。编辑网卡配置：

bash
nano /etc/network/interfaces.d/50-cloud-init


在对应网卡配置块中加入或修改 `dns-nameservers` 行，然后重启网络服务。

---

### 第四步：验证修改是否生效

bash
cat /etc/resolv.conf


确认 `nameserver` 已经变成你设置的地址，然后测试解析速度：

bash
nslookup google.com
# 或者
dig google.com


也可以直接对比修改前后的响应时间：

bash
time nslookup google.com


正常情况下，换成 1.1.1.1 之后，解析时间应该比原来的 172.31 DNS 快 30~60ms。

---

### 第五步：关于域名 DNS 服务器的迁移（建站用户）

如果你是在搬瓦工 VPS 上建站，还有一个"域名 DNS"的概念需要区分——这和 VPS 系统内的 resolv.conf 完全是两件事。

域名 DNS 服务器是告诉全球 DNS 体系"这个域名交给谁来解析"的设置，需要在你的**域名注册商**后台修改。常见选择是把域名 NS 记录改成 Cloudflare 或 DNSPod 提供的 DNS 服务器地址，再在对应平台上添加 A 记录指向你的搬瓦工 VPS IP。

具体步骤：
1. 在 DNSPod 或 Cloudflare 注册账号，添加你的域名
2. 记下平台分配给你的 DNS 服务器地址（每个账号不同）
3. 前往域名注册商（阿里云、Dynadot 等）修改 DNS 服务器为上述地址
4. 在 DNSPod/Cloudflare 添加 A 记录，值填写你的搬瓦工 VPS IP
5. 等待 DNS 生效（通常几分钟到几小时）

搬瓦工的 KiwiVM 控制面板还提供了**反向 DNS（PTR Records）**设置功能，可以将你的 VPS IP 绑定到一个域名——这主要用于自建邮件服务器场景，防止发出去的邮件被标记为垃圾邮件。

---

## 四、迁移过程中常见问题处理

**问题一：修改完重启还原了**

99% 的情况是 resolv.conf 是软链接，或者系统有 resolvconf / systemd-resolved 接管。按照第三步的方案 C 或 D 处理。

**问题二：chattr +i 命令报错**

可能是安装了 resolvconf 包导致 chattr 命令无法对该文件生效。先卸载 resolvconf，再执行 chattr。

**问题三：DNS 修改后某些服务连不上了**

检查是不是防火墙规则拦截了 DNS 请求（UDP 53 端口）。搬瓦工 VPS 默认不开防火墙，但如果你自己配置过 iptables，需要确认放行 DNS 流量。

**问题四：迁移机房后 VPS IP 变了**

如果你用搬瓦工 KiwiVM 面板一键迁移机房，IP 地址会改变。这时候需要回到 DNS 服务商（Cloudflare/DNSPod）更新 A 记录，把旧 IP 换成新 IP，其他配置无需重新设置。

---

## 五、搬瓦工主要套餐对比（按线路分类）

不管你是用搬瓦工建站、跑服务，还是部署 AI 应用，选对套餐和 DNS 配合，体验才能最大化。以下是搬瓦工目前在售的主要常规套餐（优惠码 `BWHCGLUKKB` 可享约 6.58% 折扣）：

| 系列 | 内存 | 硬盘 | 月流量 | 带宽 | 价格 | 可选机房 | 购买链接 |
|------|------|------|--------|------|------|----------|----------|
| KVM 入门套餐 | 1GB | 20GB SSD | 1TB | 1Gbps | $49.99/年 | 洛杉矶 DC2/DC4/DC8、弗里蒙特、新泽西、纽约、荷兰等9个 |  [立即购买](https://bwh81.net/aff.php?aff=80238&pid=57) |
| KVM 入门套餐（大流量） | 1GB | 20GB SSD | 2TB | 1Gbps | $52.99/半年 $99.99/年 | 同上 |  [立即购买](https://bwh81.net/aff.php?aff=80238&pid=58) |
| CN2 GIA-E 套餐（标准） | 1GB | 20GB SSD | 1TB | 2.5Gbps | $49.99/季 $169.99/年 | DC6/DC9/日本软银/荷兰9929等11+机房 |  [立即购买](https://bwh81.net/aff.php?aff=80238&pid=87) |
| CN2 GIA-E 套餐（大流量） | 1GB | 20GB SSD | 2TB | 2.5Gbps | $89.99/季 $299.99/年 | 同上 |  [立即购买](https://bwh81.net/aff.php?aff=80238&pid=88) |
| 香港 CN2 GIA（标准） | 2GB | 40GB SSD | 500GB | 1Gbps | $89.99/月 $899.99/年 | 香港 CN2 GIA 机房 |  [立即购买](https://bwh81.net/aff.php?aff=80238&pid=95) |
| 香港 CN2 GIA（大流量） | 4GB | 80GB SSD | 1TB | 1Gbps | $155.99/月 $1559.99/年 | 香港 CN2 GIA 机房 |  [立即购买](https://bwh81.net/aff.php?aff=80238&pid=96) |

**各系列简评：**

**KVM 套餐**：价格最实惠，年付 $49.99 起，适合新手练手、学 Linux、跑轻量脚本。走普通国际线路，国内速度一般，不特别推荐用来建面向国内的网站，但胜在便宜。

**CN2 GIA-E 套餐**：搬瓦工的主力性价比产品，三网 CN2 GIA 高端线路，2.5Gbps 大带宽，多达 11 个机房可自由切换，包括洛杉矶 DC6/DC9、日本软银、荷兰 9929 等，季付 $49.99，性价比最高，👉 [点这里查看最新优惠](https://bwh81.net/aff.php?aff=80238)。

**香港 CN2 GIA 套餐**：延迟最低（对华南用户尤其明显），线路顶级，价格也顶级，月付 $89.99 起，适合对延迟有极致要求或公费报销的用户。

---

## 六、迁移后还能做什么

DNS 换完之后，顺手可以做几件事让 VPS 体验更好：

**开启 BBR 加速**：一行命令开启 TCP BBR 拥塞控制，能进一步改善网络吞吐量，在 CN2 GIA 线路上效果尤其明显。

**设置反向 DNS（PTR）**：如果你要自建邮件服务器，去 KiwiVM 面板首页找到 PTR Records，设置好 IP 对应的域名，能显著降低邮件被标记为垃圾的概率。

**定期备份 resolv.conf**：在做完 DNS 修改并锁定之后，记录一份配置存档，避免以后重装系统或迁移机房后重复踩坑。

**监控 DNS 解析是否恢复默认**：偶尔 `cat /etc/resolv.conf` 确认一下，如果发现又变回 172.31，说明某次系统更新解除了 chattr 锁定。

---

## 七、总结

搬瓦工 DNS 服务器的问题，说复杂不复杂，说简单也不简单。核心逻辑就是：**默认 172.31 DNS 解析慢，换成 1.1.1.1 或 8.8.8.8 之后，能节省 40~60ms 的 DNS 解析延迟，整体体验明显提升。**

迁移步骤的关键点只有一个：**弄清楚你的系统是否有软件在接管 resolv.conf**，然后选对应的方案处理。大多数搬瓦工用户用 Debian/Ubuntu，卸载 resolvconf 再直接写文件是最干净的做法。

如果你还没有搬瓦工 VPS，正好借这个机会看看目前在售套餐。对于大多数人来说，👉 [CN2 GIA-E 套餐](https://bwh81.net/aff.php?aff=80238)（季付 $49.99 起）是性价比最高的选择，三网 CN2 GIA 优质线路，配合换完 DNS 之后的速度提升，体验会比你想象的要好很多。
