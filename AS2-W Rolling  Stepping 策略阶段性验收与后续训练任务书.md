# AS2-W Rolling / Stepping 策略阶段性验收与后续训练任务书

> 日期：2026-11-19  
> 任务类型：当前训练服务器只读审计 + 离线评测 + MuJoCo Sim2Sim + 域随机化设计  
> 当前阶段原则：**先看清现有策略、资产和跨仿真表现，再决定是否续训。默认不启动新的 PPO 训练。**  
> 文档输出位置：`docs/as2w_policy_validation_20261119/`

---

# 0. 任务目标与背景

## 0.1 当前策略目标

当前 AS2-W 策略希望实现以下运动方式：

- 前进、后退：以轮子滚动为主。
- 左移、右移：以腿部踏步为主。
- 原地左转、右转：以腿部踏步为主。
- 混合速度指令：滚动和踏步协同完成。

当前策略基于同事公开项目继续训练：

- 基础项目：<https://github.com/YuTaoV5/as2w-unitreestyle>
- 实机动力学校准资料：<https://github.com/bgzou/log/blob/main/SimDynamicsValidation_AS2W_20261114.zip>
- 相关历史审计记录：<https://github.com/bgzou/log>
- 历史专项目录名称：`as2w_layered_rolling_test/as2w-yaw-physics-aware-retrain-20261115`

此前在一台训练服务器上，分别进行了：
1. 从原始策略继续进行 PPO 训练；
2. 冻结原策略主体、使用有界残差方式继续训练。

已有报告显示，残差路线在短反转次数、峰值倾角等指标上有所改善，但部分侧移场景的抬落次数、步态幅度等指标也发生变化。报告中的数值先作为已有实验记录，不得未经复算就当作当前服务器的最新结果。

本任务的核心不是预设残差策略一定更好，也不是继续追求更高 PPO reward，而是回答：

1. 当前残差策略是否真的减少了不必要的抖动，同时保留必要的踏步能力？
2. 当前 Isaac Lab / PhysX 训练资产是否足以代表真实 AS2-W？
3. 当前策略在 MuJoCo 中是否仍然成立？
4. 现有域随机化覆盖了哪些误差，哪些重要误差仍未覆盖？
5. 当前阶段究竟应该继续训练、修改奖励、修正物理资产，还是暂时冻结策略？

## 0.2 非常重要：当前服务器与同事服务器不是同一台

**本任务只针对 Codex 当前实际连接的服务器。不得假设它就是此前同事使用的服务器。**

此前其他报告中出现过以下路径：

- `/home/robot/z00945176/Rob2Nav`
- `/root/gpufree-data/as2w-igym`
- `Rob2Nav-runtime/as2w-current/him_lab`
- `Rob2Nav-main-live`
- `upstream/HIMLoco-for-Go2W`

这些只是历史报告中的路径，不代表当前服务器一定存在。

当前服务器可能主要包含：
- Rob2Nav 相关代码和训练记录；
- AS2-W 相关训练或仿真；
- terrain 相关训练 run；
- 其他机器人或其他实验的仿真资产。

必须先检查当前服务器实际目录、Git 仓库、训练入口和 run 元数据，再判断哪些内容与本任务有关。不得把历史服务器路径写成当前服务器事实。

**不要扫描整个文件系统。** 先检查当前工作目录、用户明确的工作区、其下一级项目目录和已发现的 Git 仓库；必要时再对已确认的项目目录做有限深度搜索。禁止无边界执行 `find /`、全盘扫描或大量读取无关数据。

## 0.3 当前阶段的执行边界

允许：
- 读取当前服务器上的源码、配置、日志、checkpoint 元数据和资产；
- 对已有 checkpoint 做离线推理和评测；
- 在不影响训练任务的前提下做小规模离线仿真；
- 生成视频、曲线、CSV 和 Markdown 报告；
- 在全新目录中创建评测脚本或配置副本；
- 检查已有 MuJoCo 模型并建立独立评测入口。

未经用户后续确认，不允许：
- 启动新的正式 PPO 训练；
- 覆盖或删除现有 checkpoint；
- 修改旧训练配置或原始 URDF/USD/MJCF；
- 改变真实机器人 Kp/Kd、力矩限制或控制频率；
- 连接真机、发送运动指令或部署新策略；
- 为了让结果通过而降低评测标准；
- 在当前服务器上运行高负载任务并影响正在进行的训练。

如果需要修复评测脚本，优先复制到本任务独立目录中修改，不要直接改动原始训练代码。

---

# 1. 已有证据与复用原则

## 1.1 `SimDynamicsValidation_AS2W_20261114.zip` 已有校准工作

此前的 2026-11-15 A/B/C 线整理报告记录了以下工作：

