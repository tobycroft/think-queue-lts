# think-queue-lts for ThinkPHP 8

> 面向 ThinkPHP 8.1 / PHP 8.3+ 的队列扩展包（LTS 维护版），在原版 think-queue 基础上补齐了 **Redis Stream 消费者组模式、毫秒级延迟任务调度系统、任务幂等锁与只读实时监控**，可长期运行、支持重启恢复与多实例部署。

## 功能总览

| 能力 | 说明 |
|------|------|
| 三种驱动 | `sync`（同步，开发用）/ `database`（数据库表）/ `redis`（推荐） |
| Redis 双模式 | `list`（经典列表实现，兼容原版）/ `stream`（Stream + 消费者组，推荐生产使用） |
| 延迟任务 | 秒级延迟、DateTime 指定时间、毫秒级时间戳；ZSET + HASH 存储、Lua 原子迁移、多实例安全 |
| 投递语义 | 至少一次（at-least-once）；PEL 待处理列表 + 超时重投 + 幂等锁防重复执行 |
| 实时监控 | `queue:monitor` 只读查看排队数 / 执行中 / 延迟 / worker 存活 / 吞吐速率 |
| 全 env 化配置 | 所有参数通过 `.env` 配置，无需修改配置文件即可部署 |

## 环境要求

- PHP >= 8.3（CLI 与 Web 最好使用同一版本）
- ThinkPHP >= 8.1
- Redis 驱动需要 `phpredis` 扩展；stream 模式需要 Redis 服务端 5.0+
- 延迟任务调度、实时监控为 stream 模式专属能力

## 安装

```bash
composer require tobycroft/think-queue-lts
```

### 从原版 think-queue 升级

原版与 LTS 声明了相同的命名空间和服务类，**不能同时安装**，需要替换：

```bash
# 1. 移除原版
composer remove topthink/think-queue

# 2. 安装 LTS（如 packagist 未收录，先添加 vcs 源）
composer config repositories.tqlts vcs https://github.com/tobycroft/think-queue-lts.git
composer require tobycroft/think-queue-lts

# 3. 重新发布配置（旧配置不会自动覆盖，需手动处理）
rm config/queue.php
php think vendor:publish
```

> **配置文件升级原则**：LTS 的配置是全 env 驱动的，所有实际值都在 `.env`，`config/queue.php` 只是「翻译层」。升级包之后直接整份替换该文件是安全的：
>
> ```bash
> cp vendor/tobycroft/think-queue-lts/src/config/queue.php config/queue.php
> ```

## 快速开始

`.env` 最小配置（配置了 REDIS_HOST 即自动启用 redis 驱动）：

```ini
REDIS_HOST = 10.0.0.170
REDIS_PORT = 6379
REDIS_PASSWORD = qwerty123
REDIS_SELECT = 2

[QUEUE_REDIS]
MODE = stream
```

写一个任务类并发布：

```php
namespace app\job;

use think\facade\Queue;
use think\queue\Job;

class SendMail
{
    public function fire(Job $job, $data)
    {
        // ... 执行任务
        $job->delete();   // 成功后必须删除，否则会重试
    }
}

// 立即执行
Queue::push(\app\job\SendMail::class, ['to' => 'user@example.com']);
```

启动消费者与调度器（stream 模式）：

```bash
php think queue:work redis      # 消费任务
php think queue:scheduler       # 延迟任务调度器
php think queue:monitor         # 实时查看队列状态
```

## 配置详解

### 驱动选择

`config/queue.php` 的 `default` 决定使用哪个驱动，支持自动切换：

```php
// 优先读 QUEUE_DRIVER；未设置时若 .env 配置了 REDIS_HOST 则自动用 redis，否则 sync
'default' => env('QUEUE_DRIVER') ?: (env('REDIS_HOST') ? 'redis' : 'sync'),
```

| 场景 | 效果 |
|------|------|
| `.env` 有 `QUEUE_DRIVER=database` | 强制使用 database，忽略自动切换 |
| 只有 `REDIS_HOST`（无 QUEUE_DRIVER） | 自动使用 redis |
| 什么都没配 | sync（同步执行，任务在请求内直接运行） |

### 环境变量总表

