# 运维脚本面试题 — Linux / Bash / Python

> 适用：云原生开发 / 运维开发 / SRE  
> 关联：[feishu-stability-interview-qa.md](./feishu-stability-interview-qa.md)

---

## 一、Linux 命令基础（Q1–Q20）

### Q1. 如何查看 CPU、内存占用最高的进程？

**答**：

```bash
top          # 交互式，按 P 按 CPU、M 按内存
htop         # 更友好（需安装）
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10
```

---

### Q2. 如何查看磁盘空间和 inode 使用情况？

**答**：

```bash
df -h              # 磁盘空间
df -i              # inode（小文件过多会耗尽 inode）
du -sh /var/log/*  # 目录占用
ncdu /             # 交互式（需安装）
```

**场景**：磁盘 100% 但找不到大文件 → 查 **inode**、**已删除但未释放的 open 文件**（`lsof +L1`）。

---

### Q3. 如何查看网络连接和监听端口？

**答**：

```bash
ss -tunlp          # 推荐，比 netstat 快
ss -s              # 连接统计摘要
lsof -i :8080      # 谁占用 8080
netstat -tunlp     # 老系统
```

---

### Q4. 如何测试端口/HTTP 是否可达？

**答**：

```bash
curl -v http://host:8080/health
curl -o /dev/null -s -w "%{http_code} %{time_total}\n" URL
nc -zv host 6443
telnet host 22
ping -c 4 host
mtr host           # 路径与丢包
```

---

### Q5. 如何实时跟踪日志？

**答**：

```bash
tail -f /var/log/messages
journalctl -u kubelet -f --since "10 min ago"
kubectl logs -f pod/xxx -n kube-system
grep -E "ERROR|WARN" app.log | tail -100
```

---

### Q6. 如何查找包含某字符串的文件？

**答**：

```bash
# 按内容搜（grep）
grep -rn "pattern" /etc/
grep -r --include="*.yaml" "image:" .

# 按文件属性搜（find）
find /var/log -name "*.log" -mtime -1    # 24h 内修改
find . -type f -size +100M               # 大于 100M

# 组合：先 find 再 grep
find . -maxdepth 1 -name "*.log" -exec grep -l "Error" {} +
```