- 对 `SimDynamicsValidation_AS2W_20261114.zip` 和 `FixedStand_kp55_kd2_real_robot_20261114.zip` 做过数据清点；
- 共整理 21 个实机片段，覆盖静态站立、起立、蹲姿、交接/制动和行走等场景；
- 建立过 `REAL_REFERENCE/`，其中包括 `real_reference.json`、`clip_inventory.csv`、`real_stand_reference.csv`、`real_wheel_tracking_by_phase.csv` 和 IMU/关节图；
- 实机腿部执行合同记录为 `Kp=55, Kd=2`，轮部记录为 `Kp=0, Kd=2`；
- 曾修正四元数顺序、动作缩放截断、增益写入、FK 符号、MuJoCo 轮速增益和初始化姿态等实现问题；
- 轮端制动衰减曾反推出约 `0.34–0.42 kg·m²` 的等效惯量候选；
- 历史报告仍记录了髋关节侧向载荷和约 `1.2–1.7°` 俯仰残差等未解决问题。

上述内容是**已有校准工作的索引和历史结论**，不是要求重新从头分析全部 21 个片段。

数据包历史报告中出现过 SHA256 前缀 `1ed1ed5e82b0308f`。这不是完整 SHA256，不能把它当作完整哈希；如能找到完整 manifest 或原始数据，应以完整 SHA256 为准。Git blob SHA 也不能代替文件 SHA256。

### 对实机数据的正确处理方式

1. 先在当前服务器和已发现的项目目录中查找是否已有原始 ZIP、解压数据、`REAL_REFERENCE/` 和校准报告。
2. 如果派生数据齐全、来源可追溯、文件没有损坏，不要重新清点全部 21 个片段，也不要重复跑已经完成的静态站立、起立、蹲姿和轮速伺服校准。
3. 重点核查当前训练实际加载的物理资产、控制增益、动作链和执行器模型，是否与已有实机校准所使用的合同一致。
4. 只有发现数据缺失、派生结果不可追溯、当前资产与历史校准合同不同，或现有数据确实没有覆盖某个关键工况时，才补做对应的局部复核。
5. 如果当前服务器没有该 ZIP，不要假装已经读取。先使用已有审计报告和派生数据；必要时再从用户提供的公开 GitHub 地址获取原始数据。若网络不可用，明确记录 `SOURCE_NOT_AVAILABLE`，不要阻塞其他阶段。
6. 不得因为重新跑了一遍脚本，就把历史数据称为新采集的实机数据。

## 1.2 对 `as2w-yaw-physics-aware-retrain-20261115` 的复用原则

此前针对 `as2w_layered_rolling_test/as2w-yaw-physics-aware-retrain-20261115` 的审计记录指出：

- 训练入口使用目录外已有的 r280 USD 和 AS2-W URDF；
- 该目录没有简单地交付一份新的、已经完成验收的 URDF/USD；
- USD 与官方 URDF 曾存在 base 质量和 COM 差异；
- `expanded` 配置在运行时随机化 base 质量、COM、轮摩擦和执行增益；
- 其中 COM 后移的解释来自特定承重/力矩比拟合，不应直接认定为最终标定值；
- 后续多工况复核发现，单独调整质量或 COM 不能同时解释 T1/T2、起立、下蹲、制动和行走中的误差；
- 曾发现 PhysX 轮胎接触使用 STL 凸包，而 MuJoCo 支撑接触主要使用轮关节中心附近的圆柱；两者接触几何不一致；
- 将 MuJoCo 圆柱简单移到胎面中心并没有让所有实机误差同时改善。

这些记录只作为**历史候选方案和审计线索**。当前 Agent 必须先确认当前服务器是否存在该目录或其报告副本。若不存在，记录 `NOT_FOUND_ON_CURRENT_SERVER`，不得把它写成当前服务器资产。

如果该目录可访问：
1. 读取训练入口、环境配置、实际加载的 USD/URDF、运行时随机化代码和报告；
2. 区分文件本身的静态资产修改与环境运行时参数修改；
3. 确认训练 run 是否真的选用了 `expanded`；
4. 追踪实际生效的质量、COM、摩擦和增益，而不是只看配置默认值；
5. 将它与当前训练策略的资产合同做差异表；
6. 不要直接复制其 COM、质量或摩擦参数到当前训练环境。

## 1.3 不同训练链不能混用

公开 `as2w-unitreestyle` 项目中存在 12D 腿部策略与轮速映射组合的控制实现；历史 Rob2Nav/HIMLoco 记录中也出现过 16D、包含轮动作输出的策略链。

必须以当前服务器的真实代码为准，明确当前策略究竟属于哪种动作合同。

禁止：
- 把 12D 腿部 Actor 与固定轮速映射，和 16D 腿轮联合 Actor 混为一谈；
- 把不同训练链的 checkpoint 互相加载；
- 把不同观测维度、历史帧、动作缩放、Mapper 和执行器合同的评测结果直接比较；
- 把其他服务器的 run 结果写成当前服务器实测结果。

---

# 2. Phase 0：当前服务器环境与训练 run 识别

**目标：确认当前 Agent 实际连接的是哪台服务器、当前策略训练在哪里、有哪些相关资产和 terrain run。**

只读执行。

## 2.1 服务器和工作区识别