Redis 连接信息（`REDIS_*`）与缓存（`cache.php`）共用同一组环境变量，一处配置两边生效；队列特有参数使用 `QUEUE_*` 前缀：

```ini
[QUEUE]
DRIVER = redis

[QUEUE_REDIS]
MODE = stream

[REDIS]
HOST = 127.0.0.1
PORT = 6379
```

| 配置项 | 环境变量 | 默认值 | 说明 |
|--------|----------|--------|------|
| default | QUEUE_DRIVER | sync（REDIS_HOST 存在时自动 redis） | 默认驱动 |
| queue（队列名） | QUEUE_NAME | default | 默认队列名 |
| redis.mode | QUEUE_REDIS_MODE | list | **list**：列表实现；**stream**：Stream 消费者组实现 |
| redis.host | REDIS_HOST | 127.0.0.1 | 与 cache 共用 |
| redis.port | REDIS_PORT | 6379 | 与 cache 共用 |
| redis.password | REDIS_PASSWORD | (空) | 与 cache 共用 |
| redis.select | REDIS_SELECT | 0 | 建议队列使用独立库号 |
| redis.timeout | REDIS_TIMEOUT | 0 | 连接超时（秒） |
| redis.persistent | REDIS_PERSISTENT / CACHE_REDIS_PERSISTENT | false | 是否长连接 |
| redis.retry_after | QUEUE_REDIS_RETRY_AFTER | 60 | 任务执行超过该秒数未确认 → 重新投递（必须大于最长任务耗时） |
| redis.block_for | QUEUE_REDIS_BLOCK_FOR | null | pop 最长阻塞秒数，null 表示不阻塞 |
| redis.lock | QUEUE_REDIS_LOCK | true | 任务幂等锁开关 |
| redis.lock_ttl | QUEUE_REDIS_LOCK_TTL | 3600 | 幂等锁 TTL（秒） |
| redis.group | QUEUE_REDIS_GROUP | think-queue | stream：消费者组名称 |
| redis.consumer | QUEUE_REDIS_CONSUMER | (空，自动生成) | stream：消费者名称，默认 主机名-进程号 |
| redis.scheduler.batch | QUEUE_REDIS_SCHEDULER_BATCH | 100 | stream：单次迁移到期任务上限 |
| redis.scheduler.interval | QUEUE_REDIS_SCHEDULER_INTERVAL | 10 | stream：调度器兜底检查间隔（秒） |
| database.table | QUEUE_TABLE | jobs | database 驱动：任务表名 |
| database.connection | QUEUE_CONNECTION | null | database 驱动：数据库连接 |
| failed.type | QUEUE_FAILED_DRIVER | none | 失败任务驱动（none / database） |
| failed.table | QUEUE_FAILED_TABLE | failed_jobs | 失败任务表 |

### list 与 stream 模式对比

| | list 模式 | stream 模式 |
|--|-----------|-------------|
| 实现方式 | Redis List + Reserved ZSET | Redis Stream + 消费者组 |
| 可靠性 | 消费前落 reserved 集合 | PEL 待处理列表语义 |
| 超时重投 | 由 reserved 集合迁移 | `XPENDING` + `XCLAIM` 认领 |
| 延迟任务 | ZSET（member=完整 payload，秒级） | ZSET + HASH（member=job_id，毫秒级） |
| 延迟调度器 | 无需（pop 时顺带迁移） | 需要 `queue:scheduler` 进程 |
| 实时监控 | 基础三项数量 | 完整指标（PEL/消费者/吞吐） |
| 适用场景 | 简单队列、快速接入 | 生产环境、需要延迟任务与监控 |

> **切换模式注意**：两种模式的延迟任务存储格式不同，切换前先用旧模式处理完存量延迟任务。

## 创建任务类

推荐使用 `app\job` 作为任务类命名空间，也可以放在任意可自动加载的位置。任务类不需要继承任何类：

- 单任务类：提供一个 `fire` 方法
- 多任务类：提供多个方法，发布时用 `类名@方法名` 指定
- 每个任务方法接收 `think\queue\Job $job` 和发布时的 `$data` 两个参数
- 可选的 `failed($data)` 方法在任务达到最大重试次数后执行