> 更多 grep / find 选项与组合见 **[一点五、常用命令详解](#一点五常用命令详解grep--find--pgrep--awk--sed--cut--sort--uniq)**（A. grep、F. find）。

---

### Q7. awk 和 sed 常见用法？

**答**：

```bash
# awk：按列处理
ps aux | awk '$3>50.0 {print $2, $11}'   # CPU>50% 的 PID 和命令
cat metrics.txt | awk '{sum+=$1} END {print sum/NR}'  # 平均值

# sed：替换
sed -i 's/enforcing/disabled/g' /etc/selinux/config
sed -n '10,20p' file.txt   # 打印 10-20 行
```

> 完整 awk/sed/cut/sort/uniq 见 **一点五**。

---

### Q8. 如何查看系统负载和 CPU 核心数？

**答**：

```bash
uptime                    # load average
nproc                     # CPU 逻辑核数
lscpu
cat /proc/loadavg
```

**理解**：load 5 在 4 核机器上 ≈ 满载；在 16 核上仍有余量。

---

### Q9. 文件权限 rwx 和 chmod 755 含义？

**答**：

- `r=4, w=2, x=1`；755 = `rwxr-xr-x`（owner 全权限，其他只读执行）
- 目录的 `x` 表示能否 **cd 进入**
- `chmod +x script.sh`；`chown user:group file`

---

### Q10. 软链接和硬链接区别？


|       | 硬链接      | 软链接          |
| ----- | -------- | ------------ |
| inode | 同一 inode | 新 inode，指向路径 |
| 跨分区   | 否        | 是            |
| 源删    | 仍可用      | 失效           |


```bash
ln file hardlink
ln -s /path/to/target symlink
```

---

### Q11. 如何查某进程打开的文件？

**答**：

```bash
lsof -p <pid>
lsof -u root
ls -l /proc/<pid>/fd/
```

**场景**：日志删了仍占磁盘 → `lsof +L1` 找到进程 reload。

---

### Q12. 环境变量和 PATH？

**答**：

```bash
echo $PATH
export KUBECONFIG=/etc/kubernetes/admin.conf
env | grep KUBE
source ~/.bashrc
```

---

### Q13. 如何批量杀进程？（谨慎）

**答**：

```bash
pgrep -f "python app.py"
pkill -f "python app.py"
kill -15 <pid>    # SIGTERM 优雅
kill -9 <pid>     # SIGKILL 强杀（最后手段）
```

---

### Q14. 定时任务 cron 格式？

**答**：

```
分 时 日 月 周  命令
0  2  *  *  *   /opt/backup.sh    # 每天 2:00
*/5 * *  *  *   /opt/check.sh     # 每 5 分钟
```

查看：`crontab -l`；日志：`/var/log/cron` 或 `journalctl`。

---

### Q15. 如何压缩和解压？

**答**：

```bash
tar -czvf archive.tar.gz dir/
tar -xzvf archive.tar.gz
zip -r archive.zip dir/
```

---

### Q16. 如何查看内核/系统信息？

**答**：

```bash
uname -a
cat /etc/os-release
hostnamectl
free -h
vmstat 1
iostat -x 1
```

---

### Q17. 排查「磁盘 IO 高」？

**答**：

```bash
iostat -x 1
iotop
pidstat -d 1
```

找 **高 %util、高 await** 的磁盘和高读写进程。

---

### Q18. 排查「内存不足 / OOM」？

**答**：

```bash
free -h
dmesg | grep -i oom
journalctl -k | grep -i "out of memory"
cat /proc/meminfo
```

K8s：`kubectl describe node` 看 MemoryPressure；Pod OOMKilled → `kubectl describe pod`。

---

### Q19. 如何临时和永久修改内核参数 sysctl？

**答**：

```bash
sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.d/k8s.conf
sysctl --system
```

K8s 常见：`net.bridge.bridge-nf-call-iptables=1`。

---

### Q20. 场景题：服务突然不可用，Linux 层面排查顺序？

**答**：

```
1. 服务进程是否存活？ systemctl status / ps
2. 端口是否监听？ ss -tunlp
3. 本地 curl 127.0.0.1 是否正常？
4. 防火墙/iptables/nftables？
5. 磁盘满？df -h；内存？free
6. 最近变更？/var/log 、journalctl
7. 网络：DNS、路由、到依赖方连通性
8. strace/tcpdump 深入（最后手段）
```

---

## 一点五、常用命令详解：grep / find / pgrep / awk / sed / cut / sort / uniq

> 运维排障与脚本 **最高频** 组合；面试常考「日志分析 + 文件查找 + 进程管理 + 列提取」。

### 命令速查表


| 命令        | 用途             | 典型场景                  |
| --------- | -------------- | --------------------- |
| **grep**  | 按 **行** 匹配文本   | 日志搜 ERROR、配置查关键字      |
| **find**  | 按 **条件** 遍历目录树 | 找大文件、过期日志、批量处理        |
| **pgrep** | 按名查 **PID**    | 脚本里判断进程是否存在           |
| **pkill** | 按名发信号杀进程       | 批量停服务（慎用 -9）          |
| **awk**   | 按 **列** 处理文本   | ps/kubectl 输出取字段、统计   |
| **sed**   | 流编辑器           | 替换、删行、取行号范围           |
| **cut**   | 按分隔符切列         | 简单 CSV/空格列（不如 awk 灵活） |
| **sort**  | 排序             | 配合 uniq 做 TopN        |
| **uniq**  | 去重（相邻）         | 必须 **先 sort**         |
| **wc**    | 计数             | 行数/字数                 |


---

### A. grep 详解

#### 常用选项


| 选项                       | 含义                            |
| ------------------------ | ----------------------------- |
| `-i`                     | 忽略大小写                         |
| `-v`                     | **反向**匹配（不包含）                 |
| `-n`                     | 显示行号                          |
| `-c`                     | 只输出 **匹配行数**                  |
| `-l`                     | 只输出 **含匹配的文件名**               |
| `-L`                     | 只输出 **不含匹配** 的文件名             |
| `-r` / `-R`              | **递归**目录                      |
| `-w`                     | 整词匹配（`Error` 不匹配 `Errorsome`） |
| `-E`                     | 扩展正则（等同 `egrep`）              |
| `-F`                     | 固定字符串（等同 `fgrep`，不解析正则）       |
| `--include="*.log"`      | 只搜某类文件                        |
| `--exclude-dir`          | 跳过目录                          |
| `-A N` / `-B N` / `-C N` | 匹配行 **后/前/前后** N 行上下文         |
| `-h`                     | 多文件时不打印文件名                    |
| `-H`                     | 总是打印文件名                       |


#### 基础示例

```bash
# 搜 ERROR 或 WARN（扩展正则）
grep -E "ERROR|WARN" /var/log/app.log

# 忽略大小写搜 error
grep -i error app.log

# 递归搜 yaml 里的 image 字段
grep -rn --include="*.yaml" "image:" ./manifests/

# 统计包含 Error 的行数（单文件）
grep -c "Error" app.log

# 列出当前目录哪些 .log 含 Error
grep -l "Error" *.log

# 看匹配上下文（排障很有用）
grep -C 3 "Panic" kubelet.log

# 排除注释行
grep -v "^#" /etc/nginx/nginx.conf
grep -v "^$" file.txt          # 去空行
```

#### 正则入门（grep -E）

```bash
grep -E "^[0-9]+\." access.log           # 以数字开头的行
grep -E "502|503|504" nginx.log          # 多个模式
grep -E "time=[0-9]{2,5}ms" app.log      # 耗时格式
grep -w "Error" app.log                  # 整词 Error
```

#### 管道组合（运维高频）

```bash
# 最近 100 条 ERROR
grep "ERROR" app.log | tail -100

# 按 IP 统计 ERROR（先 grep 再 awk）
grep "ERROR" access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# 实时跟日志并过滤
tail -f app.log | grep --line-buffered "ERROR"

# 多文件合计 Error 行数
grep -h "Error" *.log | wc -l

# find + grep：当前目录 .log 中含 Error 的文件
find . -maxdepth 1 -name "*.log" -exec grep -l "Error" {} \;
```

#### grep vs ripgrep (rg)

```bash
rg "Error" --glob "*.log"    # 更快，开源项目常用；面试说 grep 即可
```

---

### B. pgrep / pkill / pidof

#### pgrep — 按名称查 PID


| 选项        | 含义                  |
| --------- | ------------------- |
| `-f`      | 匹配 **完整命令行**（不只进程名） |
| `-x`      | **精确**匹配进程名         |
| `-u user` | 指定用户的进程             |
| `-n`      | 最新启动的一个             |
| `-o`      | 最旧启动的一个             |
| `-l`      | 同时打印进程名             |
| `-a`      | 打印 **完整命令行**        |


```bash
pgrep nginx                    # 所有名为 nginx 的 PID
pgrep -x nginx                 # 精确匹配
pgrep -f "python app.py"       # 命令行含 python app.py
pgrep -u root sshd
pgrep -la kubelet              # PID + 完整命令

# 判断进程是否存在（脚本）
if pgrep -x dockerd >/dev/null; then
  echo "dockerd running"
fi
```

#### pkill — 按名称发信号

```bash
pkill nginx                    # 默认 SIGTERM (15)
pkill -9 -f "python app.py"    # SIGKILL 强杀（最后手段）
pkill -HUP nginx               # 重载配置（很多 daemon 支持）
pkill -u testuser              # 杀某用户所有进程
```

#### pidof（补充）

```bash
pidof nginx                    # 输出 PID，多个空格分隔
```

#### pgrep vs ps | grep

```bash
# ❌ 不推荐：会匹配到 grep 自身
ps aux | grep nginx

# ✅ 推荐
pgrep -a nginx
ps aux | grep [n]ginx          # 技巧：用 [n] 避免 grep 自身
```

---

### C. awk 详解

#### 核心概念

- 按 **行** 读入，按 **字段（列）** 处理  
- 默认分隔符：**空白字符**（空格/Tab）  
- `**$0`**：整行；`**$1` `$2` …**：第 1、2… 列  
- `**NR`**：当前行号；`**NF**`：当前行列数

#### 指定分隔符

```bash
# -F 指定字段分隔符
cat /etc/passwd | awk -F: '{print $1}'           # 用户名
echo "a,b,c" | awk -F',' '{print $2}'            # b

kubectl get pods -A --no-headers | awk '{print $1, $2}'   # NS NAME
df -h | awk 'NR==2 {print $5}'                   # 根分区使用率
```

#### 模式 + 动作

```bash
# 条件过滤
ps aux | awk '$3 > 50.0 {print $2, $3, $11}'     # CPU>50%: PID CPU CMD
awk '$1 == "ERROR" {count++} END {print count}' log.txt

# 正则匹配
awk '/ERROR/ {print $0}' app.log
awk '/^2026-/ {print $0}' app.log                 # 以日期开头

# 范围模式
awk 'NR>=10 && NR<=20 {print}' file.txt          # 10-20 行
sed -n '10,20p' file.txt                         # 等价 sed
```

#### BEGIN / END（统计必备）

```bash
# 求和、平均
awk '{sum+=$1; n++} END {print "sum=", sum, "avg=", sum/n}' metrics.txt

# 统计行数（等同 wc -l）
awk 'END {print NR}' file.txt

# 按列求和
awk '{sum+=$9} END {print sum}' access.log       # 第9列流量合计
```

#### 数组统计（Top IP）

```bash
# 访问日志第1列 IP 计数 Top10
awk '{cnt[$1]++} END {for (ip in cnt) print cnt[ip], ip}' access.log \
  | sort -rn | head -10
```

#### kubectl 运维常用 awk

```bash
# NotReady 节点
kubectl get nodes --no-headers | awk '$2 != "Ready" {print $1}'

# CrashLoopBackOff Pod
kubectl get pods -A --no-headers | awk '$4 == "CrashLoopBackOff" {print $1, $2}'

# 只看 default 命名空间 Pod 名
kubectl get pods -n default --no-headers | awk '{print $1}'

# 节点第2列 STATUS
kubectl get nodes | awk 'NR>1 {print $1, $2}'
```

#### printf 格式化

```bash
df -h | awk 'NR>1 {printf "%-20s %5s\n", $1, $5}'
```

---

### D. sed 详解

```bash
# 替换（首次）
sed 's/foo/bar/' file.txt

# 全局替换
sed 's/enforcing/disabled/g' /etc/selinux/config

# 原地修改（Linux）
sed -i 's/old/new/g' file.txt
sed -i.bak 's/old/new/g' file.txt    # 备份 .bak

# 打印行范围
sed -n '10,20p' file.txt

# 删除行
sed '/^#/d' file.txt                 # 删注释
sed '/^$/d' file.txt                 # 删空行

# 多行删除 / 追加
sed '1d' file.txt                    # 删首行
```

---

### E. cut / sort / uniq / wc

```bash
# cut：简单切列
echo "a:b:c" | cut -d: -f1          # a
df -h | cut -d' ' -f5               # 注意 df 空格不对齐，优先 awk

# sort
sort file.txt
sort -n numbers.txt                  # 数值排序
sort -rn access.log                  # 逆序数值
sort -k2 -n file.txt                 # 按第2列数值排

# uniq（必须先 sort）
sort access.log | uniq               # 去重
sort access.log | uniq -c            # 计数
sort access.log | uniq -c | sort -rn | head -10   # Top10

# wc
wc -l file.txt                       # 行数
grep -c Error app.log                # 含 Error 的行数
grep Error app.log | wc -l           # 等价（但 grep -c 更快）
```

---

### F. find 详解

#### 基本语法

```bash
find [起始路径] [条件] [动作]
# 默认：从起始路径递归向下，对每个匹配条件的文件执行动作（默认 -print）
find . -name "*.log"                 # 当前目录及子目录
find /var/log -type f -mtime -1      # 只搜 /var/log 下
```

#### 常用条件（测试表达式）


| 选项            | 含义                          | 示例                                 |
| ------------- | --------------------------- | ---------------------------------- |
| `-name PAT`   | 文件名匹配（**区分大小写**，支持 `*` `?`） | `-name "*.log"`                    |
| `-iname PAT`  | 文件名匹配（**忽略大小写**）            | `-iname "*.LOG"`                   |
| `-type f/d/l` | 类型：普通文件 / 目录 / 符号链接         | `-type f`                          |
| `-mtime N`    | 修改时间（**天**，24h 为单位）         | `-mtime -1` 24h 内；`-mtime +7` 7 天前 |
| `-mmin N`     | 修改时间（**分钟**）                | `-mmin -60` 1 小时内                  |
| `-atime N`    | 访问时间（天）                     | `-atime +30`                       |
| `-size N`     | 文件大小                        | `-size +100M`；`-size -1k`          |
| `-maxdepth N` | 最大搜索深度                      | `-maxdepth 1` 仅当前层                 |
| `-mindepth N` | 最小深度                        | `-mindepth 2` 跳过顶层                 |
| `-path PAT`   | 路径匹配（含目录名）                  | `-path "*/cache/*"`                |
| `-perm MODE`  | 权限                          | `-perm 644`；`-perm -u+x`           |
| `-user NAME`  | 属主                          | `-user root`                       |
| `-empty`      | 空文件或空目录                     |                                    |
| `-newer FILE` | 比某文件新                       | `-newer ref.log`                   |


`**-mtime` 符号记忆**（以「整 24 小时」为 1 天）：


| 写法          | 含义                   |
| ----------- | -------------------- |
| `-mtime 0`  | 今天（0～24h 内）          |
| `-mtime -1` | 24 小时内（比 1 天前 **新**） |
| `-mtime +7` | 7×24h **之前**修改的（更旧）  |


`**-size` 单位**：`c` 字节（默认）、`k`/`M`/`G`；`+100M` 大于 100MB，`-1k` 小于 1KB。

#### 逻辑组合

```bash
find . -name "*.log" -o -name "*.txt"          # -o：或（注意与 -a 优先级，复杂时用括号）
find . \( -name "*.log" -o -name "*.txt" \) -mtime -1   # 括号需转义
find . -type f -name "*.log" -mtime -7         # 多个条件默认 -a（且）
find . ! -name "*.tmp" -type f                 # ! 取反
```

#### 常用动作


| 动作                | 含义                          |
| ----------------- | --------------------------- |
| `-print`          | 打印路径（**默认**）                |
| `-print0`         | 以 `\0` 分隔输出（配合 `xargs -0`）  |
| `-ls`             | 类似 `ls -dils` 详细列出          |
| `-delete`         | 删除匹配项（**慎用**，先 `-print` 确认） |
| `-exec CMD {} \;` | 对每个文件执行命令（**每个文件一次**）       |
| `-exec CMD {} +`  | 批量传给命令（**更高效**，类似 xargs）    |


#### `-exec` vs `xargs`

```bash
# 每个文件单独 grep（慢但安全）
find . -name "*.log" -exec grep -l "Error" {} \;

# 批量传给 grep（推荐，{} + 末尾无分号）
find . -name "*.log" -exec grep -l "Error" {} +

# -print0 + xargs -0：文件名含空格也安全
find /var/log -name "*.log" -print0 | xargs -0 grep -l "Error"

# 删除前先预览（面试必说）
find /tmp -name "*.tmp" -mtime +7 -print      # 先看
find /tmp -name "*.tmp" -mtime +7 -delete     # 确认后再删
```

#### 基础示例

```bash
# 当前目录下 .log 文件（不递归子目录）
find . -maxdepth 1 -type f -name "*.log"

# 24 小时内修改的日志
find /var/log -type f -name "*.log" -mtime -1

# 大于 100MB 的文件
find / -type f -size +100M 2>/dev/null

# 7 天前的 .gz 日志（清理前预览）
find /var/log -type f -name "*.gz" -mtime +7 -ls

# 空目录
find . -type d -empty

# 按权限找 world-writable 文件（安全审计）
find / -type f -perm -002 2>/dev/null
```

#### find + grep / awk（运维高频）

```bash
# 当前目录 .log 中含 Error 的文件名
find . -maxdepth 1 -name "*.log" -exec grep -l "Error" {} +

# 当前目录 .log 中含 Error 的总行数
find . -maxdepth 1 -type f -name "*.log" -exec grep -h "Error" {} + | wc -l

# 递归搜 yaml 再 grep image
find ./manifests -name "*.yaml" -exec grep -H "image:" {} +

# 找大文件并按大小排序展示
find /var/log -type f -size +50M -exec ls -lh {} + | sort -k5 -h
```

#### find vs locate vs which


| 命令         | 特点                           |
| ---------- | ---------------------------- |
| **find**   | 实时遍历，条件灵活，**面试/生产首选**        |
| **locate** | 查预建索引（`updatedb`），快但可能不是最新   |
| **which**  | 查 **PATH 里可执行文件** 的路径，不搜普通文件 |


```bash
which kubectl python3
locate nginx.conf          # 需 mlocate/plocate 包
```

#### 面试/复盘速答

```bash
# Q：当前目录所有 .log 里 Error 行数？
find . -maxdepth 1 -type f -name "*.log" -exec grep -c "Error" {} +

# Q：/var/log 下大于 100MB 的文件？
find /var/log -type f -size +100M -exec ls -lh {} \;

# Q：24h 内改过的 .log？
find /var/log -name "*.log" -mtime -1

# Q：安全删除 7 天前的 tmp？
find /tmp -type f -name "*.tmp" -mtime +7 -print   # 先确认
find /tmp -type f -name "*.tmp" -mtime +7 -delete
```

---

### G. 综合实战（面试/复盘高频）

#### G1. 当前目录 `.log` 文件中含 `Error` 的行数

```bash
# 方法一：find + grep + wc
find . -maxdepth 1 -type f -name "*.log" -exec grep -h "Error" {} + | wc -l

# 方法二：按文件分别统计
for f in ./*.log; do
  [[ -f "$f" ]] || continue
  echo "$f: $(grep -c 'Error' "$f")"
done

# 方法三：awk 累计
awk '/Error/ {total++} END {print total+0}' ./*.log 2>/dev/null
```

#### G2. 24 小时内 ERROR 最多的 Top10 IP（日志在文件里）

```bash
# 假设日志格式：IP 在第1列，时间可先用 grep 过滤日期
grep "2026-05-29" access.log | grep "ERROR" | awk '{print $1}' \
  | sort | uniq -c | sort -rn | head -10
```

#### G3. 查占用 8080 的进程并优雅停止

```bash
pgrep -af ":8080"          # 若命令行含端口
ss -tlnp | grep 8080
pid=$(ss -tlnp | awk '/:8080 /{print $NF}' | grep -oP 'pid=\K\d+')
[[ -n "$pid" ]] && kill -15 "$pid"
```

#### G4. 统计 nginx 5xx 状态码

```bash
awk '$9 ~ /^5[0-9]{2}$/ {cnt[$9]++} END {for (c in cnt) print cnt[c], c}' access.log
# 或
grep -E '" (5[0-9]{2}) ' access.log | awk '{print $9}' | sort | uniq -c | sort -rn
```

#### G5. 磁盘使用率超过 80% 的分区

```bash
df -h | awk 'NR>1 {gsub(/%/,"",$5); if ($5+0 > 80) print $0}'
```

---

### H. 常见错误与面试提醒


| 错误                        | 正确做法                                       |
| ------------------------- | ------------------------------------------ |
| `uniq` 不去重                | 先 `sort` 再 `uniq`                          |
| `ps aux | grep nginx` 误匹配 | 用 `pgrep` 或 `grep [n]ginx`                 |
| `df | cut` 列错位            | 用 `awk` 处理 `df` 输出                         |
| grep 正则特殊字符               | 用 `-F` 固定字符串或转义                            |
| 忘记 `-r`                   | 搜目录要 `grep -r` 或 `find`                    |
| `find / -name` 扫全盘很慢      | 缩小路径；加 `-maxdepth`；权限错误用 `2>/dev/null`     |
| `find -delete` 误删         | 先 `-print` / `-ls` 确认；复杂条件用 `-exec rm` 更可控 |
| `-mtime` 符号记反             | `-mtime -1` 是 **24h 内**；`+7` 是 **7 天前更旧**  |
| `find | xargs` 文件名含空格     | 用 `-print0 | xargs -0`                     |
| `-exec {} \;` 与 `{} +` 混用 | 大量文件优先 `{} +`（批量）；特殊字符多时用 `\;`             |


**面试表达**：先说 **用什么命令、取哪一列、怎么聚合**，再写一行管道。

---

## 二、Bash 脚本（Q21–Q40）

### Q21. Bash 脚本开头 `#!/bin/bash` 和 `set -euo pipefail` 含义？

**答**：

```bash
#!/bin/bash
set -e          # 任一命令失败则退出
set -u          # 使用未定义变量报错
set -o pipefail # 管道中任一环节失败则整条失败
set -euo pipefail  # 运维脚本推荐三件套
```

---

### Q22. 如何获取脚本所在目录？

**答**：

```bash
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
```

避免 `$0` 在 source 时不可靠。

---

### Q23. 如何判断命令是否存在？

**答**：

```bash
if command -v kubectl &>/dev/null; then
  echo "kubectl found"
fi
```

---

### Q24. 读取命令行参数？

**答**：

```bash
while getopts ":n:h" opt; do
  case $opt in
    n) NAME="$OPTARG" ;;
    h) echo "Usage: $0 -n name"; exit 0 ;;
    *) echo "Invalid option"; exit 1 ;;
  esac
done
shift $((OPTIND-1))
echo "positional: $1"
```

---

### Q25. 写一个简单的健康检查脚本

**答**：

```bash
#!/bin/bash
set -euo pipefail

URL="${1:-http://127.0.0.1:8080/health}"
TIMEOUT=5

code=$(curl -s -o /dev/null -w "%{http_code}" --max-time "$TIMEOUT" "$URL" || echo "000")

if [[ "$code" == "200" ]]; then
  echo "OK $URL"
  exit 0
else
  echo "FAIL $URL code=$code"
  exit 1
fi
```

---

### Q26. 如何遍历文件并处理？

**答**：

```bash
# 推荐：处理含空格文件名
while IFS= read -r -d '' file; do
  echo "processing $file"
done < <(find /var/log -name "*.log" -print0)

# 简单场景
for f in /var/log/*.log; do
  [[ -f "$f" ]] || continue
  gzip "$f"
done
```

---

### Q27. Bash 中 `$()` 和反引号的区别？

**答**：`$(cmd)` 推荐，可嵌套；反引号 ``cmd`` 老写法，嵌套易错。

```bash
count=$(wc -l < file.txt)
```

---

### Q28. 如何写函数和返回值？

**答**：

```bash
check_disk() {
  local usage
  usage=$(df / | awk 'NR==2 {print $5}' | tr -d '%')
  if (( usage > 90 )); then
    return 1
  fi
  return 0
}

if ! check_disk; then
  echo "disk critical" >&2
  exit 1
fi
```

**注意**：return 只能 0–255；大数据用 `echo` 输出捕获。

---

### Q29. 错误处理和日志

**答**：

```bash
log() { echo "[$(date '+%F %T')] $*"; }
trap 'log "Error on line $LINENO"; exit 1' ERR

log "start backup"
# ...
log "done"
```

---

### Q30. 并发执行多个任务？

**答**：

```bash
for host in host1 host2 host3; do
  ssh "$host" 'uptime' &
done
wait    # 等待所有后台任务
```

控制并发数可用 `xargs -P` 或 `GNU parallel`。

---

### Q31. 用 xargs 批量操作

**答**：

```bash
kubectl get pods -A --no-headers | awk '{print $2}' | \
  xargs -I{} kubectl delete pod {} -n default --dry-run=client

# 并发 4 个
cat hosts.txt | xargs -P4 -I{} ssh {} 'hostname'
```

---

### Q32. 读取 YAML/JSON 配置（bash 局限）

**答**：Bash 不适合复杂 YAML，推荐：

```bash
# 简单 key
val=$(grep '^key:' config.yaml | awk '{print $2}')

# 复杂结构用 yq
yq '.spec.replicas' deploy.yaml

# 或交给 Python
python3 -c "import yaml; print(yaml.safe_load(open('f.yaml'))['key'])"
```

---

### Q33. 场景题：批量检查多台机器磁盘使用率

**答**：

```bash
#!/bin/bash
set -euo pipefail
THRESHOLD=85
HOSTS=(node1 node2 node3)
FAILED=0

for h in "${HOSTS[@]}"; do
  usage=$(ssh -o ConnectTimeout=5 "$h" "df / | awk 'NR==2 {print \$5}' | tr -d '%'" 2>/dev/null) || {
    echo "$h UNREACHABLE"; ((FAILED++)); continue
  }
  if (( usage > THRESHOLD )); then
    echo "$h CRITICAL ${usage}%"; ((FAILED++))
  else
    echo "$h OK ${usage}%"
  fi
done
exit $(( FAILED > 0 ? 1 : 0 ))
```

---

### Q34. 场景题：监控日志关键字并告警

**答**：

```bash
#!/bin/bash
LOG=/var/log/app.log
KEYWORD="ERROR"
WEBHOOK="https://hooks.example.com/alert"

tail -F "$LOG" | while read -r line; do
  if [[ "$line" == *"$KEYWORD"* ]]; then
    curl -s -X POST "$WEBHOOK" -d "{\"text\":\"$line\"}" >/dev/null
  fi
done
```

生产更常用 **Promtail + Loki + Alertmanager** 或 ELK。

---

### Q35. Bash 数组和关联数组

**答**：

```bash
arr=(a b c)
echo "${arr[0]}" "${#arr[@]}"

declare -A map
map[host1]=10.0.0.1
map[host2]=10.0.0.2
echo "${map[host1]}"
```

---

### Q36. 字符串操作

**答**：

```bash
s="k8s-master-01"
echo "${s#k8s-}"        # 去前缀 master-01
echo "${s%-01}"         # 去后缀 k8s-master
echo "${s/k8s/cluster}" # 替换
[[ "$s" == k8s-* ]] && echo "match"
```

---

### Q37. 调试 Bash 脚本？

**答**：

```bash
bash -x script.sh          # 跟踪
set -x                     # 脚本内开启
set +x                     # 关闭
shellcheck script.sh       # 静态检查（推荐）
```

---

### Q38. 如何在脚本里调用 kubectl 并解析？

**答**：

```bash
not_ready=$(kubectl get nodes --no-headers | awk '$2!="Ready"{print $1}')
if [[ -n "$not_ready" ]]; then
  echo "NotReady nodes: $not_ready"
  exit 1
fi
```

复杂 JSON 用 `kubectl jsonpath` 或 `jq`：

```bash
kubectl get pod foo -o json | jq -r '.status.phase'
```

---

### Q39. 写一个简单的备份脚本要点？

**答**：

- `set -euo pipefail`  
- 备份路径带 **日期**：`backup_$(date +%F).tar.gz`  
- 检查 **磁盘空间**  
- 保留 **最近 N 份** 轮转  
- 失败 **非 0 退出** + 告警  
- 敏感数据 **权限 600**

---

### Q40. Bash vs Python 何时选型？


| 选 Bash        | 选 Python         |
| ------------- | ---------------- |
| 调命令管道、部署 glue | 解析 JSON/YAML/API |
| 短脚本、cron 任务   | 复杂逻辑、单测          |
| 系统自带无依赖       | K8s client、数据处理  |


---

## 三、Python 运维脚本（Q41–Q60）

### Q41. 如何用 subprocess 安全执行 shell 命令？

**答**：

```python
import subprocess

# 推荐：列表形式，防注入
result = subprocess.run(
    ["kubectl", "get", "pods", "-n", "kube-system"],
    capture_output=True,
    text=True,
    timeout=30,
    check=False,
)
if result.returncode != 0:
    raise RuntimeError(result.stderr)
print(result.stdout)
```

**避免**：`os.system("rm -rf " + user_input)` — SQL/shell 注入风险。

---

### Q42. 读取 JSON/YAML 配置

**答**：

```python
import json
import yaml

with open("config.json") as f:
    cfg = json.load(f)

with open("deploy.yaml") as f:
    doc = yaml.safe_load(f)
replicas = doc["spec"]["replicas"]
```

---

### Q43. 用 requests 调 HTTP API（健康检查/告警）

**答**：

```python
import requests

def check_health(url: str, timeout: float = 5.0) -> bool:
    try:
        r = requests.get(url, timeout=timeout)
        return r.status_code == 200
    except requests.RequestException as e:
        print(f"health check failed: {e}")
        return False
```

---

### Q44. 用 kubernetes Python client 列出 NotReady 节点

**答**：

```python
from kubernetes import client, config

config.load_kube_config()  # 或 load_incluster_config()
v1 = client.CoreV1Api()

for node in v1.list_node().items:
    for cond in node.status.conditions or []:
        if cond.type == "Ready" and cond.status != "True":
            print(node.metadata.name, cond.reason, cond.message)
```

---

### Q45.  argparse 写 CLI 工具

**答**：

```python
import argparse

def main():
    p = argparse.ArgumentParser(description="Cluster checker")
    p.add_argument("-n", "--namespace", default="default")
    p.add_argument("--threshold", type=int, default=85)
    args = p.parse_args()
    print(args.namespace, args.threshold)

if __name__ == "__main__":
    main()
```

---

### Q46. 日志规范

**答**：

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)
log = logging.getLogger(__name__)
log.info("check started")
```

生产可加 **JSON 格式** 便于 ELK 采集。

---

### Q47. 场景题：解析 Nginx access log 统计 Top IP

**答**：

```python
from collections import Counter