记录：
- hostname、用户名、当前工作目录；
- 当前工作区顶层目录；
- GPU 型号、显存和当前利用率；
- 相关训练进程和资源占用；
- Python、PyTorch、Isaac Lab、Isaac Sim、MuJoCo 版本（仅读取已有环境，不安装）；
- 已发现的 Git 仓库路径、remote、branch、HEAD、工作区状态。

不要输出环境变量中的密钥、token 或其他凭据。

## 2.2 找到当前两条续训实验的真实位置

识别：
- 原始策略 checkpoint；
- 基线全策略续训 checkpoint；
- 有界残差续训 checkpoint；
- 各自训练日志和评测结果；
- 训练入口和实际加载配置；
- 训练时使用的 robot asset；
- 当前正在运行或最近完成的 run。

每个 run 记录：
- 绝对路径；
- Git commit；
- checkpoint 文件名和 SHA256；
- 训练迭代数；
- seed；
- 观测/动作维度；
- Physics dt、control dt、decimation；
- Kp/Kd、动作缩放、限幅和滤波；
- 使用的 URDF/USD/MJCF 路径及 SHA256；
- 域随机化配置；
- 是否有独立评测结果。

如果无法唯一识别用户所说的两条续训线，先列出候选并说明原因，不要猜测。

## 2.3 Rob2Nav 和 terrain 训练记录

在当前服务器实际存在的 Rob2Nav 工作区中，查找：
- AS2-W 相关训练任务；
- AS2-W 仿真资产；
- terrain 训练 run；
- terrain 生成器和地形配置；
- 训练日志、checkpoint、评测结果；
- Isaac Lab 环境配置；
- 是否存在 MuJoCo 模型和转换工具。

对 terrain run 必须确认机器人模型、任务名称和资产来源。不能因为目录名叫 `terrain` 就认定它属于 AS2-W；也不能把 Go2 或其他机器人训练的随机地形参数直接用于 AS2-W。

## 2.4 Phase 0 交付

写入：

`docs/as2w_policy_validation_20261119/00_SERVER_AND_RUN_DISCOVERY.md`

至少包含：
- 当前服务器真实身份和工作区；
- 找到的 Git 仓库；
- 当前策略两条续训线的真实路径；
- Rob2Nav/terrain 相关 run 清单；
- 找到和未找到的 AS2-W 资产；
- 当前训练资源状态；
- 所有 `NOT_FOUND` 项；
- 哪些历史路径不属于当前服务器。

完成后继续 Phase 1，不要因历史路径不存在而停止。

---

# 3. Phase 1：Isaac Lab 策略可视化与步态质量复核

**目标：解决当前只有文本日志、看不到机器人实际运动的问题。**

不得启动 PPO 训练。只对已有 checkpoint 做离线评测和渲染。

## 3.1 对比策略

至少比较：
1. 原始策略；
2. 基线全策略续训；
3. 有界残差续训。

所有 checkpoint 先计算 SHA256 并冻结。评测时不能覆盖权重。

如果当前服务器缺少某个 checkpoint，记录缺失项，不得用其他策略冒充。

## 3.2 固定评测条件

- 尽量使用同一套 Isaac Lab 环境和同一物理资产；
- 使用相同初始状态、命令时序和测试 seed；
- 训练环境与评测环境配置分开记录；
- 至少 3 个评测 seed；如成本较高，先做 1 个诊断 seed，再补 2 个确认 seed；
- 明确是否启用了 domain randomization；
- 同时保留 nominal plant 评测和随机化评测，不能混在一张表里；
- 如果某个策略的观测/动作合同不同，先做合同适配审计，不得直接比较。

## 3.3 必须覆盖的命令场景

至少包括：
1. 前进；
2. 后退；
3. 左移；
4. 右移；
5. 原地左转；
6. 原地右转；
7. 前进 + 左转；
8. 前进 + 右转。

每个场景记录命令幅值、开始时间、保持时间、停止时间和恢复时间。

## 3.4 视频要求

尽可能使用 Isaac Lab 当前已有的 headless/offscreen 渲染方式，不要为了视频安装新依赖或改动系统环境。

输出：
- 每个策略一组场景视频；
- 最重要的 6 个场景做三策略对比；
- 如支持，生成同视角、同初始状态、同命令的并排视频；
- 视频需显示机器人全身、四轮和四条腿，不能只截取局部；
- 在画面上叠加命令和关键状态量。

建议视频画面叠加：
- `vx_cmd / vx_meas`
- `vy_cmd / vy_meas`
- `wz_cmd / wz_meas`
- roll / pitch
- 四轮接触状态和法向力
- 四条腿的关键关节角
- 当前策略名称、checkpoint SHA 短标识、seed 和仿真时间

视频保存到：

`runs/as2w_policy_validation_20261119/phase1_isaac/videos/`

图表和原始数据保存到：

`runs/as2w_policy_validation_20261119/phase1_isaac/`

## 3.5 必须计算的步态指标

不能只看 reward、跌倒次数和机身倾角。

### A. 运动跟踪
- `vx`、`vy`、`wz` 跟踪误差；
- 稳态误差；
- 启动响应时间；
- 命令停止后的速度衰减；
- 左右方向对称性。

