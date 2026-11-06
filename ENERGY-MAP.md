# ENERGY-MAP — pipez 能量传输系统现状底图 (2026-10-18, 能量专项开工参考)

## 本质
零库存调度台: 能量从不进入管道, 网络里没有"流动"; 每次传输 = 跨网络原子直送.
管道只做两件事: 维护目的地表 + 按表转发.

## 流程
推: 机器 ──receiveEnergy──▶ 面孔(准入: 该面isExtracting+红石) ──即时分发──▶ 目的地
拉: tick空闲窗口(lastReceived) ──extractEnergy──▶ 提取面紧邻方块(唯一源) ──pushEnergy──▶ 目的地
目的地表: connectionCache = BFS预计算全网络可收口(带距离), 仅提取端BE持有

## 组件
1. PipeBlock: 每方向 isConnected, 纯方块状态, 无BE开销
2. EnergyPipeTileEntity: 仅提取面/断开面管道持久BE(hasReasonToStay), 其余裸方块
3. PipeEnergyStorage: 每面一个IEnergyStorage面孔; 容量谎报#15; lastReceived#20
4. EnergyPipeType: 无状态算法层(rate=配置×升级档 + 4份分发算法)
5. 配置: 分布模式/红石/过滤器/RR指针 → 全在升级物品NBT(#10)

## 结构
- extractingConnectionCache[6]: 提取面紧邻单方块 = 拉源(距离恒1)
- connectionCache: BFS目的地表(带距离), markPipesDirty拓扑传播失效
- 防环三层: 管道间不暴露capability(硬) + recursionDepth + lastReceived窗口(#11/#17)

## tick时间轴
1. Mekanism倒计时递减(#2)
2. 每提取面: 空闲判定 → 拉一次(无退避#16)
3. 任意时刻: 机器推 → 即时分发(无预算#12)
4. 拓扑变更 → 全网缓存作废重建

## 涌现性质(AUDIT对照)
零缓冲 | 无预算(吞吐=调用次数×rate) | 双通道时间窗互斥(simulate污染#20)
| 配额每tick蒸发(#21) | 距离可能错(#1→NEAREST/FURTHEST)

## 阶段1手术映射
#20 → receiveEnergy第一行(simulate不刷时间戳)
#3  → insertEqually(Pair原索引)
#1  → updateConnectionCache(ArrayDeque FIFO)
#16 → tick空闲拉取(指数退避)
#18 → 两条equally(消临时列表)