counter = Counter()
with open("/var/log/nginx/access.log") as f:
    for line in f:
        ip = line.split()[0]
        counter[ip] += 1

for ip, cnt in counter.most_common(10):
    print(ip, cnt)
```

---

### Q48. 场景题：批量 SSH 执行命令（paramiko 思路）

**答**：

```python
import subprocess

hosts = ["node1", "node2", "node3"]
cmd = ["uptime"]

for h in hosts:
    r = subprocess.run(
        ["ssh", "-o", "BatchMode=yes", h, " ".join(cmd)],
        capture_output=True, text=True, timeout=10,
    )
    print(h, "OK" if r.returncode == 0 else "FAIL", r.stdout.strip())
```

也可用 **Fabric / paramiko** 库。

---

### Q49. 如何处理重试和退避？

**答**：

```python
import time
import requests

def get_with_retry(url, retries=3, backoff=2):
    for i in range(retries):
        try:
            r = requests.get(url, timeout=5)
            r.raise_for_status()
            return r.json()
        except requests.RequestException as e:
            if i == retries - 1:
                raise
            time.sleep(backoff ** i)
```

---

### Q50. 读取环境变量和配置文件优先级

**答**：

```python
import os

DB_HOST = os.environ.get("DB_HOST", "localhost")
DEBUG = os.environ.get("DEBUG", "false").lower() == "true"
```

**惯例**：环境变量 > 配置文件 > 默认值（12-factor）。

---

### Q51. 用 pathlib 处理路径

**答**：

```python
from pathlib import Path