### B. 轮地接触
- 每个轮子的接触占空比；
- 法向力均值、峰值和波动；
- 轮子离地次数和持续时间；
- 轮速与轮地相对速度；
- 纵向和侧向滑移；
- 是否存在依靠轮胎侧滑代替腿部踏步的情况。

### C. 腿部步态
- 每个腿关节的角度范围、速度和加速度；
- 抬落事件次数；
- 步态周期；
- 支撑占空比；
- 摆动高度和后小腿净空；
- 左右腿相位差；
- 左移、右移、左转、右转的镜像差异。

### D. 平顺性
- action 一阶差分；
- action 二阶差分或 jerk；
- 关节速度峰值；
- 关节加速度峰值；
- 高频振荡频谱；
- 机身角速度波动。

### E. 安全与稳定
- 跌倒次数；
- 最大 roll/pitch；
- 非轮部位触地次数；
- 关节限位触发；
- 力矩或速度饱和比例；
- episode 是否因 timeout/reset 被截断。

## 3.6 Phase 1 的判断要求

明确回答：
- 残差策略是否真的减少了高频抖动？
- 是不是通过缩小关节运动幅度、减少踏步来降低抖动？
- 左移、右移和左右转的踏步是否仍然有效？
- 轮子是否正常滚动，还是出现异常滑移？
- 左右转是否对称？
- 是否有某个指标改善、但另一个关键能力退化？

不能只根据视频观感判断，也不能只根据单一指标判断。

## 3.7 Phase 1 交付

`docs/as2w_policy_validation_20261119/01_ISAAC_POLICY_VIDEO_AND_GAIT_EVAL.md`

另附：
- `ISAAC_EVAL_SUMMARY.csv`
- `GAIT_METRICS.csv`
- 视频相对路径和 checkpoint SHA；
- 每个场景的失败案例截图或关键帧；
- 三种策略的明确对比结论。

---

# 4. Phase 2：当前训练资产、Rob2Nav 资产和 terrain 资产审计

**目标：搞清楚当前训练到底在什么机器人模型上学习，以及 Rob2Nav 里面哪些资产值得复用。**

只读审计，不修改旧资产。

## 4.1 建立资产清单

至少区分：

A. 官方 AS2-W URDF 和 STL；

B. 当前策略实际加载的 Isaac Lab / PhysX USD；

C. 当前策略实际使用的 MuJoCo MJCF（若存在）；

D. Rob2Nav 中 AS2-W 的 USD、URDF、MJCF 和训练 run；

E. terrain run 使用的机器人模型和地形资产。

每个资产必须记录：
- 路径；
- SHA256；
- 来源；
- 对应 Git commit；
- 被哪些 run 加载；
- 是静态资产还是运行时修改；
- 是否经过实机校准；
- 是否有对应 MuJoCo 模型。

## 4.2 质量和动力学参数

逐项比较：
- 整机总质量；
- base_link 质量；
- 每条腿的质量；
- 轮子质量；
- COM XYZ；
- 完整惯量矩阵；
- 惯量坐标系；
- 轮胎实际半径和宽度；
- 轮轴位置；
- 关节轴方向；
- 关节零位；
- 关节限位；
- 碰撞体类型；
- 碰撞体相对视觉几何的位置。

不得只比较 URDF 文本。要追踪 USD 导入后的实际值和运行时修改。

## 4.3 执行器和时间合同

检查：
- 腿部 Kp/Kd；
- 轮部 Kp/Kd 或速度驱动实现；
- 力矩上限；
- 速度上限；
- armature；
- damping；
- 轮端等效惯量；
- 动作缩放和 clip；
- 低通滤波；
- 命令限速；
- 观测频率；
- 策略推理频率；
- Physics dt；
- control dt；
- decimation；
- 传感器和控制延迟。

必须确认参数在运行时真实生效，不能只读配置文件。

## 4.4 轮胎接触模型重点审计

重点比较 PhysX 和 MuJoCo：

- STL、Convex Hull、Cylinder 或其他代理体；
- 胎面和轮轴中心的偏移；
- 实际接地点；
- 轮胎半径；
- 轮宽；
- 法向接触参数；
- 静摩擦和动摩擦；
- 侧向摩擦；
- 滚动摩擦；
- 扭转摩擦；
- contact offset；
- solver/substep；
- friction combine mode；
- 轮关节与接触几何的相对位置。

不能把两个引擎中的摩擦系数设置为同一个数字，就宣称接触模型等价。

## 4.5 terrain 训练资产单独归档

对发现的 terrain run，检查：
- 机器人型号；
- 训练任务；
- terrain 类型；
- terrain 生成方式；
- 训练和评测的地形分布；
- 是否有坡面、台阶、随机高度场；
- 是否存在轮足接触专用设置；
- 是否和当前 AS2-W 的轮胎几何、关节结构一致。

如果 terrain run 是其他机器人，不要把其参数直接移植。可以提取地形生成和课程学习方法作为参考，但必须标注为方法参考，而不是已验证的 AS2-W 配置。

## 4.6 Phase 2 交付