```php
namespace app\job;

use think\queue\Job;

class SendMail
{
    public function fire(Job $job, $data)
    {
        // ... 这里执行具体的任务

        if ($job->attempts() > 3) {
            // 检查这个任务已经重试了几次
        }

        $job->delete();          // 执行成功后删除任务（必须，否则会重复执行）

        // $job->release(10);     // 或者重新发布，10 秒后再试一次
    }

    public function failed($data)
    {
        // ... 任务达到最大重试次数后失败了，可记录告警
    }
}
```

多任务类：

```php
namespace app\lib\job;

use think\queue\Job;

class Report
{
    public function daily(Job $job, $data) { /* ... */ }
    public function weekly(Job $job, $data) { /* ... */ }

    public function failed($data) { /* ... */ }
}
```

## 发布任务

```php
use think\facade\Queue;

// 立即执行（默认连接、默认队列）
Queue::push(\app\job\SendMail::class, ['to' => 'user@example.com']);

// 立即执行，指定队列
Queue::push(\app\job\SendMail::class, $data, 'mails');
Queue::pushOn('mails', \app\job\SendMail::class, $data);

// 延迟 5 秒执行（延迟秒数）
Queue::later(5, \app\job\SendMail::class, $data);

// 指定执行时间（DateTimeInterface，秒级精度）
Queue::later(new \DateTime('+10 minutes'), \app\job\SendMail::class, $data);

// 延迟执行，指定队列
Queue::laterOn('mails', 5, \app\job\SendMail::class, $data);

// 毫秒级时间戳指定执行时间（stream 模式，毫秒精度调度）
Queue::connection('redis')->laterAtMs(round(microtime(true) * 1000) + 1500, \app\job\SendMail::class, $data);
```

`$job` 的写法：

- 命名空间为 `app\job` 的类：直接写类名，如 `\app\job\SendMail` 或短名
- 其他命名空间：写完整类名
- 多任务类：`app\lib\job\Report@daily`

`$queue` 指定队列名（不同业务可拆不同队列，worker 用 `--queue=` 对应监听）。

## 运行队列

### 开发环境

```bash
php think queue:listen           # 每次请求重新引导框架（改代码即时生效）
php think queue:work             # 常驻进程（性能更好）
php think queue:work --once      # 只处理一个任务
php think queue:work redis --queue=mails --tries=3 --timeout=60 --sleep=1
```

可选参数用 `--help` 查看。

### 生产环境部署

stream 模式需要两类常驻进程：**worker（消费者）** 与 **scheduler（延迟调度器）**，由进程管理器托管，保证崩溃自动拉起、开机自启。

**Supervisor 配置**（`/etc/supervisor/conf.d/think-queue.conf`）：

```ini
[program:queue-worker]
command=php /www/wwwroot/AOSSTP8/think queue:work redis --queue=default --tries=3 --timeout=60 --sleep=1
directory=/www/wwwroot/AOSSTP8
autostart=true
autorestart=true
user=www-data
numprocs=4                       ; 并发 worker 数
process_name=%(program_name)s_%(process_num)02d
stdout_logfile=/var/log/queue-worker.log
stdout_logfile_maxbytes=10MB
stopwaitsecs=30                  ; 等待当前任务处理完再停止

[program:queue-scheduler]
command=php /www/wwwroot/AOSSTP8/think queue:scheduler redis --queue=default --interval=5
directory=/www/wwwroot/AOSSTP8
autostart=true
autorestart=true
user=www-data
numprocs=2                       ; 可多实例，Lua 原子迁移保证不重复迁移
stdout_logfile=/var/log/queue-scheduler.log
```

**systemd 模板单元**（`/etc/systemd/system/queue-worker@.service`）：