log_dir = Path("/var/log/myapp")
log_dir.mkdir(parents=True, exist_ok=True)
latest = max(log_dir.glob("*.log"), key=lambda p: p.stat().st_mtime)
print(latest.read_text(encoding="utf-8", errors="replace")[-2000:])
```

---

### Q52. 并发：ThreadPoolExecutor 批量请求

**答**：

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests

urls = ["http://svc1/health", "http://svc2/health"]

def check(url):
    try:
        return url, requests.get(url, timeout=3).status_code
    except Exception as e:
        return url, str(e)

with ThreadPoolExecutor(max_workers=10) as ex:
    futs = [ex.submit(check, u) for u in urls]
    for f in as_completed(futs):
        print(f.result())
```

IO 密集用线程；CPU 密集用 **ProcessPoolExecutor**。

---

### Q53. 解析 Prometheus 指标（简单文本格式）

**答**：

```python
def parse_prom_text(text: str) -> dict[str, float]:
    metrics = {}
    for line in text.splitlines():
        if not line or line.startswith("#"):
            continue
        parts = line.rsplit(" ", 1)
        if len(parts) == 2:
            try:
                metrics[parts[0]] = float(parts[1])
            except ValueError:
                pass
    return metrics
```

生产可用 `prometheus-api-client` 库。