`docs/as2w_policy_validation_20261119/02_ASSET_AND_ROBNAV_AUDIT.md`

同时生成：
- `ASSET_MATRIX.csv`
- `RUN_MATRIX.csv`
- `TERRAIN_RUN_MATRIX.csv`

每条结论标记：
- `SOURCE_CONFIRMED`
- `RUN_CONFIRMED`
- `RESULT_CONFIRMED`
- `INFERENCE`
- `UNKNOWN`

---

# 5. Phase 3：复用实机校准数据，定位真正缺失的工况

**目标：不要重复校对全部历史实机数据，而是确认当前策略的仿真资产是否继承了已经完成的校准。**

## 5.1 先做数据 lineage 检查

查找：
- `SimDynamicsValidation_AS2W_20261114.zip`
- `FixedStand_kp55_kd2_real_robot_20261114.zip`
- `REAL_REFERENCE/`
- `real_reference.json`
- `clip_inventory.csv`
- `real_stand_reference.csv`
- `real_wheel_tracking_by_phase.csv`
- 相关校准脚本、报告、CSV、NPZ 和图表。

如果当前服务器没有这些文件：
- 不要重新制作一套假的 REAL_REFERENCE；
- 先读取已有报告；
- 记录 `SOURCE_NOT_AVAILABLE_ON_CURRENT_SERVER`；
- 继续其他阶段。

## 5.2 不要重复执行的工作

如果原始数据和派生结果完整、来源一致，以下工作视为已经完成，不要无理由重跑：
- 21 个片段的重复清点；
- 已完成的静态站立对照；
- 已完成的起立和蹲姿数据整理；
- 已完成的 55/2 增益确认；
- 已完成的四元数顺序、动作缩放、增益写入、FK 符号等软件错误修复；
- 已完成的轮速伺服基础测试。

需要确认这些结论是否被当前训练环境真正采用，而不是重复做同样的数据处理。

## 5.3 需要重点复核的差异

将当前训练的资产和执行器合同与已有实机校准逐项比较：

1. 当前腿部增益是否为实机对应的 55/2；
2. 当前轮部控制是否与实机 0/2 合同一致；
3. 当前轮端惯量是否考虑了历史制动衰减辨识；
4. 当前轮胎接触模型是否能解释侧向髋关节载荷；
5. 当前 COM 和质量是否来自已验证的多工况结果；
6. 当前起立模型是否考虑实机 calf 机械干涉或已明确排除该工况；
7. 当前前后轮支撑高度是否有测量依据；
8. 当前长时间侧移和持续转向是否有实机参考数据。

注意：
- `0.34–0.42 kg·m²` 是历史轮端制动响应反推的等效惯量候选，不得未经模型定义核对就称为真实电机转子惯量。
- 髋关节侧向力矩差异可能涉及橡胶轮胎侧向刚度，但这仍是待验证解释。
- 约 `11.5 mm` 的前后轮支撑高度差是历史反推假设，不是已经直接测量的确定值。
- 任何历史质量/COM 修改都必须经过多工况检查，不能只凭 FixStand 站姿决定。

## 5.4 仅补缺失的测试

如果历史数据没有覆盖，优先补以下仿真侧测试：
- 长时间左移、右移；
- 10–20 s 持续左转和右转；
- 转向停止后的驻停；
- 侧向刮擦和轮胎侧向滑移；
- 不同轮端负载下的制动响应；
- 当前训练资产与校准资产的差异复演。

如果必须补实机数据，先输出明确的数据缺口和所需采集内容，等待用户单独授权。当前任务不授权任何真机运动测试。

## 5.5 Phase 3 交付

`docs/as2w_policy_validation_20261119/03_REAL_DATA_REUSE_AND_GAP_AUDIT.md`

内容包括：
- 已有数据位置和可用性；
- 原始数据与派生结果的关系；
- 已完成校准项；
- 当前训练资产是否继承这些校准；
- 尚未解释的物理误差；
- 真正需要补测的工况；
- 哪些结论是测量值，哪些是模型推断。

---

# 6. Phase 4：MuJoCo 模型检查与 Sim2Sim 验收

**目标：判断当前策略是否只适用于 Isaac Lab / PhysX，还是能在独立 MuJoCo 模型中保持相同运动能力。**

不要把简单 action 回放称为完整 Sim2Sim。

## 6.1 找到正确的 MuJoCo 模型

优先查找与当前策略同一训练链的 MJCF。

如果找不到：
1. 检查当前策略仓库是否有官方或已有转换脚本；
2. 检查 Rob2Nav 中是否有 AS2-W MJCF；
3. 检查 `as2w-yaw-physics-aware-retrain-20261115` 是否在当前服务器或可访问的共享目录；
4. 如果仍然没有，先报告缺少同源 MJCF；
5. 可以在独立目录中评估 URDF→MJCF 的转换可行性，但转换结果必须标记为 `UNVALIDATED_CANDIDATE`，不能直接当作合格模型。

不要把 Go2 或其他机器人 MJCF 当作 AS2-W 模型。

## 6.2 第一关：策略接口 parity