```ini
[Unit]
Description=Think Queue Worker %i
After=network.target

[Service]
User=www-data
WorkingDirectory=/www/wwwroot/AOSSTP8
ExecStart=/usr/bin/php /www/wwwroot/AOSSTP8/think queue:work redis --queue=default --tries=3 --timeout=60 --sleep=1
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

多个实例 + 统一管理入口（`/etc/systemd/system/queue.target`）：

```ini
[Unit]
Description=Think Queue (workers + scheduler)
Wants=queue-worker@1.service queue-worker@2.service queue-scheduler@1.service

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now queue-worker@1 queue-worker@2 queue-scheduler@1
systemctl enable queue.target    # 之后用 queue.target 一条命令管理全部
```

**部署后平滑重启**：PHP 常驻进程持有旧代码，每次发布必须重启 worker。内置优雅重启机制——worker 会等当前任务处理完再退出：

```bash
php think queue:restart          # 加入部署脚本最后一步，进程管理器自动用新代码拉起
```

**参数关系**（三个超时必须满足大小关系，否则会出现重复执行）：

```
lock_ttl(3600) > retry_after(60) > timeout(60) ≈ 最长任务耗时
```

| 参数 | 含义 | 约束 |
|------|------|------|
| `--timeout` | 单任务最长执行秒数，超时杀进程 | 应大于最长任务耗时 |
| `retry_after` | 执行超过该秒数未确认 → 重新投递 | 必须大于 `--timeout` |
| `lock_ttl` | 幂等锁 TTL | 必须大于 `retry_after` |

## 延迟任务调度（stream 模式）

延迟任务使用「ZSET 时间索引 + HASH 任务数据」组合存储，到期后通过 Lua 脚本原子迁移到现有 Stream，由原有消费者组和 worker 消费，投递语义为至少一次：

| Redis Key | 类型 | 说明 |
|-----------|------|------|
| `{queues:xxx}:delayed` | ZSET | 时间索引，member 为任务 job_id，score 为计划执行的 Unix 毫秒时间戳 |
| `{queues:xxx}:delayed:data` | HASH | 任务数据，field 为 job_id，value 为 payload JSON |
| `{queues:xxx}:delayed:wake` | Pub/Sub 频道 | 唤醒通知，仅用于让调度器提前重新计算等待时间，不作为可靠性依据 |

发布延迟任务：

```php
Queue::later(5, $job, $data);                                       // 延迟秒数
Queue::later(new \DateTime('+10 seconds'), $job, $data);            // 执行时间
Queue::connection('redis')->laterAtMs($msTimestamp, $job, $data);   // 毫秒时间戳
```

启动调度器（生产环境 stream 模式**必须**至少启动一个，可多实例）：

```bash
php think queue:scheduler                          # 使用默认连接
php think queue:scheduler redis --queue=default    # 指定连接与队列
php think queue:scheduler --batch=200 --interval=5 # 覆盖单次迁移数量与兜底间隔
```

调度器机制：

- **动态等待**：查询 ZSET 最早任务的毫秒时间戳，按剩余时间设置等待时长，到期立即迁移；不使用高频轮询，延迟集合为空时按 `interval` 兜底定期检查
- **唤醒机制**：生产者写入延迟任务后通过 Pub/Sub 发布唤醒通知，调度器收到后立即重新查询最早任务；通知丢失或断线期间由兜底间隔覆盖
- **原子迁移**：到期任务通过 Lua 脚本「查询 → 写入 Stream → 清理 ZSET/HASH」一次执行完成，多个调度器并发运行不会重复迁移同一条任务
- **至少一次投递**：迁移后任务完全交由现有 Stream 消费机制（ACK、PEL、重试、幂等锁）处理；重复投递由 job_id 幂等锁兜底
- **重启恢复**：调度器不持有任何任务状态，重启后自动从 ZSET 恢复调度；Redis 断线自动重试，恢复后继续调度

## 队列监控（queue:monitor）

实时只读监控队列状态，全程仅使用 Redis 只读命令（`XINFO` / `XPENDING` / `ZCARD` / `SCAN`），不向业务队列写入任何数据：

```bash
php think queue:monitor                    # 实时刷新（默认 2 秒）
php think queue:monitor redis --queue=default --interval=1
php think queue:monitor --once             # 单次快照（适合 cron / 告警脚本）
php think queue:monitor --json             # JSON 快照（适合程序采集、接 Web 面板）
```

输出示例（stream 模式）：

```text
┌─ think-queue monitor ─ stream ─ queue=default ─ 2026-11-20 09:31:24 ─
Stream:  待消费 3 │ 执行中(PEL) 1 │ 其中超时 0
吞吐:    已生产 5 │ 已完成 ≈1 │ 失败 -
Delayed: 10 个 │ 最近: [delay-0…] app\job\SendMail @09:32:24 ...
Workers: tobycroftMacBook-Pro.local-48443 pending=1 idle=0.0s
锁占用:  1
```

各字段怎么看：

| 字段 | 含义 | 关注点 |
|------|------|--------|
| 待消费 | Stream 中未处理完的消息数 | 持续增长说明消费速度跟不上生产速度，需加 worker |
| 执行中(PEL) | 已投递未确认的任务数 | 卡住不动说明 worker 崩溃或任务卡死 |
| 其中超时 | PEL 中 idle 超过 `retry_after`、将被重投的数量 | 持续不为 0 说明任务执行太慢或 worker 不足 |
| 已生产 / 已完成 | `entries-added` 累计生产；完成数由生产 − 在途 − 重回延迟推导（`≈` 估算值） | 观察整体吞吐；重试任务每轮完成都会计入 |
| 速率 | 相邻两次刷新的完成数差分 | 实时处理能力 |
| Delayed | 延迟任务总数 + 最近到期任务时间轴 | 确认延迟任务按计划到期 |
| Workers | 在线消费者及各自 pending、idle | idle 长时间不变或列表里出现已下线机器 → worker 掉线 |
| 锁占用 | 幂等锁数量 | 与 PEL 数量对应；异常增长说明有任务长时间未完成 |
| 失败 | 失败任务数（仅 `database` 失败驱动可统计） | 触发告警的指标 |

注意事项：

- 完成数是估算值：延迟任务超过 300 条时采样统计，显示 `≈≈`，误差会增大
- list 模式自动降级为 Ready / Reserved / Delayed 三项基础数量

## 任务幂等锁（lock）

默认开启。任务执行时长超过 `retry_after` 时，消息会被重新投递给其他 worker，导致同一任务被多个进程同时执行。开启幂等锁后，pop 到已被其他 worker 处理中的同一任务（按 payload 中的任务 id 判定）会直接跳过本次投递，避免重复执行。

使用约束：

- `lock_ttl` 应大于最长任务执行时长，否则锁过期后仍可能重复执行
- worker 崩溃时，其锁定的任务需等待 `lock_ttl` 过期后才能被重新认领
- list 模式下被跳过的重复投递会被直接丢弃（任务至少执行一次）；stream 模式下消息保留在 PEL 中，持有者完成后正常删除
- 可通过 `QUEUE_REDIS_LOCK=false` 关闭（关闭后请务必保证 `retry_after` > 最长任务时长）

## 失败任务处理

`.env` 中 `QUEUE_FAILED_DRIVER=database` 时，失败任务（达到最大重试次数）写入 `failed_jobs` 表（表结构用 `queue:failed-table` 命令生成迁移），之后可用内置命令管理：

```bash
php think queue:failed           # 列出失败任务
php think queue:retry <id>       # 重试指定任务（all 表示全部）
php think queue:forget <id>      # 删除指定失败任务
php think queue:flush-failed     # 清空失败任务
php think queue:failed-table     # 生成 failed_jobs 表迁移
```

## 命令速查

| 命令 | 说明 |
|------|------|
| `queue:work [connection]` | 常驻消费任务（生产用，配合进程管理器） |
| `queue:listen [connection]` | 请求式消费（开发用） |
| `queue:scheduler [connection]` | 延迟任务调度器（stream 模式必需） |
| `queue:monitor [connection]` | 实时只读监控 |
| `queue:restart` | 通知所有 worker 当前任务完成后退出（部署后执行） |
| `queue:failed` / `queue:retry` / `queue:forget` / `queue:flush-failed` | 失败任务管理 |
| `queue:table` / `queue:failed-table` | 生成 jobs / failed_jobs 表迁移 |

## 运行测试

```bash
composer install
vendor/bin/phpunit                        # 单元测试 + list 驱动测试

# 集成测试需要真实 Redis（stream 模式 / 延迟任务 / 监控指标）
redis-server --port 6399 --save '' --appendonly no --daemonize yes
REDIS_PORT=6399 vendor/bin/phpunit
```