---

### Q54. 写单元测试（运维脚本也要测）

**答**：

```python
import unittest

def disk_ok(usage: int, threshold: int = 90) -> bool:
    return usage <= threshold

class TestDisk(unittest.TestCase):
    def test_ok(self):
        self.assertTrue(disk_ok(80))
    def test_critical(self):
        self.assertFalse(disk_ok(95))

if __name__ == "__main__":
    unittest.main()
```

---

### Q55. 场景题：检查 Deployment 副本是否就绪（K8s）

**答**：

```python
from kubernetes import client, config

def deployment_ready(name: str, namespace: str = "default") -> bool:
    config.load_kube_config()
    apps = client.AppsV1Api()
    dep = apps.read_namespaced_deployment(name, namespace)
    spec = dep.spec.replicas or 0
    ready = dep.status.ready_replicas or 0
    return spec > 0 and ready >= spec

if not deployment_ready("nginx"):
    raise SystemExit("nginx not ready")
```

---

### Q56. 如何处理 secrets 不写进代码？

**答**：

- 从 **环境变量** 或 **K8s Secret 挂载文件** 读取  
- 禁止 commit `.env`；用 `.gitignore`  
- 日志里 **脱敏** token

```python
token = os.environ["API_TOKEN"]  # 启动前注入
```