在同一时刻、同一状态下，给 PyTorch Actor 和 MuJoCo 评测包装输入完全相同的 observation。

比较：
- observation 每个维度；
- 归一化前后数值；
- 历史帧顺序；
- command 单位；
- action raw；
- action scale；
- action clip；
- 滤波器状态；
- 最终腿部关节目标；
- 最终轮速目标。

确认策略架构和动作合同：
- 如果是 12D 腿 Actor + 固定轮速映射，必须按该合同实现；
- 如果是 16D 腿轮联合 Actor，必须按该合同实现；
- 不得为了让 MuJoCo 能运行而静默改变 observation/action 语义。

## 6.3 第二关：开环物理回放

两种仿真接收完全相同的最终控制目标，不运行策略反馈。

比较：
- 关节角、速度；
- 机身姿态；
- 轮速；
- 轮端力矩；
- 四轮法向接触力；
- 轮胎滑移；
- 接触状态；
- 停车和制动衰减。

开环回放用于检查物理模型和执行器响应，不代表策略已经通过 Sim2Sim。

## 6.4 第三关：闭环 Sim2Sim

同一个 checkpoint 分别在 PhysX 和 MuJoCo 中闭环运行：

`observation → policy → action mapping → actuator → physics → new observation`

每个引擎都必须根据自己的仿真状态重新计算 observation 和 action。

测试场景至少包括：
- 前进；
- 后退；
- 左移；
- 右移；
- 左转；
- 右转；
- 前进左弧线；
- 前进右弧线；
- 停止保持。

每个场景至少 3 个 seed；如果仿真成本较高，先跑 1 个诊断 seed，确认没有接口错误后再补 2 个 seed。

如存在 120 s 实机操作波形，检查它是否适合当前策略动作合同。若适合，再完整回放，不得截断关键长转窗口。

## 6.5 MuJoCo 评测指标

必须报告：
- 运动速度跟踪；
- yaw 跟踪；
- roll/pitch；
- 四轮接触；
- 轮速目标与实测；
- 轮端力矩；
- 轮胎滑移；
- 腿部抬落次数；
- 步态周期；
- 关节活动范围；
- 后小腿净空；
- 非轮触地；
- 跌倒；
- 零指令漂移；
- 转向停止后的驻停表现。

对比必须注明：
- checkpoint SHA；
- policy/action/observation contract；
- physics config SHA；
- MJCF SHA；
- dt、control dt 和记录频率；
- seed；
- actuator 参数；
- contact 参数。

## 6.6 Phase 4 交付

`docs/as2w_policy_validation_20261119/04_MUJOCO_SIM2SIM_VALIDATION.md`

同时保存：
- `MUJOCO_PARITY.csv`
- `OPEN_LOOP_PLANT_COMPARISON.csv`
- `CLOSED_LOOP_SIM2SIM.csv`
- 对比视频；
- 失败案例和原因分类。

如果当前服务器没有可用 MuJoCo 环境或正确模型，明确记录阻塞点，不要伪造通过结果。

---

# 7. Phase 5：域随机化现状审计与新方案设计

**目标：先判断当前随机化是否有效，再决定要不要增加随机化。**

## 7.1 先审计实际生效的随机化

对当前策略训练和 Rob2Nav 中每个相关 run 分别检查：

- mass；
- COM；
- inertia；
- wheel radius；
- wheel armature；
- wheel damping；
- wheel/ground static friction；
- wheel/ground dynamic friction；
- lateral friction 或侧向接触模型；
- rolling friction；
- torsional friction；
- contact stiffness/damping；
- actuator Kp/Kd；
- torque/velocity limits；
- joint zero offset；
- actuator delay；
- observation noise；
- observation delay；
- command noise；
- initial joint pose；
- initial base pose；
- terrain geometry。

对每项记录：
- 源码文件和函数；
- 配置范围；
- 分布类型；
- 每次 episode、每次 reset 或每步更新；
- 作用对象；
- 是否真正启用；
- 评测时是否启用；
- 是否有运行时 readback 或数值验证。

区分：
- 代码里有配置；
- 训练 run 确实启用；
- 运行时参数确实生效；
- 评测确实覆盖了该随机化。

这四者不能混为一谈。

## 7.2 随机化设计原则

先建立经过已有实机数据约束的 nominal plant，再围绕剩余不确定性设计随机化。

必须遵守：
1. 不要用宽范围随机化掩盖确定性的资产错误。
2. 有实测值和误差范围的参数，优先使用测量置信区间。
3. 没有测量依据的参数，先标记为假设，不要直接设定任意大范围。
4. 具有物理相关性的参数，考虑联合或相关采样，不能一律独立均匀随机。
5. 训练随机化范围和测试保留范围分开。
6. 保留 nominal、in-distribution 和 OOD 三类评测。
7. 地形随机化与轮胎/执行器动力学随机化分开设计。
8. 不能把其他机器人 terrain run 的随机化参数直接移植到 AS2-W。

## 7.3 建议评估的随机化维度

