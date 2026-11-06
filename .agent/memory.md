<!-- ID:001 -->
[2026-10-18] pipez fork 项目(工作目录 D:/Documents/mcmod/LMaxs-Pipez-fork):
- 上游: henkelmax/pipez outdated/1.20.1 分支(已clone到 ./pipez), 参考 Shinkarna/pipez 的fork思路
- 核心需求: 1.传输模式由tick轮询改为并行(最重要) 2.性能优化+正确性修复 3.管道自动模式(自动输入输出) 4.重设计交互:废除"端口位置切换提取/接收,提取位插升级插件提速"的旧设计
- 备忘录需求来源: "D:\Documents\my\日常\!备忘录-剪贴板-随笔记-待办_不要删除这个文件.md" 第33-38行
tag: pipez项目, LMaxs-Pipez-fork, mc模组
<!-- END:001 -->
<!-- ID:002 -->
[2026-10-18] max 的 fork 模组通用规范(来自备忘录第20-21行):
- 命名: 统一 LMaxs ... fork 样式; 原创模组正常起名
- 版本号两段式: 如 1124.2+1.20.1+forge, 第一段(如1124)表示基于的原模组版本,点后是max自己的版本号,原模组大版本更新则自己的小版本号重置回.1
- max自己的版本号正常三位式1.0.0,用不上三段式时可以只写一位(如 1)
tag: fork模组通用规范, 版本号, 命名
<!-- END:002 -->
<!-- ID:003 -->
[2026-10-18] pipez 1.20.1 代码审查结论(详见工作目录 AUDIT.md): 正确性bug: 1.BFS距离锁死(HashMap.findAny遍历,NEAREST/FURTHEST排序可能错,修法=ArrayDeque FIFO) 2.Mekanism 10tick补丁应用addListener替代 3.能量RR指针语义脏. 性能: canInsert热循环每次2stream+2列表(修=预分类过滤器缓存), travelPositions.contains O(n), isFull全扫, Shinkarna用TrackingItemHandler mixin验证了空槽位跳过思路. 架构: 三介质两套节奏(物品模运算爆发/流体每刻/能量半事件驱动), 上游的"只提取端有BE+BFS拓扑"设计值得保留, 能量推式转发模型是自动模式的参考. Shinkarna CE=换皮(modid改pipez_ce)+backoff mixin+mekanism阀门mixin, 代码不可直接抄.
tag: pipez项目, 代码审查, LMaxs-Pipez-fork
<!-- END:003 -->
<!-- ID:004 -->
[2026-10-18] pipez 能量管道深查(已并入AUDIT.md #12-22): 病灶=模型不完整. 1.速率契约漏洞: receive()的rate只限单次调用,推路径每tick吞吐=调用次数×rate无预算记账 → M3用per-side tickBudget(推拉共享扣减) 2.推拉双通道4份分发算法,全mod共8份复制 → M3抽Distributor层,tickBudget是合并双通道的粘合剂 3.端口语义拧巴: extracting side实为源口+收口双向口,是max"端口设计要改"的病灶本体 4.无限sink伪装(getMaxEnergyStored=MAX_VALUE零缓冲) 5.空闲轮询无退避(务实解=指数退避) 6.新bug: receiveEnergy无条件刷新lastReceived含simulate分支→第三方周期探测永久抑制空闲拉取,修法=simulate不更新 7.均分怪癖: 余数蒸发+每目的地只吃均份额不让贤(吞吐≤rate) 8.RR指针修一半实锤: 推路径Pair带原索引正确,拉路径错位(#3),M3采纳Pair方案 9.路径成本不对称: 拉路径pushEnergy三往返 vs 推路径一往返. 分发流程: 准入→距离排序→RR两阶段(收集探测/均份发放,指针仅非simulate写回)或顺序(前吃撑后喝汤).
tag: pipez项目, 代码审查, 能量管道, 架构
<!-- END:004 -->
<!-- ID:005 -->
(2026-10-18) max的pipez历史bug根因定罪: 能量管道默认分布=ROUND_ROBIN(PipeType.java:43基类默认,EnergyPipeType未override;ItemPipeType.java:47特意override成NEAREST是旁证). RR语义=每tick所有N个目的地各得rate/N细流,吃不完的份额当tick蒸发(不让贤). 症状: 用电器多时大户机器永远收不到电(细流<消耗),小电池同细流零消耗几百tick攒满(看似瞬间充满),提取端总量够(直觉没错,被均分摊薄),N越大越糟. 解药=仅NEAREST+吃满让位(ordered语义,无摊薄). 次要放大器: 懒初始化capability在BFS时未present→机器掉出目的地表(#2 Mekanism类).
tag: pipez项目, LMaxs-Pipez-fork, 能量管道, bug根因
<!-- END:005 -->