---

### Q57. 定时任务：cron vs Python schedule vs APScheduler


| 方式          | 场景        |
| ----------- | --------- |
| cron        | 系统级、简单脚本  |
| APScheduler | 复杂调度、进程内  |
| K8s CronJob | 集群内定时 Pod |


---

### Q58. 场景题：合并多个监控检查结果发钉钉/飞书 webhook

**答**：

```python
import json
import requests

def notify(webhook: str, title: str, items: list[tuple[str, bool]]):
    lines = [f"- {'✅' if ok else '❌'} {name}" for name, ok in items]
    body = {"msg_type": "text", "content": {"text": f"{title}\n" + "\n".join(lines)}}
    requests.post(webhook, json=body, timeout=10)

checks = [
    ("API health", True),
    ("Disk", False),
]
notify(WEBHOOK, "Daily Check", checks)
```

---

### Q59. Python 2 vs 3 / 虚拟环境

**答**：

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

运维脚本统一 **Python 3.8+**；依赖 pin 版本（`requirements.txt`）。

---

### Q60. 结合 ConfigPilot：Python 在运维中的优势？

**答**：你的 Meshscope 用 Python 做 **YAML 解析、图算法、正交实验、Prom/Zipkin API 集成**——复杂数据结构用 Python 比 Bash 可维护得多；发布门禁可写 **CLI + CI 插件**。

