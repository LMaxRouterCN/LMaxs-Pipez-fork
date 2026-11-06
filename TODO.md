# TODO — LMaxs-Pipez fork (2026-10-18 更新: max 拍板能量分发简化)

## 路线状态
- 主线: 只修能量管道 (max 明确范围, 其他介质不管)
- 已拍板: 能量分发 = 仅 NEAREST 顺序. 语义 = 洪水填充(BFS)按距离排序,
  开头吃满则余量后传, 配额尽则截流. = 现有 ordered 路径原样保留.
  RR/FURTHEST/RANDOM 代码切断但不删 (2 分派点 + GUI 锁, 可复活).
- 归档: 连通器新电线模型 (记忆004) | 挂起: M3 重构 (等剩余决策点), #4-#7 其他介质

## 阶段 0 — 环境验证 (前置)
- [x] gradlew build 基线 (项目从未构建过)
- [ ] 构建后取 corelib sources, 验证 pushEnergy 三步量差 (#19)

## 阶段 1 — 能量正确性批 (已按新模型重排)
- [x] #1 BFS 重写 → ArrayDeque FIFO (第一刀: 洪水填充本体, 唯一排序依据;
      同距离按方向枚举序稳定平局)
- [ ] #20 receiveEnergy simulate 分支不刷 lastReceived (一行)
- [ ] 模式切断 (新):
      a. 侦察: 两分派点位置 + ExtractScreen 按钮 (消 #23f 未验证项)
      b. receive()/拉路径 恒走 ordered
      c. GUI 能量侧分布按钮锁死 (默认做, max 可否决)
      验收: 三模式不可达, 函数与 NBT 状态完整保留
- [ ] #16 空闲拉取指数退避 (1→2→…→64 封顶; 成功/拓扑变更复位)

## 阶段 2 — 小语义决策 (默认已给, 点头即执行)
- [ ] #15 getMaxEnergyStored 如实返回 0

## 死代码区 (随切断自动失效, 不修不删)
- #3 拉路径 RR 指针 | #18 equally 分配 | #21 均分怪癖
  复活条件: 重新启用均分模式时一并修

## 阶段 3 — 需实测
- [ ] #2 Mekanism → LazyOptional.addListener (dev 环境 + Mekanism)
- [ ] #19 视 corelib 验证结果决定是否绕开 pushEnergy

## 挂起区
- M3 Distributor 层 + per-side tickBudget | #4-#7 | 连通器模型 (SOC 均衡/leader/微缓冲/n 休眠)