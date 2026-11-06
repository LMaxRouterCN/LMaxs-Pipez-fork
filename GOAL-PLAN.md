# GOAL-PLAN — LMaxs Pipez Fork

## 项目定位
- 基于上游 henkelmax/pipez 的 outdated/1.20.1 分支 (commit 96c2722) 的修改版
- MC 1.20.1 / Forge 47.2.0 / Java 17 / 依赖 corelib 1.1.1 + JEI/TOP/Jade/Mekanism 集成
- 版本号: 1226.1+1.20.1+forge (按 max 的 fork 规范: 原版 1.2.26 → 1226, 自身版本 .1 起步) — 待 max 确认
- modid 决策: 上游是 pipez; Shinkarna CE 改成 pipez_ce. 建议**保留 pipez**以兼容已有存档与配置 — 待 max 确认

## 需求清单 (来源: max 备忘录 33-38 行)
1. [核心] 传输模式由轮询改为并行
2. 性能优化
3. 正确性修复
4. 管道自动模式 (自动输入输出) — 语义待 max 确认
5. 交互重设计: 废除"端口位置切换提取/接收 + 提取位插升级插件提速" — 目标形态待 max 描述

## 上游架构速写 (审查后)
- 只在提取端生成 BlockEntity, 非提取管道是纯方块 → 网络拓扑靠提取端 BFS 遍历
- PipeLogicTileEntity.tick → PipeType.tick (四介质) → getSortedConnections → insert*
- 配置 (过滤器/分布模式/红石模式/升级) 全部存在升级插件的 NBT 里, TE 只做缓存
- 能量管道: 机器推入时被动转发 (PipeEnergyStorage), 空闲时主动拉取 — 半事件驱动, 是四介质里最先进的模型

## 核心方案 (草案, 待 max 拍板)
### A. 并行/统一传输模型
- 三介质统一为"每刻配额流": quota 按 tick 累积 (rate × elapsed), 告别 gameTime % speed 模运算爆发
- 旧 speed 配置语义迁移: 新速率 = 旧amount / 旧speed, 保留旧配置键做兼容映射
- 计算与调度解耦: PipeType 只保留"最小传输单元", 调度策略外置
- 调度器形态 (决策点): per-TE ticker (保守, 零迁移风险) vs 中央调度器 (优雅, 需自建注册表)
### B. 正确性修复批 (先于 A 落地, 见 AUDIT.md)
### C. 自动模式
- 待 max 确认语义后设计
### D. 交互重设计
- 数据结构先行: 配置从"升级插件 NBT"迁出/共存, 必须做 NBT 迁移器保证老存档可用
- UI 形态等 max 描述

## 风险与护栏
- MC 世界操作必须主线程, "并行"指逻辑流水线/统一节奏, 不是线程级并行
- 存档兼容是红线: 升级插件 NBT / TE NBT 变更必须有迁移路径
- Reference (reference/Shinkarna-pipez): backoff 思路可借鉴, 代码不可直接抄 (modid/包名不同)

## 里程碑
M1 基础设施: git 分支 lmaxs/1.20.1, 文档 ✓ (本次会话)
M2 正确性修复批: BFS 距离 bug / 热循环分配 / capability 监听替代 10tick 补丁
M3 并行传输重构: 配额流模型, 三介质统一
M4 自动模式
M5 交互重设计 + NBT 迁移
M6 测试: gradlew build / runClient 实测 / 大网络性能对照