---

## 四、综合场景 & 手撕题（Q61–Q75）

### Q61. 手撕：统计 /var/log 下大于 100MB 的文件

**Bash**：

```bash
find /var/log -type f -size +100M -exec ls -lh {} \;
```

> `-size`、`-exec` 详解见 **一点五 F. find**。

**Python**：

```python
from pathlib import Path
for p in Path("/var/log").rglob("*"):
    if p.is_file() and p.stat().st_size > 100 * 1024 * 1024:
        print(p.stat().st_size, p)
```

---

### Q62. 手撕：找出占用 8080 端口的 PID 并 kill

```bash
pid=$(ss -tlnp | awk '/:8080 /{print $NF}' | grep -oP 'pid=\K\d+')
[[ -n "$pid" ]] && kill -15 "$pid"
```

---

### Q63. 手撕：检查证书剩余天数

```python
import ssl
import socket
from datetime import datetime, timezone

def cert_days_left(host: str, port: int = 443) -> int:
    ctx = ssl.create_default_context()
    with socket.create_connection((host, port), timeout=5) as sock:
        with ctx.wrap_socket(sock, server_hostname=host) as ssock:
            cert = ssock.getpeercert()
    expire = datetime.strptime(cert["notAfter"], "%b %d %H:%M:%S %Y %GMT")
    expire = expire.replace(tzinfo=timezone.utc)
    return (expire - datetime.now(timezone.utc)).days

print(cert_days_left("kubernetes.io"))
```

