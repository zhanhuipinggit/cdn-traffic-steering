# OpenCDN 系统架构交互流程文档

## 🌐 全局架构概览
```mermaid
graph TD
    subgraph 客户端
        A[用户A] -->|WebSocket/QUIC| B(边缘节点TY)
        C[用户B] -->|WebSocket/QUIC| D(边缘节点SA)
    end

    subgraph 控制平面
        E[路由决策引擎] --> F[BGP控制器]
        E --> G[AI调度引擎]
        E --> H[合规管理中心]
    end

    subgraph 数据平面
        B -->|gRPC| E
        D -->|gRPC| E
        B -->|CRDT同步| D
    end

    F -->|BGP路由更新| I[骨干网路由器]
    G -->|模型更新| J[监控数据湖]
    H -->|TEE指令| K[可信执行环境]
```

## 🔄 核心交互流程详解
### 1. 用户接入与路由决策
```mermaid
sequenceDiagram
    participant User
    participant EdgeNode
    participant GeoEngine
    participant BGPCtrl

    User->>EdgeNode: 1. 连接请求 (携带客户端IP)
    EdgeNode->>GeoEngine: 2. 地理解析请求 (gRPC)
    GeoEngine->>MaxMindDB: 3. 查询地理位置
    MaxMindDB-->>GeoEngine: 4. 返回城市/国家代码
    GeoEngine->>BGPCtrl: 5. 请求可用节点列表 (protobuf)
    BGPCtrl->>BGPCtrl: 6. 执行路由策略检查
    BGPCtrl-->>GeoEngine: 7. 返回候选节点+指标
    GeoEngine->>AIEngine: 8. 请求动态权重计算
    AIEngine-->>GeoEngine: 9. 返回优化后的节点评分
    GeoEngine-->>EdgeNode: 10. 返回最优节点信息
    EdgeNode->>User: 11. 返回接入点地址 (HTTP 302)
```
#### 关键技术点：
- 使用EDNS Client Subnet传递客户端真实IP
- BGP控制器集成实时RIB(路由信息库)
- AI引擎基于PPO算法计算节点权重：
```python
def calculate_score(latency, loss, load):
    return 0.4*(1 - latency/500) + 0.3*(1 - loss) + 0.3*(1 - load) 
```

### 2. 消息传输与协议优化
```mermaid
flowchart TD
    subgraph 发送端处理
        A[接收用户消息] --> B{内容类型检测}
        B -->|视频流| C[启用QUIC协议]
        B -->|文本消息| D[启用WebSocket]
        C --> E[FEC冗余编码]
        D --> F[心跳保活机制]
        E --> G[边缘节点缓存]
        F --> G
    end

    subgraph 接收端处理
        G --> H{网络质量评估}
        H -->|丢包率>3%| I[切换TCP协议]
        H -->|延迟>200ms| J[启用多路径传输]
        H -->|正常| K[维持当前协议]
        I --> L[状态同步]
        J --> L
        K --> L
    end
```

#### 协议切换逻辑：
```go
func ProtocolSwitch(conn *Connection) {
    metrics := GetNetworkMetrics(conn.ID)
    if metrics.LossRate > 0.03 {
		conn.SwitchTo(TCP)
    } else if metrics.RTT > 200 {
		conn.EnableMultipath()
    }
    // 状态迁移保证
    MigrateSessionState(conn)
}
```

### 3. 异常处理与容灾机制

```mermaid
stateDiagram-v2
    [*] --> Normal
    Normal --> NetworkJitter: 检测到丢包率>5%
    Normal --> NodeFailure: 节点健康检查失败
    
    state NetworkJitter {
        [*] --> EnableFEC
        EnableFEC --> ProtocolFallback: FEC无法恢复
        ProtocolFallback --> [*]
    }
    
    state NodeFailure {
        [*] --> RouteWithdraw
        RouteWithdraw --> TrafficRedirect
        TrafficRedirect --> SessionMigration
        SessionMigration --> [*]
    }
    
    NetworkJitter --> Normal: 网络恢复
    NodeFailure --> Normal: 新节点就绪
```

#### 容灾操作步骤
**1.路由撤回**：通过BGP UPDATE消息通知全网
```bash
gobgp global rib del 192.168.1.0/24
```

**2.流量重定向**：更新GeoDNS权重分配
**3.会话迁移**：使用CRDT算法同步状态

```go
crdt.Merge(stateA, stateB, func(conflict){
    return conflict.HighestTimestamp()
})
```
## 📊 监控与优化体系
### 全链路监控架构
```mermaid
graph LR
    subgraph 数据采集
        A[eBPF探针] -->|内核事件| B[指标聚合器]
        C[应用SDK] -->|业务指标| B
        D[BGP监听] -->|路由变更| B
    end

    subgraph 分析处理
        B --> E[实时流处理]
        E --> F[异常检测]
        E --> G[AI训练]
    end

    subgraph 可视化
        F --> H[告警系统]
        G --> I[动态权重调整]
        E --> J[Grafana仪表盘]
    end
```
#### 关键监控指标：

---
指标类型	&nbsp;&nbsp;&nbsp;&nbsp;  采集频率&nbsp;&nbsp;&nbsp;&nbsp;	 告警阈值

---
节点CPU使用率&nbsp;&nbsp;&nbsp;&nbsp;	10s&nbsp;&nbsp;&nbsp;&nbsp;	>80% 持续5分钟

---
跨境时延(P95)&nbsp;&nbsp;&nbsp;&nbsp;1min	&nbsp;&nbsp;&nbsp;&nbsp;>500ms

---
WebSocket重连率&nbsp;&nbsp;&nbsp;&nbsp;	30s&nbsp;&nbsp;&nbsp;&nbsp;	>5%

---
BGP路由震荡次数&nbsp;&nbsp;&nbsp;&nbsp;	实时&nbsp;&nbsp;&nbsp;&nbsp;	>10次/分钟

---

## 🛠️ 开发者指南
### 本地调试环境搭建
```bash
# 启动最小化集群
docker-compose -f deploy/docker-compose-dev.yaml up

# 运行集成测试
go test -v ./test/integration -tags=integration

# 性能剖析示例
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.pprof
go tool pprof cpu.pprof
```

### 扩展开发示例：添加新协议
1.在pkg/protocol创建新协议包
2.实现Protocol接口：
```go
type MyProtocol struct {
    // 实现必要方法
}

func (p *MyProtocol) Migrate(conn *Conn) error {
    // 协议特定迁移逻辑
}
```

3.注册到协议管理器：
```go
protocol.Register("myproto", &MyProtocol{})
```






