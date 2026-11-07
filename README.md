# 高可用架构与 SLA 保障实战指南

> 本仓库聚焦 VPS 高可用架构设计与 SLA 保障实战方案，涵盖多区域部署、故障转移、负载均衡、实时监控、灾备恢复等核心技术实践。

---

## 目录

- [项目概述](#项目概述)
- [高可用架构核心模式](#高可用架构核心模式)
- [SLA 指标体系](#sla-指标体系)
- [多区域部署方案](#多区域部署方案)
- [健康检查与自动故障转移](#健康检查与自动故障转移)
- [负载均衡实战](#负载均衡实战)
- [CDN 集成与流量调度](#cdn-集成与流量调度)
- [DNS Failover 自动化](#dns-failover-自动化)
- [Auto-Scaling 自动扩缩容](#auto-scaling-自动扩缩容)
- [灾备恢复 RTO/RPO](#灾备恢复-rtorpo)
- [DDoS 防护策略](#ddos-防护策略)
- [实战脚本库](#实战脚本库)
- [经典案例分析](#经典案例分析)
- [FAQ 常见问题](#faq-常见问题)
- [附录 A · 存储退化与磁盘故障的早期识别](#附录-a--存储退化与磁盘故障的早期识别)

---

## 项目概述

### 为什么需要高可用架构？

在生产环境中，服务器宕机、网络抖动、硬件故障等突发事件随时可能发生。**单点部署**意味着任何一次故障都可能导致服务中断、数据丢失，直接影响用户体验和业务收入。

**高可用（High Availability, HA）** 架构通过冗余设计、自动故障检测与切换、多区域部署等手段，将系统可用性提升至 99.9% 甚至 99.99% 以上。

### 本仓库涵盖的核心主题

| 主题 | 描述 |
|------|------|
| **多区域部署** | 跨可用区（AZ）/跨地域（Region）的冗余架构 |
| **故障转移（Failover）** | 自动检测故障并切换到备用节点的机制 |
| **负载均衡（Load Balancing）** | 流量在多节点间的智能分发策略 |
| **SLA 计算** | 可用性目标的量化衡量与合同承诺 |
| **实时监控** | Uptime、Latency、Error Rate 的持续追踪 |
| **灾备恢复** | DR 演练、RTO/RPO 设计与数据同步方案 |
| **CDN 集成** | 内容分发网络与边缘计算的协同 |
| **DNS Failover** | 基于 DNS 的智能路由与故障切换 |
| **Auto-Scaling** | 弹性扩缩容与成本优化 |
| **DDoS 防护** | 流量清洗与攻击缓解策略 |

> **VPSVIP & ClashVIP 精选推荐**：选择支持多可用区、配备高可用基础设施的 VPS 服务商，是构建高可用系统的第一步。  
> 推荐导航：[nav.clashvip.net](https://nav.clashvip.net) | [clashvip.net](https://clashvip.net)

---

## 高可用架构核心模式

### 1. 主从（Master-Slave / Primary-Replica）模式

```
[客户端] → [VIP / 负载均衡器]
              ↓
      ┌───────────────┐
      │   Master/N1   │ ← 主节点（写）
      └───────┬───────┘
              │ 同步复制
      ┌───────┴───────┐
      │   Slave/N2   │ ← 备用节点（读/故障转移）
      │   Slave/N3   │ ← 冷备节点（定时同步）
      └───────────────┘
```

**适用场景**：数据库主从、Redis 主从、数据存储层  
**优势**：实现简单，主节点故障时从节点快速接管  
**劣势**：存在数据同步延迟，单点写入仍集中在主节点

### 2. 双活（Active-Active）模式

```
        [全球负载均衡 / Anycast DNS]
               /          \
    ┌──────────┐          ┌──────────┐
    │ Region A │          │ Region B │
    │ Active   │  ←──→    │ Active   │
    └──────────┘  跨区域   └──────────┘
                   同步
```

**适用场景**：全球化业务、需要极低延迟的多地域服务  
**优势**：两区域同时服务，单区域故障不影响全局可用性  
**劣势**：跨区域同步复杂度高，数据一致性挑战大

### 3. 分布式无中心（Distributed / Multi-Master）模式

**适用场景**：超大规模系统、Cassandra / CockroachDB 等分布式数据库  
**优势**：无单点故障，可线性扩展  
**劣势**：一致性协议复杂（如 Paxos / Raft），运维难度高

### 4. 熔断器（Circuit Breaker）模式

防止级联故障的核心保护机制：

```
正常状态（Closed）
  └─→ 故障率超过阈值（如 50% / 10秒）
       └─→ 熔断打开（Open），快速失败，拒绝请求
            └─→ 半开试探（Half-Open）
                 ├─→ 成功 → 恢复 Closed
                 └─→ 失败 → 继续 Open
```

**推荐库**：
- **.NET**：Polly
- **Java**：Resilience4j
- **Node.js**：opossum
- **Go**：sony/gobreaker

---

## SLA 指标体系

### SLA 是什么？

SLA（Service Level Agreement，服务等级协议）是服务提供商与客户之间对服务质量的法律承诺，核心指标包括：

### 核心 SLA 指标

| 指标 | 定义 | 计算公式 |
|------|------|----------|
| **可用性（Availability）** | 服务正常工作时间占总时间的比例 | `(总时间 - 宕机时间) / 总时间 × 100%` |
| **MTTR** | 平均恢复时间（Mean Time To Repair） | `Σ(故障恢复时间 - 故障发生时间) / 故障次数` |
| **MTBF** | 平均故障间隔时间（Mean Time Between Failures） | `总运行时间 / 故障次数` |
| **RTO** | 恢复时间目标（Recovery Time Objective） | 业务中断到恢复的最大可接受时间 |
| **RPO** | 恢复点目标（Recovery Point Objective） | 最大可接受的数据丢失时间窗口 |

### SLA 与可用性等级换算

| SLA 承诺 | 年允许宕机时间 | 月允许宕机时间 | 周允许宕机时间 |
|---------|-------------|-------------|-------------|
| 99% | 3.65 天 | 7.31 小时 | 1.68 小时 |
| 99.5% | 1.83 天 | 3.65 小时 | 50.4 分钟 |
| 99.9% | 8.76 小时 | 43.83 分钟 | 10.08 分钟 |
| 99.95% | 4.38 小时 | 21.92 分钟 | 5.04 分钟 |
| 99.99% | 52.6 分钟 | 4.38 分钟 | 1.01 分钟 |
| 99.999% | 5.26 分钟 | 26.3 秒 | 6.05 秒 |

### SLA 计算实战（PowerShell）

```powershell
# PowerShell: 计算月度和年度可用性
function Get-SLAStatus {
    param(
        [int]$DowntimeMinutesPerMonth,
        [double]$SlaTarget = 99.9
    )

    $totalMinutesPerMonth = 30 * 24 * 60
    $totalMinutesPerYear  = 365 * 24 * 60

    $monthlyAvailability = (($totalMinutesPerMonth - $DowntimeMinutesPerMonth) / $totalMinutesPerMonth) * 100
    $yearlyDowntimeMinutes = $DowntimeMinutesPerMonth * (365 / 30)
    $yearlyAvailability    = (($totalMinutesPerYear - $yearlyDowntimeMinutes) / $totalMinutesPerYear) * 100

    Write-Host "===========================================" -ForegroundColor Cyan
    Write-Host "  SLA 可用性分析报告" -ForegroundColor Green
    Write-Host "===========================================" -ForegroundColor Cyan
    Write-Host "  目标 SLA:           $($SlaTarget)%"
    Write-Host "  月度宕机时间:        $($DowntimeMinutesPerMonth) 分钟"
    Write-Host "  年度宕机时间:        $([math]::Round($yearlyDowntimeMinutes, 1)) 分钟 ($([math]::Round($yearlyDowntimeMinutes/60, 2)) 小时)"
    Write-Host "  ----------------------------------------"
    Write-Host "  月度可用性:          $([math]::Round($monthlyAvailability, 4))%"
    Write-Host "  年度可用性:          $([math]::Round($yearlyAvailability, 4))%"
    Write-Host "  ----------------------------------------"

    if ($monthlyAvailability -ge $SlaTarget) {
        Write-Host "  ✅ 状态: 达标 - 当前可用性 >= 目标 SLA" -ForegroundColor Green
    } else {
        $gap = $SlaTarget - $monthlyAvailability
        $allowedDowntimeMonthly = ($totalMinutesPerMonth * (100 - $SlaTarget)) / 100
        Write-Host "  ❌ 状态: 不达标 - 当前可用性 < 目标 SLA" -ForegroundColor Red
        Write-Host "  差距: $([math]::Round($gap, 4))%"
        Write-Host "  月度可用时间内需将宕机控制在 $([math]::Round($allowedDowntimeMonthly, 1)) 分钟以内"
    }
    Write-Host "==========================================="
}

# 示例：月度宕机 30 分钟，目标 99.9%
Get-SLAStatus -DowntimeMinutesPerMonth 30 -SlaTarget 99.9

# 示例：月度宕机 5 分钟，目标 99.99%
Get-SLAStatus -DowntimeMinutesPerMonth 5 -SlaTarget 99.99
```

---

## 多区域部署方案

### 区域规划策略

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| **同 Region 多 AZ** | 同一地域的不同可用区，延迟 < 5ms | 常规高可用、数据同城灾备 |
| **跨 Region 主备** | 主力区域 + 灾备区域，主写主备读 | 数据异地灾备、合规要求 |
| **跨 Region 双活** | 两区域均为主动服务，互为备份 | 全球化低延迟业务 |
| **Anycast 路由** | 用户就近接入最近节点 | CDN / DNS / 静态资源分发 |

### 推荐的多区域架构（以 3 区域为例）

```
                    ┌─────────────────┐
                    │  Global DNS     │
                    │  (Route53/AliDNS)│
                    └────────┬────────┘
                             │ Geo-DNS 智能解析
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │  美西节点   │  │  亚太节点   │  │  欧洲节点   │
     │  (us-west) │  │  (ap-east) │  │  (eu-west) │
     └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
           │                │                │
      ┌────┴────┐      ┌────┴────┐      ┌────┴────┐
      │ Primary │      │ Primary │      │ Primary │
      │ Standby │      │ Standby │      │ Standby │
      └─────────┘      └─────────┘      └─────────┘
           └────────────────┼────────────────┘
                     Cross-Region Sync
                     (同步复制 / CRDT)
```

### 跨区域数据同步策略

| 同步模式 | 描述 | RPO | 延迟 |
|---------|------|-----|------|
| **同步复制** | 写入时等待所有副本确认 | 0（无数据丢失） | 高 |
| **半同步复制** | 主节点 + 至少一个从节点确认 | ≈ 0 | 中 |
| **异步复制** | 主节点写入即返回 | > 0（有数据丢失风险） | 低 |
| **CRDT** | 无冲突复制数据类型（最终一致） | 毫秒级 | 低 |

---

## 健康检查与自动故障转移

### 健康检查策略

| 检查类型 | 实现方式 | 检查频率 | 适用场景 |
|---------|---------|---------|----------|
| **TCP 端口检测** | 尝试建立 TCP 连接 | 5-30 秒 | 服务监听端口存活 |
| **HTTP(S) 探测** | GET /health 端点，检测 2xx | 5-30 秒 | Web 服务、API |
| **ICMP Ping** | 网络层可达性检测 | 10-60 秒 | 基础网络连通性 |
| **自定义脚本** | 执行自定义检查逻辑 | 1-5 分钟 | 复杂业务逻辑检查 |

### 健康检查脚本（PowerShell）

```powershell
# ==============================================================
# VPS 高可用健康检查脚本 - Windows PowerShell
# 功能：检测多个 VPS 节点状态，连续失败 N 次触发故障转移
# ==============================================================

$VPS_NODES = @(
    @{ Name="Node-01 (美西)"; Host="104.XX.XX.01"; Port=443; URL="https://104.XX.XX.01/health"; Weight=1; FailCount=0; SuccessCount=0; Status="UNKNOWN" },
    @{ Name="Node-02 (亚太)"; Host="104.XX.XX.02"; Port=443; URL="https://104.XX.XX.02/health"; Weight=1; FailCount=0; SuccessCount=0; Status="UNKNOWN" },
    @{ Name="Node-03 (欧洲)"; Host="104.XX.XX.03"; Port=443; URL="https://104.XX.XX.03/health"; Weight=1; FailCount=0; SuccessCount=0; Status="UNKNOWN" }
)

$CHECK_INTERVAL      = 15   # 检查间隔（秒）
$TIMEOUT_SEC         = 8    # 超时（秒）
$FAIL_THRESHOLD      = 3    # 连续失败 N 次判定故障
$RECOVERY_THRESHOLD  = 2    # 连续成功 N 次判定恢复
$WEBHOOK_URL         = ""   # 可选 Webhook（飞书/钉钉/Bark）

function Write-Log {
    param([string]$Message, [string]$Level = "INFO")
    $ts = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $color = @{ OK="Green"; FAIL="Red"; WARN="Yellow"; INFO="Cyan" }[$Level]
    Write-Host "[$ts] [$Level] $Message" -ForegroundColor $color
}

function Send-Webhook {
    param($Title, $Body)
    if (-not $WEBHOOK_URL) { return }
    try {
        $payload = @{ title=$Title; content=$Body; date=(Get-Date -Format "yyyy-MM-dd HH:mm:ss") } | ConvertTo-Json -Compress
        Invoke-RestMethod -Uri $WEBHOOK_URL -Method POST -Headers @{"Content-Type"="application/json"} -Body ([System.Text.Encoding]::UTF8.GetBytes($payload)) -TimeoutSec 5 | Out-Null
        Write-Log "Webhook 通知已发送" "INFO"
    } catch { Write-Log "Webhook 发送失败: $_" "WARN" }
}

function Test-NodeHealth {
    param($Node)
    $failCount   = $Node.FailCount
    $successCount = $Node.SuccessCount
    $status = "UNKNOWN"; $latency = -1

    try {
        $sw = [Diagnostics.Stopwatch]::StartNew()
        $resp = Invoke-WebRequest -Uri $Node.URL -Method GET -TimeoutSec $TIMEOUT_SEC -UseSSL -ErrorAction SilentlyContinue
        $sw.Stop(); $latency = $sw.ElapsedMilliseconds
        if ($resp.StatusCode -eq 200) {
            $status = "HEALTHY"; $Node.SuccessCount = $successCount + 1; $Node.FailCount = 0
        } else {
            $status = "UNHEALTHY"; $Node.FailCount = $failCount + 1; $Node.SuccessCount = 0
        }
    } catch {
        try {
            $tcp = New-Object System.Net.Sockets.TcpClient
            $conn = $tcp.BeginConnect($Node.Host, $Node.Port, $null, $null)
            $wait = $conn.AsyncWaitHandle.WaitOne($TIMEOUT_SEC * 1000, $false)
            if ($wait -and $tcp.Connected) {
                $status = "PARTIAL"; $Node.FailCount = $failCount + 1; $Node.SuccessCount = 0
            } else {
                $status = "DOWN"; $Node.FailCount = $failCount + 1; $Node.SuccessCount = 0
            }
            $tcp.Close()
        } catch {
            $status = "DOWN"; $Node.FailCount = $failCount + 1; $Node.SuccessCount = 0
        }
    }
    $Node.Status = $status; $Node.Latency = $latency
    return @{ Name=$Node.Name; Status=$status; Latency=$latency; FailCount=$Node.FailCount; IsHealthy=($Node.FailCount -lt $FAIL_THRESHOLD) }
}

function Invoke-FailoverDecision {
    param($results)
    $healthy = $results | Where-Object { $_.IsHealthy }
    $unhealthy = $results | Where-Object { -not $_.IsHealthy }

    Write-Log "==========================================" "INFO"
    Write-Log "  节点健康状态汇总" "INFO"
    Write-Log "==========================================" "INFO"

    foreach ($node in $results) {
        $failTag = if (-not $node.IsHealthy) { " ← ⚠️ 故障转移" } else { "" }
        $latTag  = if ($node.Latency -ge 0) { " [${node.Latency}ms]" } else { " [N/A]" }
        $lvl = if ($node.IsHealthy) { "OK" } else { "FAIL" }
        Write-Log "  $($node.Name): $($node.Status)$latTag  连续失败: $($node.FailCount)$failTag" $lvl
    }
    Write-Log "  健康节点: $($healthy.Count) / $($results.Count)" "INFO"

    if ($unhealthy.Count -gt 0) {
        $msg = "故障节点: $($unhealthy.Name -join ', ')"
        Write-Log $msg "FAIL"
        Send-Webhook -Title "⚠️ VPS 节点故障告警" -Body $msg
    }
    return @{ PrimaryNode=($healthy | Sort-Object Latency | Select-Object -First 1).Name; HealthyCount=$healthy.Count; TotalCount=$results.Count }
}

Write-Log "==========================================" "INFO"
Write-Log "  VPS 高可用健康检查系统启动" "INFO"
Write-Log "  监控节点数: $($VPS_NODES.Count) | 间隔: ${CHECK_INTERVAL}s | 故障阈值: ${FAIL_THRESHOLD}次" "INFO"
Write-Log "==========================================" "INFO"

while ($true) {
    $results = @()
    foreach ($node in $VPS_NODES) {
        $results += Test-NodeHealth -Node $node
    }
    $decision = Invoke-FailoverDecision -NodeResults $results

    if ($decision.HealthyCount -eq 0) {
        $alert = "🚨 紧急: 所有 VPS 节点均不可用！立即介入！"
        Write-Log $alert "FAIL"
        Send-Webhook -Title "🚨 全节点宕机紧急告警" -Body $alert
    } elseif ($decision.HealthyCount -eq 1 -and $decision.TotalCount -gt 1) {
        $warn = "⚠️ 警告: 仅剩 $($decision.PrimaryNode) 一个健康节点"
        Write-Log $warn "WARN"
        Send-Webhook -Title "⚠️ 单节点运行告警" -Body $warn
    }
    Write-Log "等待 ${CHECK_INTERVAL} 秒后进行下次检查...`n" "INFO"
    Start-Sleep -Seconds $CHECK_INTERVAL
}
```

### 自动故障转移脚本（Nginx Upstream 动态更新）

```bash
#!/bin/bash
# failover.sh - Nginx Upstream 动态故障转移脚本
# 适合场景：VPS 作为 Web 后端，Nginx 做负载均衡

UPSTREAM_FILE="/etc/nginx/conf.d/upstream_vps.conf"
LOG_FILE="/var/log/failover.log"
TELEGRAM_BOT_TOKEN="YOUR_BOT_TOKEN"
TELEGRAM_CHAT_ID="YOUR_CHAT_ID"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"; }

NODES=(
    "104.XX.XX.01:443:1"
    "104.XX.XX.02:443:1"
    "104.XX.XX.03:443:1"
)

check_node() {
    local host=$(echo $1 | cut -d: -f1)
    local port=$(echo $1 | cut -d: -f2)
    if curl -sf --max-time 5 "https://${host}/health" > /dev/null 2>&1; then
        echo "UP"
    elif timeout 3 bash -c "echo > /dev/tcp/${host}/${port}" 2>/dev/null; then
        echo "PARTIAL"
    else
        echo "DOWN"
    fi
}

notify() {
    local msg=$1; local lvl=$2
    log "[${lvl}] $msg"
    if [[ -n "$TELEGRAM_BOT_TOKEN" ]]; then
        curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
            -d "chat_id=${TELEGRAM_CHAT_ID}&text=[${lvl}] $msg" > /dev/null 2>&1
    fi
}

generate_conf() {
    local content="# 自动生成于 $(date '+%Y-%m-%d %H:%M:%S')\nupstream vps_backend {\n    least_conn;\n"
    local up_count=0
    for node in "${NODES[@]}"; do
        local host=$(echo $node | cut -d: -f1)
        local port=$(echo $node | cut -d: -f2)
        local weight=$(echo $node | cut -d: -f3)
        local status=$(check_node "$node")
        if [[ "$status" == "UP" ]]; then
            content+="    server ${host}:${port} weight=${weight} max_fails=3 fail_timeout=30s;\n"
            ((up_count++))
        elif [[ "$status" == "PARTIAL" ]]; then
            content+="    server ${host}:${port} weight=1 max_fails=5 fail_timeout=60s;  # 降权\n"
            notify "节点 ${host}:${port} 健康检查异常（降权运行）" "WARN"
        else
            notify "节点 ${host}:${port} 已下线" "WARN"
        fi
    done
    content+="    # 健康节点: ${up_count} / ${#NODES[@]}\n}\n"
    echo -e "$content"
}

reload_nginx() {
    log "重载 Nginx 配置..."
    nginx -t && nginx -s reload && log "Nginx 重载成功" || log "Nginx 重载失败" "ERROR"
}

log "========== 开始故障检查 =========="
generate_conf > "${UPSTREAM_FILE}.new"
mv "${UPSTREAM_FILE}.new" "$UPSTREAM_FILE"
reload_nginx
log "========== 检查完成 =========="
```

---

## 负载均衡实战

### 负载均衡算法对比

| 算法 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| **Round Robin** | 轮询分配 | 实现简单 | 不考虑服务器差异 | 节点配置相同 |
| **Weighted RR** | 按权重轮询 | 资源差异化分配 | 需要手动配置权重 | 节点性能不均 |
| **Least Connections** | 优先分给连接数最少 | 自适应负载 | 实现稍复杂 | 长连接场景 |
| **IP Hash** | 按客户端 IP 哈希 | 会话粘性 | 分布不均 | 需会话保持 |
| **Least Response Time** | 优先最快响应 | 最佳用户体验 | 需健康监控配合 | 动态优化 |

### Layer 4 vs Layer 7 负载均衡

| 层级 | OSI 层 | 原理 | 优势 | 劣势 |
|------|--------|------|------|------|
| **L4 (TCP/UDP)** | 传输层 | 基于 IP + Port 转发 | 高性能、低延迟、无协议限制 | 无法感知应用层信息 |
| **L7 (HTTP/HTTPS)** | 应用层 | 基于 URL/Header/Cookie | 智能路由、缓存、SSL 卸载 | 延迟稍高、资源消耗大 |

### Nginx 负载均衡配置（高可用场景）

```nginx
# /etc/nginx/conf.d/ha_lb.conf

upstream vps_backend {
    least_conn;
    server 104.XX.XX.01:443 weight=3 max_fails=2 fail_timeout=30s;
    server 104.XX.XX.02:443 weight=2 max_fails=2 fail_timeout=30s;
    server 104.XX.XX.03:443 weight=1 max_fails=2 fail_timeout=30s;
    server 104.XX.XX.99:443 backup;  # 备用节点
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate     /etc/ssl/certs/yourdomain.crt;
    ssl_certificate_key /etc/ssl/private/yourdomain.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    # 健康检查端点
    location /health {
        access_log off;
        return 200 "OK";
        add_header Content-Type text/plain;
    }

    # 主业务代理
    location / {
        proxy_pass         https://vps_backend;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_connect_timeout 5s;
        proxy_next_upstream error timeout http_502 http_503 http_504;
    }
}
```

---

## CDN 集成与流量调度

### CDN 在高可用架构中的作用

- **就近访问**：用户从最近的 CDN 边缘节点获取内容，减少延迟
- **源站保护**：真实源站 IP 被隐藏，抵御直接攻击
- **带宽减压**：静态资源由 CDN 分担，源站负载大幅降低
- **DDoS 缓解**：CDN 节点吸收恶意流量，清洗后回源

### CDN 回源健康检查脚本（PowerShell）

```powershell
# CDN 源站健康检查与切换脚本
# 适用：Cloudflare / 阿里云 CDN / AWS CloudFront 回源管理

$SourceOrigins = @(
    @{ Name="源站-01 (美西)"; Host="origin01.yourdomain.com"; IP="104.XX.XX.01"; Port=443; Status="ACTIVE" },
    @{ Name="源站-02 (亚太)"; Host="origin02.yourdomain.com"; IP="104.XX.XX.02"; Port=443; Status="ACTIVE" },
    @{ Name="源站-03 (欧洲)"; Host="origin03.yourdomain.com"; IP="104.XX.XX.03"; Port=443; Status="BACKUP" }
)

function Test-OriginHealth {
    param($Origin)
    try {
        $resp = Invoke-WebRequest -Uri "https://$($Origin.Host)/health" -Method GET -TimeoutSec 6 -UseSSL -ErrorAction SilentlyContinue
        if ($resp.StatusCode -eq 200) { $Origin.Status = "ACTIVE"; return $true } else { $Origin.Status = "DEGRADED"; return $false }
    } catch { $Origin.Status = "DOWN"; return $false }
}

Write-Host "========== CDN 源站健康检查 ==========" -ForegroundColor Cyan
foreach ($origin in $SourceOrigins) {
    $healthy = Test-OriginHealth -Origin $origin
    $color = if ($healthy) { "Green" } else { "Red" }
    Write-Host "  $($origin.Name): $($origin.Status)" -ForegroundColor $color
}

$active = $SourceOrigins | Where-Object { $_.Status -eq "ACTIVE" }
Write-Host "`n当前活跃源站: $($active.Count) / $($SourceOrigins.Count)" -ForegroundColor Cyan

# Cloudflare API 切换主源站（示例）
$CF_API_TOKEN = $env:CF_API_TOKEN
$CF_ZONE = "yourzoneid"
if ($active.Count -gt 0) {
    $primary = $active[0]
    Write-Host "Cloudflare 主源站: $($primary.Host)" -ForegroundColor Yellow
    # Invoke-RestMethod -Uri "https://api.cloudflare.com/client/v4/zones/${CF_ZONE}/policies/load_balancers/..." ...
}
```

---

## DNS Failover 自动化

### 主流 DNS Failover 方案对比

| 服务商 | 产品 | 健康检查方式 | 切换速度 | 价格 |
|--------|------|------------|---------|------|
| **AWS Route53** | Health Check + DNS Failover | HTTP/HTTPS/TCP | 30秒内 | 按检查次数计费 |
| **Cloudflare** | Health Check (Pro+ Plan) | HTTP/HTTPS | ~60秒 | Pro 计划起 |
| **阿里云 DNS** | 健康检查 + 智能解析 | HTTP/TCP/ICMP | ~60秒 | 企业版 |
| **NS1** | Filter Chains + Health Checks | HTTP/HTTPS | 可配置 | 企业级 |
| **DNSPod** | 负载均衡 + 健康检查 | HTTP/TCP | ~60秒 | 按量 |

### Route53 DNS Failover 配置 (CLI)

```bash
# 创建健康检查
aws route53 create-health-check \
    --caller-reference $(date +%s) \
    --config '{
        "FullyQualifiedDomainName": "primary.yourdomain.com",
        "Type": "HTTPS",
        "Port": 443,
        "ResourcePath": "/health",
        "RequestInterval": 10,
        "FailureThreshold": 3
    }'

# 创建故障转移记录
aws route53 change-resource-record-sets \
    --hosted-zone-id Z0123456789ABCDEF \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "yourdomain.com", "Type": "A",
                "SetIdentifier": "primary", "Failover": "PRIMARY",
                "HealthCheckId": "hcp-abc123xyz",
                "TTL": 60, "ResourceRecords": [{"Value": "104.XX.XX.01"}]
            }
        }, {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "yourdomain.com", "Type": "A",
                "SetIdentifier": "secondary", "Failover": "SECONDARY",
                "TTL": 60, "ResourceRecords": [{"Value": "104.XX.XX.02"}]
            }
        }]
    }'
```

### PowerDNS DNS Failover 自动切换（PowerShell）

```powershell
# PowerDNS DNS Failover 自动切换脚本
$PDNS_API_URL = "http://127.0.0.1:8081/api/v1"
$PDNS_API_KEY = "YOUR_PDNS_API_KEY"
$DOMAIN       = "yourdomain.com"
$RECORD_NAME  = "api"
$PRIMARY_IP   = "104.XX.XX.01"
$SECONDARY_IP = "104.XX.XX.02"
$CHECK_URL    = "https://${PRIMARY_IP}/health"
$FAIL_THRESHOLD = 3
$failCount = 0

function Invoke-PDNSRequest {
    param($Method, $Endpoint, $Body = $null)
    $headers = @{ "X-API-Key"=$PDNS_API_KEY; "Content-Type"="application/json" }
    $params = @{ Method=$Method; Uri="${PDNS_API_URL}${Endpoint}"; Headers=$headers; TimeoutSec=10 }
    if ($Body) { $params.Body = ($Body | ConvertTo-Json -Compress) }
    try { Invoke-RestMethod @params } catch { Write-Host "PDNS API 错误: $_" -ForegroundColor Red; return $null }
}

function Set-FailoverRecord {
    param($TargetIP, $Reason)
    $patchBody = @{ rrsets = @(@{
        name = "${RECORD_NAME}.${DOMAIN}"
        type = "A"
        ttl  = 60
        records = @(@{ content=$TargetIP; disabled=$false })
    })} | ConvertTo-Json -Compress
    $result = Invoke-PDNSRequest -Method PATCH -Endpoint "/servers/localhost/zones/${DOMAIN}." -Body $patchBody
    if ($result) {
        Write-Host "DNS 记录已切换: ${RECORD_NAME}.${DOMAIN} → ${TargetIP}" -ForegroundColor Yellow
        Write-Host "原因: $Reason" -ForegroundColor Yellow
    }
}

while ($true) {
    $ok = $false
    try {
        $resp = Invoke-WebRequest -Uri $CHECK_URL -TimeoutSec 6 -UseSSL -ErrorAction SilentlyContinue
        if ($resp.StatusCode -eq 200) { Write-Host "[$(Get-Date -Format 'HH:mm:ss')] 主节点正常 ✓" -ForegroundColor Green; $ok = $true }
        else { Write-Host "[$(Get-Date -Format 'HH:mm:ss')] 主节点异常 (HTTP $($resp.StatusCode))" -ForegroundColor Red }
    } catch { Write-Host "[$(Get-Date -Format 'HH:mm:ss')] 主节点不可达: $_" -ForegroundColor Red }

    if ($ok) { $failCount = 0 } else { $failCount++ }

    if ($failCount -ge $FAIL_THRESHOLD) {
        Write-Host "触发故障转移！连续失败 ${failCount} 次" -ForegroundColor Red
        Set-FailoverRecord -TargetIP $SECONDARY_IP -Reason "主节点连续 ${failCount} 次检测失败"
        $failCount = 0
    }
    Start-Sleep -Seconds 10
}
```

---

## Auto-Scaling 自动扩缩容

### 弹性扩缩容核心指标

| 指标 | 说明 | 扩容阈值建议 | 缩容阈值建议 |
|------|------|------------|------------|
| **CPU 利用率** | 节点 CPU 占用 | > 70% (持续 3min) | < 30% (持续 10min) |
| **内存利用率** | 节点内存占用 | > 80% (持续 2min) | < 40% (持续 10min) |
| **请求队列长度** | 等待处理的请求数 | > 100 (持续 1min) | < 20 (持续 5min) |
| **网络吞吐量** | 入口/出口带宽 | > 80% 带宽 | < 30% 带宽 |
| **响应延迟 (P99)** | 99 分位响应时间 | > 500ms | < 100ms |

### 自动扩缩容脚本（Bash）

```bash
#!/bin/bash
# autoscale.sh - VPS 节点自动扩缩容脚本
MIN_NODES=2; MAX_NODES=10; CURRENT_NODES=2
CPU_SCALE_UP=70; CPU_SCALE_DOWN=30
COOLDOWN_UP=300; COOLDOWN_DOWN=600

log() { echo "[$(date '+%H:%M:%S')] $1"; }

create_vps_node() {
    local node_id=$(uuidgen | tr '[:upper:]' '[:lower:]')
    echo "创建 VPS: $node_id"
    # aws ec2 run-instances --image-id ami-xxxx --instance-type t3.medium ...
    echo $node_id
}

terminate_vps_node() {
    local node_id=$1
    echo "终止 VPS: $node_id"
    # aws ec2 terminate-instances --instance-ids $node_id ...
}

get_avg_cpu() {
    # promtool query instant --server=http://prometheus:9090 \
    #   'avg(rate(node_cpu_seconds_total[5m])) * 100'
    echo 65
}

scale_up() {
    [[ $CURRENT_NODES -ge $MAX_NODES ]] && { log "已达最大节点数 ($MAX_NODES)"; return; }
    create_vps_node; ((CURRENT_NODES++))
    log "扩容成功: 当前节点数 = $CURRENT_NODES"
    # nginx -s reload
}

scale_down() {
    [[ $CURRENT_NODES -le $MIN_NODES ]] && { log "已达最小节点数 ($MIN_NODES)"; return; }
    local node_to_remove="104.XX.XX.10"
    echo "驱逐并下线节点: $node_to_remove"
    terminate_vps_node $node_to_remove
    ((CURRENT_NODES--))
    log "缩容成功: 当前节点数 = $CURRENT_NODES"
}

log "========== Auto-Scaling 监控启动 =========="
while true; do
    cpu=$(get_avg_cpu)
    log "平均 CPU: ${cpu}% | 节点数: ${CURRENT_NODES}"
    if (( $(echo "$cpu > $CPU_SCALE_UP" | bc -l) )); then
        log "→ 触发扩容 (CPU ${cpu}% > ${CPU_SCALE_UP}%)"
        scale_up; sleep $COOLDOWN_UP
    elif (( $(echo "$cpu < $CPU_SCALE_DOWN" | bc -l) )); then
        log "→ 触发缩容 (CPU ${cpu}% < ${CPU_SCALE_DOWN}%)"
        scale_down; sleep $COOLDOWN_DOWN
    fi
    sleep 30
done
```

---

## 灾备恢复 RTO/RPO

### 典型 RTO/RPO 目标

| 业务等级 | RTO | RPO | 方案示例 |
|---------|-----|-----|---------|
| 核心交易系统 | < 15 分钟 | < 1 分钟 | 同步双活 + 实时复制 |
| 关键业务系统 | < 1 小时 | < 15 分钟 | 异步复制 + 每15分钟快照 |
| 普通业务系统 | < 4 小时 | < 1 小时 | 每日备份 + 异步复制 |
| 非关键系统 | < 24 小时 | < 24 小时 | 每日备份 |

### 灾备恢复脚本（PowerShell）

```powershell
# ==============================================================
# 灾备恢复脚本 - VPS 故障切换与数据恢复
# ==============================================================

function Invoke-DisasterRecovery {
    param(
        [string]$RecoveryType = "Auto",
        [int]$RTOBudgetMinutes = 60
    )

    $sw = [Diagnostics.Stopwatch]::StartNew()

    Write-Host "========================================" -ForegroundColor Red
    Write-Host "  🚨 灾备恢复流程启动 | 类型: $RecoveryType | RTO 预算: ${RTOBudgetMinutes} 分钟" -ForegroundColor Red
    Write-Host "========================================" -ForegroundColor Red

    # [阶段 1/5] 故障确认
    Write-Host "`n[1/5] 故障确认中..." -ForegroundColor Cyan
    $primaryFail = $true  # 实际实现: 检查主站点健康状态
    if (-not $primaryFail) {
        Write-Host "主站点正常，取消恢复" -ForegroundColor Green; return
    }
    Write-Host "✓ 主站点故障已确认" -ForegroundColor Green

    # [阶段 2/5] 切换 DNS
    Write-Host "`n[2/5] 切换 DNS 至备用站点..." -ForegroundColor Cyan
    Switch-DNSFailover -Target "DrSite-Asia"  # 实际实现: 调用 DNS API
    Write-Host "✓ DNS 已切换（TTL 约 60 秒生效）" -ForegroundColor Green

    # [阶段 3/5] 启动备用节点
    Write-Host "`n[3/5] 启动备用节点..." -ForegroundColor Cyan
    foreach ($ip in @("104.XX.XX.101", "104.XX.XX.102")) {
        Start-BackupNode -IP $ip  # 实际实现: 调用 VPS API 启动
        Write-Host "✓ 备用节点 $ip 已启动" -ForegroundColor Green
    }

    # [阶段 4/5] 数据一致性校验
    Write-Host "`n[4/5] 数据一致性校验..." -ForegroundColor Cyan
    $rpoActual = 12  # 实际实现: 比较主备数据时间戳
    Write-Host "✓ 数据完整性检查通过 (RPO: ${rpoActual} 分钟)" -ForegroundColor Green

    # [阶段 5/5] 业务验证
    Write-Host "`n[5/5] 业务验证测试..." -ForegroundColor Cyan
    Write-Host "✓ API / DB / Cache 验证通过" -ForegroundColor Green

    $sw.Stop()
    $rtoActual = [math]::Round($sw.Elapsed.TotalMinutes, 1)

    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "  灾备恢复完成" -ForegroundColor Green
    Write-Host "  实际 RTO: ${rtoActual} 分钟 | RTO 预算: ${RTOBudgetMinutes} 分钟" -ForegroundColor White
    $status = if ($rtoActual -le $RTOBudgetMinutes) { "✅ 达标" } else { "❌ 超限" }
    Write-Host "  RTO 状态: $status" -ForegroundColor $(if ($status -eq "✅ 达标") { "Green" } else { "Red" })
    Write-Host "========================================" -ForegroundColor Cyan
}

Invoke-DisasterRecovery -RecoveryType "Auto" -RTOBudgetMinutes 60
```

---

## DDoS 防护策略

### 多层 DDoS 防护架构

```
[互联网流量]
    ↓
┌──────────────────┐
│ Layer 3/4 防护    │ ← 运营商级黑洞路由、Anycast 分散
│ (Cloudflare / AWS Shield / 阿里云高防) │
└────────┬─────────┘
         ↓ 清洗后合法流量
┌──────────────────┐
│ Layer 7 WAF       │ ← 应用防火墙 CC 防护 / 速率限制
└────────┬─────────┘
         ↓
┌──────────────────┐
│ 负载均衡器         │ ← Nginx / HAProxy 限速
└────────┬─────────┘
         ↓
    [后端 VPS 集群]
```

### Nginx 速率限制配置

```nginx
# /etc/nginx/conf.d/rate_limit.conf

limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
limit_req_zone $binary_remote_addr zone=login_limit:10m rate=5r/m;
limit_req_zone $binary_remote_addr zone=global:10m   rate=1000r/s;

# IP 白名单
geo $limit {
    default        1;
    10.0.0.0/8     0;
    172.16.0.0/12  0;
    192.168.0.0/16 0;
    127.0.0.1      0;
}

map $limit $limit_key { 0 ""; 1 $binary_remote_addr; }

server {
    location / {
        limit_req zone=global burst=2000 nodelay;
    }
    location /api/ {
        limit_req zone=api_limit burst=50 nodelay;
        limit_req_status 429;
        proxy_pass https://vps_backend;
    }
    # 登录接口严格限速
    location /api/login {
        limit_req zone=login_limit burst=3 nodelay;
        limit_req_status 429;
        proxy_pass https://vps_backend;
    }
    error_page 429 = @rate_limit_exceeded;
    location @rate_limit_exceeded {
        return 429 '{"error":"rate_limit_exceeded","retry_after":60}';
        add_header Content-Type application/json;
    }
}
```

---

## 实战脚本库

| 脚本 | 平台 | 功能 |
|------|------|------|
| `health_check.ps1` | Windows | 多节点健康检查 + 故障转移通知 |
| `failover.sh` | Linux | Nginx Upstream 动态故障转移 |
| `autoscale.sh` | Linux | 自动扩缩容脚本 |
| `dns_failover.ps1` | Windows | PowerDNS DNS Failover |
| `dns_failover.sh` | Linux | Route53 DNS Failover |
| `sla_calculator.ps1` | Windows | SLA 可用性计算工具 |
| `cdn_origin_check.ps1` | Windows | CDN 源站健康检查 |
| `disaster_recovery.ps1` | Windows | 灾备恢复流程脚本 |

> **更多脚本持续更新中，欢迎提交 PR！**

---

## 经典案例分析

### 案例 1：电商平台多区域高可用部署

**背景**：某中型电商平台，面向亚太用户，峰值 QPS 约 5000，期望 99.9% SLA。

**架构方案**：

```
[全球 CDN (Cloudflare)] → [静态资源缓存]
                              ↓
[API Gateway (美西 + 亚太双活)]
        ↓                        ↓
[美西 VPS 集群 (3节点)]   [亚太 VPS 集群 (3节点)]
        ↑                        ↑
   [主数据库] ←── 同步复制 ──→ [备数据库]
```

**SLA 计算**：
- 单节点月故障率约 0.1%（约 43.8 分钟宕机/月）
- 3 节点集群：3 个节点同时故障概率 ≈ 0.1%³ = 0.000001%
- 实际可用性 ≈ **99.999%**（远超 99.9% 目标）

**实际 RTO/RPO**：
- 故障检测：10 秒
- 自动切换：< 30 秒
- 业务恢复：< 60 秒（RTO < 1 分钟）
- 数据同步：< 1 秒（RPO = 0）

**关键决策**：
1. 选择**双活**而非主备：亚太用户延迟从 150ms 降至 30ms
2. 使用 **CockroachDB** 跨区域分布式数据库：强一致 + 低延迟
3. Cloudflare Argo Tunnel 隐藏源站 IP：免受定向攻击

---

### 案例 2：金融 API 服务 99.99% SLA 实践

**背景**：金融数据 API 服务，面向机构客户，承诺 99.99% SLA（年宕机 ≤ 52.6 分钟）。

**技术方案**：

| 组件 | 方案 | 容灾措施 |
|------|------|---------|
| 接入层 | Anycast + BGP AnyCast | 单一 IP，全球最近接入 |
| 负载均衡 | AWS ALB + NLB 双层 | 跨 AZ 自动分散 |
| 应用层 | ECS Fargate 多 AZ | 自动恢复，故障零接触 |
| 数据库 | RDS Multi-AZ（同步） | 主备自动切换 < 60s |
| 缓存 | ElastiCache Redis 集群 | 3 主分片 + 副本 |
| 监控 | CloudWatch + Grafana | P1 告警 < 1 分钟 |
| 备份 | RDS 自动备份（5分钟粒度） | PITR 恢复 |

**达成 99.99% SLA 的关键**：
1. **主动式容灾**：多 AZ 部署确保单 AZ 故障不影响服务
2. **数据库同步复制**：主库故障自动切换，RPO ≈ 0
3. **连接池 + 熔断**：防止数据库故障引发级联崩溃
4. **混沌工程**：每月进行故障注入演练，验证恢复能力
5. **容量预警**：CPU/内存超 70% 自动扩容，避免过载宕机

---

### 案例 3：个人 VPS 集群的高可用化改造

**背景**：个人开发者，3 台低配 VPS（2核/4G），运行个人项目（博客 + API + 代理），原可用性约 95%。

**改造方案（低成本高可用）**：

| 改造项 | 原方案 | 改造后 | 成本增加 |
|--------|--------|--------|---------|
| 健康检查 | 无 | 每 15 秒自动检测 | 免费 |
| 故障转移 | 手动重启 | Nginx 自动摘除故障节点 | 免费 |
| 数据备份 | 手动 | 每日 Rsync 增量备份到对象存储 | ¥5/月 |
| DNS | 单 A 记录 | DNSPod 智能解析 + 故障切换 | 免费 |
| 监控告警 | 无 | UptimeRobot 免费监控 | 免费 |
| **综合可用性** | **~95%** | **~99.5%** | **~¥5/月** |

**结果**：年度宕机时间从 ~18 天降至 ~1.8 天，SLA 从 95% 提升至 99.5%，月成本仅增加约 5 元（对象存储备份）。

> **VPSVIP 推荐**：低成本高可用方案，推荐选择支持 API 重装的 VPS 服务商，便于自动化扩缩容。  
> 推荐导航：[vpsvip.net](https://vpsvip.net) | [nav.clashvip.net](https://nav.clashvip.net)

---

## FAQ 常见问题

### Q1: 如何选择合适的 SLA 等级？

**A**: 主要考虑三个维度：
- **业务影响**：宕机造成的收入损失和用户流失
- **合规要求**：金融、医疗等行业有强制性可用性标准
- **成本预算**：SLA 越高，对应基础设施成本越高

| 业务场景 | 推荐 SLA |
|---------|---------|
| 核心交易系统 | 99.99%+ |
| 关键业务服务 | 99.9%+ |
| 普通 Web 应用 | 99.5%+ |
| 内部工具 / 非关键系统 | 99%+ |

### Q2: 多区域部署会增加多少成本？

**A**: 成本增加取决于架构设计：
- **同 Region 多 AZ**：通常增加 30%~50%（备用 AZ 资源费用）
- **跨 Region 双活**：通常增加 80%~150%（双倍基础设施 + 跨区域流量费）
- **优化建议**：从同 Region 多 AZ 开始，根据业务需求逐步扩展

### Q3: 健康检查频率如何设置？

**A**: 建议平衡**故障检测速度**与**资源消耗**：

| 检查类型 | 推荐频率 | 说明 |
|---------|---------|------|
| 关键业务（支付、核心 API）| 5~10 秒 | 快速检测，快速切换 |
| 普通业务（Web、App 后端）| 15~30 秒 | 平衡检测速度与 API 成本 |
| 非关键业务 | 60 秒+ | 降低检查开销 |

**注意**：健康检查过于频繁（如每秒）可能产生额外网络开销和误判。

### Q4: 如何避免"惊群效应"（Thundering Herd）？

**A**: 故障转移时多个备用节点同时收到大量请求导致过载：

1. **渐进式恢复**：备用节点逐步接入，逐步承接流量
2. **连接队列**：负载均衡器在切换期间缓冲请求
3. **容量预留**：备用节点平时保持 30% 以上空闲容量
4. **熔断保护**：备用节点负载过高时主动拒绝新请求

### Q5: 如何验证高可用架构的有效性？

**A**: **混沌工程**是验证高可用架构的最佳方式：

| 演练类型 | 验证内容 | 推荐工具 |
|---------|---------|---------|
| **实例终止** | 单节点故障自动恢复 | Chaos Monkey |
| **网络故障** | 网络分区时的行为 | Toxiproxy / Gremlin |
| **数据库故障** | 主从切换 / 数据一致性 | chaosblade |
| **全区域故障** | DR 切换 RTO / RPO | 手动演练（生产环境慎用）|

> ⚠️ **重要提醒**：生产环境混沌演练前务必制定回滚方案和人员值守。

### Q6: 如何选择 VPS 的高可用配置？

**A**: 选择 VPS 时关注以下高可用相关特性：

- ✅ **多可用区（Multi-AZ）支持**
- ✅ **API 控制（重启、重装、快照）**
- ✅ **快照 / 镜像功能**
- ✅ **内网 VPC / 私有网络**
- ✅ **BGP 动态带宽（低延迟路由）**
- ✅ **99.9%+ SLA 承诺**

> **推荐配置**：生产环境至少 2 台 VPS 分布在不同机房，最低配置 1核/2G 以上。  
> VPS 推荐导航：[clashvip.net](https://clashvip.net) | [nav.clashvip.net](https://nav.clashvip.net)

---

---

## 附录 A · 存储退化与磁盘故障的早期识别

高可用架构里最容易被忽略的一环，是**存储不会突然坏，它只会慢慢坏**。CPU、内存、网络故障通常是"断崖式"的（进程崩、连接超时），而磁盘故障是"斜坡式"的：延迟先涨 3 倍，再涨 10 倍，重映射扇区缓慢增多，最后某天 ext4 把你 remount 成只读，或者一次 scrub 报出校验错误。RAID 与副本能扛住"整块盘消失"，却扛不住**静默损坏**和**同时退化**。

这一附录给的是"斜坡上的刻度"——如何在业务还没感觉到的时候，就从指标里看出盘要坏了。

### A.1 为什么 RAID 和快照都不能替代这一节

| 认知 | 现实 |
|------|------|
| "有 RAID1，坏一块没事" | 重建期间是单点，且重建本身就是全盘顺序读，最容易把同批次的另一块盘读挂 |
| "有快照，坏了回滚" | 快照与源盘同物理介质，介质级损坏会一起丢 |
| "云盘有副本，服务商保证 99.999999" | 保证的是"不丢"，不保证"不慢"。退化期的 IO 延迟上升照样打穿 SLA |
| "SMART 全 PASS 就是健康" | SMART 的 PASS 阈值极宽松，很多盘在 PASS 状态下延迟已劣化 5 倍 |
| "监控有磁盘使用率告警就够了" | 空间告警只在最后 1% 才触发，与"是否退化"毫无关系 |

结论：**容量告警解决"写满"，SMART 趋势解决"写坏"，延迟分布解决"写慢"**，三者缺一不可。

### A.2 SMART 关键属性：只看这几个就够

`smartctl -a` 输出上百行，真正有判读价值的是下面几项。注意 SATA 与 NVMe 的语义完全不同：

**SATA / SAS 盘**

| ID | 名称 | 判读方式 |
|----|------|---------|
| 5 | Reallocated_Sector_Ct | **绝对值 + 斜率**。> 0 就要记录；一周内增长 ≥ 8 建议换盘 |
| 187 | Reported_Uncorrect | 不可纠正错误，**非 0 即高危**，说明已有读失败 |
| 188 | Command_Timeout | 持续增长说明链路/固件问题，常见于线材或背板 |
| 197 | Current_Pending_Sector | 待重映射（读不出来的扇区），**非 0 即需立即备份** |
| 198 | Offline_Uncorrectable | 离线扫描发现的坏块，与 197 联动看 |
| 199 | UDMA_CRC_Error_Count | 纯链路问题（线、接口），换线往往直接归零 |
| 9 | Power_On_Hours | 用于判断是否进入故障高发期（机械盘 > 4 万小时需提高关注） |
| 241/242 | Total_LBAs_Written | 换算 TBW 消耗，判断 SSD 寿命余量 |

**NVMe 盘**（`smartctl -a /dev/nvme0` 或 `nvme smart-log`）

| 字段 | 判读方式 |
|------|---------|
| `percentage_used` | 寿命消耗百分比。> 80% 开始规划更换；注意它对写入量非线性 |
| `available_spare` / `available_spare_threshold` | 备用块余量。**低于阈值 = 盘已进入只读保护倒计时** |
| `media_errors` | 介质错误数，**非 0 就要查**（对应 SATA 的 197/198） |
| `num_err_log_entries` | 错误日志条数，短时间快速增长说明控制器异常 |
| `unsafe_shutdowns` | 非正常掉电次数，异常高说明电源/虚机被强杀，是隐性风险 |
| `critical_warning` | 位图，非 0 时逐位查手册，通常意味着"立即更换" |

一段可直接放进 cron 的轻量巡检脚本，只输出"需要人看"的盘：

```bash
#!/usr/bin/env bash
# /usr/local/bin/disk-health.sh — 只报告异常项，退出码非 0 表示有盘需要处理
set -uo pipefail
ALERT=0

for dev in $(lsblk -dno NAME,TYPE | awk '$2=="disk"{print $1}'); do
  path="/dev/$dev"
  # 跳过不支持 SMART 的虚拟盘（virtio 通常无 SMART）
  if ! smartctl -i "$path" >/dev/null 2>&1; then continue; fi

  out=$(smartctl -H -A "$path" 2>/dev/null) || true
  health=$(echo "$out" | awk -F': *' '/SMART overall-health/{print $2}')
  realloc=$(echo "$out" | awk '$1=="5"{print $10}')
  pending=$(echo "$out" | awk '$1=="197"{print $10}')
  uncorr=$(echo "$out"  | awk '$1=="198"{print $10}')

  realloc=${realloc:-0}; pending=${pending:-0}; uncorr=${uncorr:-0}
  if [[ "$health" != "PASSED" ]] || (( realloc > 0 || pending > 0 || uncorr > 0 )); then
    printf 'ALERT %s health=%s realloc=%s pending=%s uncorr=%s\n' \
      "$path" "${health:-UNKNOWN}" "$realloc" "$pending" "$uncorr"
    ALERT=1
  fi
done

# NVMe 单独判读
for nv in /dev/nvme[0-9]n[0-9]; do
  [[ -e "$nv" ]] || continue
  used=$(smartctl -A "$nv" 2>/dev/null | awk -F': *' '/Percentage Used/{gsub(/[^0-9]/,"",$2);print $2}')
  spare=$(smartctl -A "$nv" 2>/dev/null | awk -F': *' '/Available Spare:/{gsub(/[^0-9]/,"",$2);print $2}')
  media=$(smartctl -A "$nv" 2>/dev/null | awk -F': *' '/Media and Data Integrity Errors/{print $2+0}')
  if [[ -n "${used:-}" ]] && (( used > 80 )); then
    echo "ALERT $nv percentage_used=${used}%"; ALERT=1
  fi
  if [[ -n "${media:-}" ]] && (( media > 0 )); then
    echo "ALERT $nv media_errors=${media}"; ALERT=1
  fi
done

exit $ALERT
```

**关键心态**：SMART 是"趋势指标"不是"健康证明"。把它当作体温计——37.2℃ 不代表生病，但连续三天往上走就必须查。

### A.3 从绝对值到斜率：把变化率写进监控

单次读数是噪声，**一周内的增长量**才是信号。用 `smartctl_exporter` 采集到 Prometheus 后，用 `deriv()` 算斜率：

```yaml
# promql: 重映射扇区 7 天增长率（正数代表在恶化）
deriv(smartctl_device_attribute{attribute_name="Reallocated_Sector_Ct"}[7d]) * 86400 * 7
```

告警规则建议（数值按自有盘群的历史基线校准，不要照抄）：

```yaml
groups:
  - name: storage-degradation
    rules:
      - alert: DiskReallocatedGrowing
        expr: increase(smartctl_device_attribute{attribute_name="Reallocated_Sector_Ct"}[7d]) > 8
        for: 6h
        labels: { severity: warning }
        annotations:
          summary: "{{ $labels.instance }} 重映射扇区 7 天内增长 > 8，疑似加速退化"

      - alert: DiskPendingSectors
        expr: smartctl_device_attribute{attribute_name="Current_Pending_Sector"} > 0
        for: 10m
        labels: { severity: critical }
        annotations:
          summary: "{{ $labels.instance }} 存在待重映射扇区，立即备份并计划换盘"

      - alert: NvmeSpareLow
        expr: smartctl_nvme_available_spare < smartctl_nvme_available_spare_threshold
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "{{ $labels.instance }} NVMe 备用块低于阈值，即将进入只读模式"

      - alert: DiskLatencyHigh
        expr: rate(node_disk_write_time_seconds_total[5m]) / rate(node_disk_writes_completed_total[5m]) > 0.05
        for: 15m
        labels: { severity: warning }
        annotations:
          summary: "{{ $labels.device }} 平均单次写耗时 > 50ms，检查是否退化"
```

### A.4 延迟分布才是业务感受到的"坏"

吞吐（MB/s）好看但 P99 很差，是退化盘的典型画像。做基线时不要只记平均 IOPS：

```bash
# 4K 随机读，QD1（模拟单线程数据库），重点看 clat p99/p999 的分布
fio --name=r4k-qd1 --ioengine=libaio --direct=1 --rw=randread \
    --bs=4k --iodepth=1 --size=1G --runtime=60 --time_based \
    --percentile_list=50:90:99:99.9 --group_reporting

# 4K 随机写，QD32（模拟高并发）
fio --name=w4k-qd32 --ioengine=libaio --direct=1 --rw=randwrite \
    --bs=4k --iodepth=32 --numjobs=4 --size=1G --runtime=60 --time_based \
    --percentile_list=50:90:99:99.9 --group_reporting
```

判读要点：

- **QD1 延迟**是数据库单条事务的真实体验，NVMe 正常 < 0.2 ms，超过 1 ms 就值得查
- **QD32 与 QD1 的扩展比**：健康盘约 5–15 倍；若扩展比接近 1（并发上去延迟不降），说明队列已饱和或盘在内部重试
- **p999 突然抬升而 p50 不变**，几乎一定是介质重试，不是应用问题
- 每次测完把结果追加到基线文件，用两列对比（本期 vs 上期），比看单次数字有用得多

```bash
# 把 fio 的关键指标抽成一行，方便长期对比
fio ... --output-format=json | jq -r '
  [.jobs[0].read.clat_ns.percentile["99.000000"], .jobs[0].read.iops] | @tsv' \
  | awk -v d="$(date -F +%F)" '{printf "%s\tp99=%.3fms\tiops=%.0f\n", d, $1/1e6, $2}' \
  >> /var/log/disk-baseline.tsv
```

### A.5 两种"盘没坏但服务挂了"的经典场景

**场景一：inode 耗尽。** `df -h` 显示还有 40% 空间，但应用报 `No space left on device`。原因是小文件把 inode 用完了。

```bash
df -i                       # IUse% 才是真相
# 定位元凶：找出单目录文件数最多的路径
for d in /var/* /srv/* /home/*; do
  printf '%8s %s\n' "$(find "$d" -xdev -type f 2>/dev/null | wc -l)" "$d"
done | sort -rn | head -10
```

常见元凶：session 文件、未轮转的日志分片、`/var/spool/postfix` 积压、`/tmp` 里的上传临时文件、Docker 的 overlay2 层（虽然层数有限，但每层内的文件会计入宿主 inode）。ext4 扩容可以顺带加 inode（`mkfs.ext4 -i`），但**已格式化的分区无法在线增加 inode 总数**——这是选型时就要考虑的事。

**场景二：文件系统自己切成只读。** ext4 在遇到元数据不一致时会按 `errors=remount-ro` 保护性重挂载；XFS 更激进，一旦检测到结构损坏直接 shutdown。

```bash
# 确认是否被 remount 为只读
mount | grep -E ' / | /srv '     # 看是否带 ro
dmesg -T | grep -iE 'EXT4-fs error|XFS .*shutdown|I/O error|remount' | tail -20
```

应急顺序（**先保住数据，再恢复服务**）：

1. 立即停止写入类业务，避免在半坏的文件系统上继续扩大损坏面
2. 把能读的关键数据先拷出来（`rsync` 到另一块盘/对象存储），不要先 fsck
3. 卸载后 `fsck -n` 只读检查，看清损坏范围；必要时 `fsck -y`（此操作可能丢文件，务必先有备份）
4. XFS 用 `xfs_repair -n` 预检，再 `xfs_repair`；日志损坏时可能需要 `-L` 清日志（**会丢未落盘数据**）
5. 恢复后立刻做全量备份 + SMART 判读，决定是否换盘

> **纪律**：只读重挂载是内核在替你止损，不是故障本身。此时最危险的动作是"赶紧 remount rw 继续跑"。

### A.6 dmesg 关键词速查

| 关键词 | 含义 | 处置优先级 |
|--------|------|-----------|
| `I/O error, dev sdX, sector NNN` | 该扇区读失败 | 高：立即确认是否重复出现 |
| `blk_update_request: I/O error` | 块层错误，常伴随上一层 | 高：结合上面定位盘 |
| `EXT4-fs error (device sdX): ...` | 文件系统层错误 | 高：准备取证与迁移 |
| `XFS (sdX): Corruption detected` | XFS 结构损坏，随后 shutdown | 紧急：先拷数据 |
| `nvme0: I/O NNN QID N timeout` | NVMe 超时，常见于盘过热或固件 | 高：查温度与固件 |
| `ataX: hard resetting link` | 链路重置，多为线材/背板 | 中：换线验证 |
| `medium error` | 介质级错误（而非链路） | 高：介质在坏 |
| `Buffer I/O error on dev` | 上层文件系统的 IO 失败 | 高 |

建议把这组词做成 journal 告警，不要等出事才 `dmesg | grep`：

```bash
journalctl -k -p err --since "24 hours ago" | grep -icE 'I/O error|medium error|hard resetting'
```

### A.7 决策树：什么时候"忍"，什么时候"换"

```
发现异常指标
├─ Current_Pending_Sector > 0 或 media_errors > 0
│   └─ 立即：全量备份 → 24h 内换盘（不要等它变多）
├─ Reallocated 增长但 < 8/周，且无 pending
│   └─ 加密巡检到每日一次，30 天内计划更换；同时确认备份可恢复
├─ 仅 UDMA_CRC 增长
│   └─ 换 SATA 线/换槽位，通常可归零，不必换盘
├─ 延迟 p99 上升 3 倍但 SMART 正常
│   └─ 先排除：写放大（日志/临时表）、TRIM 未开、邻居噪声（云盘）、swap 抖动
│      排除后仍劣化 → 视为亚健康，排期更换
├─ inode 耗尽
│   └─ 清理小文件 + 加监控（df -i 告警），不改分区
└─ 文件系统只读重挂载
    └─ 按 A.5 顺序处置，恢复后强制换盘判读
```

**"同批次盘"原则**：RAID 阵列里若一块盘因介质错误退役，其余同批次、同工况的盘大概率在同一时间段进入退化期。此时应当**主动**规划更换，而不是等第二块也坏——重建窗口叠加二次故障是中小企业最常见的全量数据丢失场景。

### A.8 季度巡检清单（可与灾备演练合并执行）

- [ ] `smartctl -a` 全盘导出存档，与上季度对比重映射/待映射/CRC 三项
- [ ] NVMe 的 `percentage_used` 与 `available_spare` 记录在案
- [ ] `df -h` 与 `df -i` 双查，两者余量均 ≥ 30%
- [ ] `dmesg -T | grep -i 'i/o error'` 为空
- [ ] fio QD1/QD32 各跑一次，p99 与上季度偏差 < 30%
- [ ] 确认 `errors=remount-ro`（ext4）或备份策略（XFS）符合预期
- [ ] 阵列状态正常（`mdadm --detail` 无 degraded，`zpool status` 无 errors）
- [ ] 备份可恢复性抽验：随机取一个文件从备份中还原并比对哈希
- [ ] 记录"若今天这块盘彻底坏掉"的恢复步骤，与实际演练结果对齐

> 这一节的每一项都能在 30 分钟内完成，但它能避免的是"凌晨三点磁盘只读 + 备份也没验证过"这种最坏的组合。

---

## 推荐阅读

| 主题 | 资源 |
|------|------|
| AWS 高可用架构最佳实践 | [AWS Well-Architected - Reliability](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/) |
| Google SRE 手册 | [Site Reliability Engineering](https://sre.google/sre-book/table-of-contents/) |
| Chaos Engineering | [principlesofchaos.org](https://principlesofchaos.org/) |
| Prometheus 监控 | [prometheus.io/docs](https://prometheus.io/docs/introduction/overview/) |

---

## 相关项目

| 项目 | 描述 |
|------|------|
| [clash-for-windows](https://github.com/clashforwindows-net/Clash-for-Windows) | 主流代理客户端 |
| [nav.clashvip.net](https://nav.clashvip.net) | VPS & 代理导航门户 |

---

## 参与贡献

欢迎提交 Issue 和 Pull Request！  
- 报告 Bug 或建议新功能 → [Open an Issue](https://github.com/clashforwindows-net/vps-review-20260427/issues)
- 完善文档或添加脚本 → Fork 后提交 PR

---

## 许可证

本项目采用 [MIT License](LICENSE)。

---

> **声明**：本仓库内容仅供技术参考与学习交流，所涉及的服务商评价基于公开信息整理，不构成商业推荐。所有技术方案请根据实际业务需求评估适用性。

---

**最后更新：2026-11-07 | 含附录 A · 存储退化早期识别 | By VPSVIP & ClashVIP 推荐**

> 🌐 [nav.clashvip.net](https://nav.clashvip.net) | [clashvip.net](https://clashvip.net) | [vpsvip.net](https://vpsvip.net) | [clashhub.net](https://clashhub.net) | [bbs.clashhub.net](https://bbs.clashhub.net) | [clash-for-windows.net](https://clash-for-windows.net)