根据当前实际误差决定是否纳入：
- 质量和载荷；
- COM；
- 惯量；
- 轮端等效惯量；
- 轮胎几何和接地点；
- 纵向/侧向接触；
- 静摩擦/动摩擦；
- 执行器增益；
- 执行器延迟；
- 关节零位偏差；
- 观测噪声；
- 控制时序抖动；
- 地面坡度、台阶和起伏。

每个维度必须说明它对应哪项实机误差，以及当前数据是否支持该范围。

## 7.4 域随机化消融建议

不要一开始把所有随机化一起打开。

先设计可区分原因的消融：
- DR-0：nominal plant，无随机化；
- DR-1：只随机化已测量不确定的执行器参数；
- DR-2：在 DR-1 基础上加入质量/COM；
- DR-3：在前述基础上加入轮地接触；
- DR-4：再加入时序、观测噪声和初始状态；
- DR-5：独立 OOD 组合测试。

具体参数范围必须根据审计结果填写，不得在本阶段凭空指定。

## 7.5 Phase 5 交付

`docs/as2w_policy_validation_20261119/05_DOMAIN_RANDOMIZATION_AUDIT_AND_PLAN.md`

同时输出：
- `DOMAIN_RANDOMIZATION_MATRIX.csv`
- `DR_ABLATION_MATRIX.csv`
- 每项随机化的实测依据、未知项和风险。

本阶段只做审计和方案设计，不启动 DR 训练。

---

# 8. Phase 6：训练结果与仿真结果综合判断

**目标：给出明确的继续训练决策，而不是只写“建议继续观察”。**

## 8.1 需要区分的四类问题

### A. Policy-limited
资产和控制合同可信，PhysX/MuJoCo 基本一致，但策略本身不能完成目标动作。

下一步：针对步态、奖励或网络做受控训练。

### B. Plant-limited
策略输出正确，但执行器或接触模型无法交付目标运动，或仿真与实机的物理差异仍明显。

下一步：先做物理模型和控制可达性分析，不应直接增加 PPO 轮数。

### C. Sim2Sim-mismatch
Isaac Lab 通过，MuJoCo 失败，且接口 parity 或物理参数不一致。

下一步：定位 observation/action mismatch、执行器模型或接触几何差异。

### D. Generalization-limited
nominal plant 下通过，但在合理的参数变化或未见地形下失败。

下一步：基于实测不确定性做有针对性的 domain randomization 或课程学习。

## 8.2 继续训练的必要条件

只有同时满足以下条件，才建议进入下一轮训练：
1. 当前训练 checkpoint 和训练合同已经确认；
2. Isaac Lab 的步态评测确认存在明确、可重复的策略缺陷；
3. nominal plant 没有明显的系统性资产错误；
4. 该缺陷不是 MuJoCo 接口错误或接触模型错误造成的；
5. 现有奖励和观测确实能够为目标动作提供有效学习信号；
6. 有一个可比较的基线和独立测试集。

如果条件不满足，暂停 PPO，先解决相应问题。

## 8.3 下一轮训练设计要求

本任务不启动新训练，但需要在报告中给出一份有明确变量控制的训练建议：

- 从哪个 checkpoint 开始；
- 是否冻结原策略主体；
- 哪些网络参数允许更新；
- 是否需要调整 reward；
- 是否需要修改 observation normalization；
- 是否需要 DR；
- 每个实验只改变哪些变量；
- 预计训练迭代数和评测间隔；
- 每个 checkpoint 如何验收；
- 失败后如何决定停止。

至少比较以下可能路线：
1. 不训练，只保留当前 residual checkpoint；
2. residual policy 小幅继续训练；
3. 修正 reward 后从 residual checkpoint 继续训练；
4. 先做 nominal plant 修正，再做 residual fine-tune；
5. 保留原策略，使用独立辅助控制或动作约束，而不是直接改 PPO。

不要把“继续训练”设为默认选项。

## 8.4 Phase 6 交付

`docs/as2w_policy_validation_20261119/06_FINAL_DECISION_AND_NEXT_TRAINING.md`

必须明确给出：
- 当前最佳 checkpoint；
- 是否建议继续训练；
- 如果继续，原因和唯一首要修改项；
- 如果暂停，具体阻塞因素；
- 是否通过 Isaac Lab；
- 是否通过 MuJoCo；
- 是否通过 DR；
- 是否建议进入真机测试（仅建议，不构成授权）。

---

# 9. 总交付目录与文件管理

所有 Markdown、CSV 和汇总文档放在：

`docs/as2w_policy_validation_20261119/`

建议目录结构：

```text
docs/
└── as2w_policy_validation_20261119/
    ├── 00_SERVER_AND_RUN_DISCOVERY.md
    ├── 01_ISAAC_POLICY_VIDEO_AND_GAIT_EVAL.md
    ├── 02_ASSET_AND_ROBNAV_AUDIT.md
    ├── 03_REAL_DATA_REUSE_AND_GAP_AUDIT.md
    ├── 04_MUJOCO_SIM2SIM_VALIDATION.md
    ├── 05_DOMAIN_RANDOMIZATION_AUDIT_AND_PLAN.md
    ├── 06_FINAL_DECISION_AND_NEXT_TRAINING.md
    ├── ASSET_MATRIX.csv
    ├── RUN_MATRIX.csv
    ├── TERRAIN_RUN_MATRIX.csv
    ├── DOMAIN_RANDOMIZATION_MATRIX.csv
    └── DR_ABLATION_MATRIX.csv
```