---

### Q64. 手撕：从 kubectl 输出提取 CrashLoopBackOff 的 Pod

```bash
kubectl get pods -A --no-headers | awk '$4=="CrashLoopBackOff"{print $1, $2}'
```

```python
import subprocess
out = subprocess.check_output(["kubectl", "get", "pods", "-A", "--no-headers"], text=True)
for line in out.splitlines():
    ns, name, *rest = line.split()
    if len(rest) >= 3 and rest[2] == "CrashLoopBackOff":
        print(ns, name)
```

---

### Q65. 手撕：简单端口扫描（合规环境）

```python
import socket

def scan(host: str, ports: range, timeout=0.5):
    open_ports = []
    for port in ports:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.settimeout(timeout)
            if s.connect_ex((host, port)) == 0:
                open_ports.append(port)
    return open_ports

print(scan("127.0.0.1", range(1, 1024)))
```

---

### Q66. 场景：发布前自动化检查清单脚本应检查什么？

**答**（可用 Bash+Python 组合）：

- `kubectl get nodes` 全部 Ready  
- 目标 Deployment readyReplicas == replicas  
- 镜像 tag 非 latest  
- 关键 Pod 无 CrashLoopBackOff  
- PVC Bound  
- 最近 5min 5xx 错误率 < 阈值  
- 磁盘/内存未告警

---

### Q67. 场景：如何安全地在生产执行脚本？

- **dry-run** 默认开启  
- **变更窗口** + 审批  
- **幂等** 设计  
- **回滚脚本** 成对  
- 日志审计 **who/when/what**  
- 避免 `rm -rf /` 类：变量加引号、路径白名单

---

### Q68. idempotency 在运维脚本中的例子？

**Ansible 风格**：再跑一遍结果相同。

```bash
# 用户存在则跳过
id myuser &>/dev/null || useradd myuser
```

---

### Q69. 如何管理多环境（dev/staging/prod）配置？

- 不同 **kubeconfig / context**  
- 环境变量 `ENV=prod`  
- 目录 `config/dev.yaml`、`config/prod.yaml`  
- **禁止** prod 脚本里硬编码 dev 地址

```bash
kubectl config use-context prod-cluster
```

---

### Q70. 你用过哪些运维自动化工具？

**可答**：Shell/Python 自研脚本、Ansible（批量配置）、Terraform（IaC）、Helm（K8s 包管理）、GitHub Actions/GitLab CI、Cursor 辅助写脚本。

---

### Q71–Q75. 快速问答


| #   | 问题                        | 简答                         |
| --- | ------------------------- | -------------------------- |
| 71  | `tail -f` 和 `less +F` 区别？ | less 可翻历史；tail 简单跟随        |
| 72  | `|` 管道返回值？                | 默认最后命令；`pipefail` 任一段失败则失败 |
| 73  | `$?` 含义？                  | 上一条命令退出码，0=成功              |
| 74  | Python GIL 影响运维脚本？        | IO 型几乎无影响；CPU 密集用多进程       |
| 75  | Meshscope 若用 Bash 会怎样？    | YAML/图/正交表难维护，故选 Python    |


---

## 五、复习建议（运维开发岗）


| 优先级 | 内容                                                          |
| --- | ----------------------------------------------------------- |
| ⭐⭐⭐ | top/ss/df/journalctl、**grep/find/pgrep/awk 一点五**、kubectl 排障 |
| ⭐⭐⭐ | Bash：`set -euo pipefail`、函数、参数、健康检查                         |
| ⭐⭐  | Python：subprocess、requests、yaml、K8s client                  |
| ⭐⭐  | 场景：批量巡检、日志分析、发布检查                                           |
| ⭐   | sed/正则、paramiko、Prometheus API                              |


---

## 相关文档


| 文档      | 链接                                                                           |
| ------- | ---------------------------------------------------------------------------- |
| 集群搭建实操  | [k8s-cluster-setup-guide.md](../05-cli-bootstrap/k8s-cluster-setup-guide.md) |
| 稳定性通用答法 | [feishu-stability-interview-qa.md](./feishu-stability-interview-qa.md)       |