大型视频、原始 CSV/NPZ 和评测日志放在：

`runs/as2w_policy_validation_20261119/`

例如：

```text
runs/as2w_policy_validation_20261119/
├── phase1_isaac/
│   ├── videos/
│   ├── csv/
│   ├── plots/
│   └── raw/
├── phase4_mujoco/
│   ├── videos/
│   ├── csv/
│   ├── plots/
│   └── raw/
└── manifests/
    ├── checkpoint_sha256.txt
    ├── asset_sha256.txt
    └── environment.txt
```

不要把文档、视频、模型、临时脚本散落在项目根目录或其他无关目录。

---

# 10. Agent 执行顺序与自动推进规则

## 第一批：必须先做

1. Phase 0 当前服务器识别；
2. 找到当前策略两条续训线和 checkpoint；
3. 找到当前服务器 Rob2Nav 和 terrain 相关 run；
4. 建立 checkpoint 和资产 SHA256 清单；
5. 确认训练和评测合同。

## 第二批：继续执行离线评测

在不干扰当前训练任务的前提下：
1. Phase 1 生成 Isaac Lab 对比视频；
2. 计算步态、抖动、接触和跟踪指标；
3. 输出三策略对比报告；
4. Phase 2 完成当前资产和 Rob2Nav/terrain 审计。

如果 headless 视频渲染不可用，先检查已有渲染脚本和环境；不要安装软件或改系统。说明阻塞点并继续可执行的数值评测。

## 第三批：复用实机校准和做跨仿真验证

1. Phase 3 复用已有实机数据和校准结果；
2. 不重复完整数据校对；
3. 定位当前资产与历史校准合同的差异；
4. Phase 4 检查 MuJoCo 模型；
5. 完成接口 parity、开环物理对比和闭环 Sim2Sim。

如果缺少 MuJoCo 模型，输出明确阻塞项和转换方案，不得把其他机器人模型代替。

## 第四批：给出 DR 和训练决策

1. Phase 5 审计实际 DR；
2. 给出基于实测误差的 DR 方案；
3. Phase 6 综合 Isaac Lab、实机数据和 MuJoCo 结果；
4. 给出是否继续训练的明确结论。

## 10.1 可以自动继续的工作

只要不涉及训练、真机和破坏性修改，Agent 不需要每完成一个 Phase 就停下来等用户：
- 可以连续完成目录审计；
- 可以连续完成 checkpoint 离线评测；
- 可以生成视频和 CSV；
- 可以读取已有实机校准数据；
- 可以检查已有 MuJoCo 模型；
- 可以做离线 Sim2Sim；
- 可以生成 DR 设计和最终报告。

## 10.2 必须停止并报告的情况

遇到以下情况必须停止对应操作：
- 当前有训练进程，评测会明显争抢 GPU 或 CPU；
- 发现需要修改原始训练源码或旧资产；
- 需要覆盖旧 checkpoint；
- 需要下载大量外部文件且网络/存储影响不明确；
- 找不到同一策略对应的 MuJoCo 模型，却需要改变动作合同才能运行；
- 需要新的实机运动数据；
- 任何真机运动、控制增益或力矩权限变更。

---

# 11. 最终回答必须回答的核心问题

完成任务后，最终总结不要只列文件名。必须直接回答：

1. 当前服务器究竟有哪些 AS2-W 训练和仿真环境？
2. 当前两条续训线实际用了什么资产？
3. Rob2Nav 中哪些资产和 terrain run 对当前 AS2-W 有参考价值？哪些不是同一机器人或同一训练链？
4. 当前 residual 策略在 Isaac Lab 中是否真的更好？改善和退化分别是什么？
5. 是否已经复用了 `SimDynamicsValidation_AS2W_20261114` 的实机校准结果？哪些校准被当前训练采用，哪些没有？
6. `as2w-yaw-physics-aware-retrain-20261115` 的资产修改和随机化是否与当前训练有关？是否有证据支持复用？
7. 当前 MuJoCo 模型是否与 Isaac Lab 对应？接口 parity、开环和闭环分别通过了什么？
8. 当前域随机化覆盖了什么，遗漏了什么？
9. 当前策略最大的限制属于 Policy-limited、Plant-limited、Sim2Sim-mismatch 还是 Generalization-limited？
10. **现在是否应该继续 PPO 训练？如果应该，下一轮只改什么？如果不应该，先解决什么？**

最终结论必须由当前服务器上真实找到的源码、checkpoint、日志、资产和评测结果支持。历史报告只能作为线索，不得冒充当前服务器实测证据。

**本任务最终目标是做出一个有证据的训练决策，而不是为了完成任务书而强行启动下一轮训练。**
