# AS2-W R4000 Rolling：E3-P4 训练推进与长转停车 Debug 完整 Agent 任务书

> 日期：2026-11-14  
> 范围：R4000 Rolling 训练、轮动作 PPO、MuJoCo 回归、12 s 转向后的姿态和停车问题  
> 状态：**授权开展独立目录下的仿真诊断与短中程训练；不授权实机部署或更改实机电机参数。**  
> **本文是独立执行任务书**。按本文推进，不需要再向用户逐项申请 100/300/500 iteration 的训练许可。不同训练分支必须保留原始模型和源码，禁止覆盖基线与其他运行中的进程。

## 0. 执行摘要：现在该干什么

**不再继续纯零训练审计，应立即从“可训练的轮动作通道”入手，跑同起点的训练消融；同时单独排查 12 s 长转向后腿部姿态误差。**

已获得的事实：

1. R4000-3999 的轮输出 `clip1_fraction=1.0`，A+B 缩放后仍为 1.0；二者在测试样本中对映射后的轮目标无可观测有效差异。
2. A+B 恢复部分中间层响应、并使短程 MuJoCo 由 6/12 到 9/12，但消融显示主要改善来自腿部输出变化；它不是一个已解决轮控制学习的最终权重。
3. 历史 S2 模型的 `clip1_fraction=0.0667`，出现非零学习残差；对应的 RSL-RL 自定义 PPO 在 **裁剪前高斯均值** `distribution_params[0][:,12:]` 上添加 `wheel_saturation` 正则项，其权重 0.025、margin 0.9。**历史 S2 与 R4000-3999 不同父模型，不能直接拿它作性能比较或直接替换权重。**
4. 前次“PPO surrogate 对轮输出层梯度为零”结论因探针将 `a=mu+sigma*eps` 直接接入 `log_prob` 而产生解析抵消，**已撤回**。正确的 PPO 更新使用 rollout 期间保存且脱离当前计算图的 actions / old log_probs，不要继续以该错误探针阻止训练。
5. 12 s MuJoCo 偏航转速没有明显塌缩，但后续 `joint_pose` 失败、部分场景膝净空降到约 0.024–0.048 m，需独立查执行过程、状态转移、腿目标滤波及姿态误差；**MuJoCo 12 s 表现不能替代实机 3–5 s 轮差塌缩验证**。

**立即执行三条工作线**：

- **线 A（主线）：**核实 R4000 当前实际 Trainer → 准备 A+B、A+B+历史轮输出正则、A+B+轮头重初始化、A+B+两者组合 → 每组 500 iteration → 按门槛自动延长最优组到 2000/4000 iteration。
- **线 B（并行）：**零训练排查 12 s `joint_pose` 和 HOLD 失败，提供每关节时间曲线、失败第一帧和真正的 `held` 原因；验证是否需改状态机/腿动作，而不是盲调 PPO。
- **线 C（并行）：**对最终候选同时用短程 6 s 和长程 12 s 合同评测，检查 ONNX 与 Python、仿真与实机控制链路。**暂不动实机 kd=2 和差速前馈系数 0.66。**

## 1. 必须先更正的一处“训练框架混用”问题（可与准备训练并行，不能略过）

E3-P3 报告引用 `train_stair_low_speed_course.py`，判断训练使用 Stable-Baselines3（SB3）PPO；S2 报告实际分析的是 `as2w_rolling_regularized_ppo.py`（RSL-RL 风格）。这两套框架**不能默认是同一个 R4000 主训练链**。

历史 R4000 的真实训练快照：

- `training/as2w-backup/as2w-current/runs/rolling/filtered-4000-env4096-stage020.ovvRL3/run.sh` 使用 `/isaac-sim/python.sh` 启动 `train_as2w_isaac_robust.py`，`--num-envs 4096 --iterations 4000`；
- 入口内使用 `RslRlVecEnvWrapper`、`create_rsl_rl_runner()`；
- 训练时把 `runner_cfg.algorithm.class_name` 指向 `as2w_rolling_regularized_ppo:PPO`。

这证明**历史 4000 轮 Isaac 主训练使用 RSL-RL**。但最新 `runs/r4000-turn-20k` 的实际入口必须从当前服务器脚本、日志首行和 run-contract 单独确认，不能仅凭名称推定。

立刻提交 `ACTIVE_TRAINER_CONTRACT.md`：列出**正在继续训练的确切模型**、启动脚本、`python` 入口、PPO 类路径、算法版本、Actor 输出层、优化器、checkpoints 结构、rollout 与 update 过程。沿着一次真实 `run.sh → entrypoint → runner → alg.act → alg.update → loss.backward → optimizer.step` 追踪。不要拿另一个 MuJoCo SB3 训练脚本证明 Isaac Runner 的反传逻辑。

操作规则：若最新训练使用 RSL-RL，把历史 `wheel_saturation` 代码作为最小差异迁移；若使用 SB3，则在其真实 `policy` 和 loss 中实现**语义等价**的裁剪前均值正则，不能复制 RSL-RL PPO 类。先在真实 minibatch 运行一次反传及参数增量检查，无须长时间停在审计阶段。

关键源码历史快照：

- [历史 R4000 训练脚本](https://github.com/YuTaoV5/Rob2Nav/blob/2de96cbace96ac03fbe6adb9f249525aeec2d4cc/training/as2w-backup/as2w-current/runs/rolling/filtered-4000-env4096-stage020.ovvRL3/run.sh)
- [历史 RSL-RL 入口](https://github.com/YuTaoV5/Rob2Nav/blob/2de96cbace96ac03fbe6adb9f249525aeec2d4cc/training/as2w-backup/as2w-current/runs/rolling/filtered-4000-env4096-stage020.ovvRL3/source/train_as2w_isaac_robust.py)
- [S2 PPO 正则损失](https://github.com/YuTaoV5/Rob2Nav/blob/2de96cbace96ac03fbe6adb9f249525aeec2d4cc/training/as2w-backup/as2w-current/runs/rolling/actor-wheel-saturation-loss.Uliv0s/source/as2w_rolling_regularized_ppo.py)

## 2. 技术判断：真正的学习障碍是什么

### 2.1 不要再把环境裁剪等同于 PyTorch 梯度为零

假设训练时动作来自 `a ~ N(mu,sigma)`，rollout 使用 `a.detach()` 记录，PPO 在新的 `mu` 下计算 `log_prob(a)`。虽然环境中的 `clip(a,-1,1)` 不参与反向传播，PPO 的 score-function 梯度仍可以非零。但当 `|mu|` 远大于 1 且 `sigma` 很小时，绝大部分采样动作都在相同的 ±1 平台上，动作探索难以改变执行器输入，奖励对轮动作的可辨别信息极弱。这是**有效控制权不足和信用分配弱化**，不是严格数学梯度阻断。

旧探针 `a=mu+sigma*eps`（**不 detach**）再代回同一 `log_prob(a;mu)`，因 `a-mu=sigma*eps`，局部梯度会抵消，不能作为 PPO 权重是否可训的证据。新探针应严格复现真实 rollout 的 `actions.detach()`、`old_log_probs.detach()`、advantages、当前 policy `log_prob(actions)`、PPO ratio 和完整 loss，并记录 `gradient_norm`、`parameter_delta`、KL、entropy、ratio-clip fraction。

### 2.2 轮输出范围才是第一个应解决的训练问题

当前对比（报告给出的共同观测集）：

| 模型 | h4 活性 | clip1_fraction | 学习差动残差 std | 解释 |
|---|---:|---:|---:|---|
| R4000-3999 | 0/128 | 1.0000 | 0 | 常量化，输出饱和 |
| A+B | 128/128 | 1.0000 | 0 | 中间层有效，但终端仍裁剪 |
| 历史 S2-1200 | 36/128 | 0.0667 | 0.01157 | 轮输出越界正则有用，仍未证明任务性能足够好 |

不能仅因 `h4` 存活率高，就判定下游能控制车轮；也不能仅因 `clip1 < 0.2` 就判定转向真正学会。必须看**相同或镜像状态下 yaw 指令变化引起的学习差动响应**及其对跟踪的实际收益。

### 2.3 历史轮动作正则的具体作用和边界

S2 历史项（仅在其真实 RSL-RL 训练链中可直接照此实现）：

```python
wheel_mean = distribution_params[0].float()[:, 12:]
wheel_saturation = torch.nn.functional.smooth_l1_loss(
    wheel_mean, wheel_mean.clamp(-0.9, 0.9).detach()
)
loss = base_ppo_loss + 0.025 * wheel_saturation
```

注意 `base_ppo_loss` 只是示意名称；必须把该项添加进**现有完整 PPO loss**（保留腿部与 symmetry 等原有项），不能以此覆盖原损失。损失项必须在**裁剪前的高斯均值**上计算，且日志独立输出 `wheel_saturation`。`0.025` 是历史可工作的起点，不保证对 R4000 一定最优；按结果可比较 0.01 / 0.025 / 0.05，但不要一开始就展开大规模网格搜索。

**纠正先前一个错误提议：**“只在 0 iteration 添加 wheel_saturation 再进行前向评测，看 clip1 是否变小”没有物理意义，因为增加损失本身不会改变权重。0 iteration 可用来测量损失值、梯度方向和将来一小步参数更新的预期，但去饱和必须实际优化或手工重置轮头。

## 3. 线 A：现在就跑训练——固定起点的完整对照

### 3.1 四组独立候选

所有实验保存相同的基线 hash，采用**相同的命令分布、随机种子、环境设置和评测器**；在第一阶段禁止同时改轮端 kd、差速前馈、动作映射、奖励和命令采样。优先使用 `model_3999_AplusB.pt` 作为共同起点，并保留 upstream-3999 作为参照。

| ID | 初始化 / 训练项 | 目的 | 执行决定 |
|---|---|---|---|
| **R0** | A+B 原始模型，0-iter 基线（已知 9/12） | 对照 | 只评测，不再无目的长训 |
| **R1** | A+B + S2 `wheel_saturation`（0.025, 0.9） | 验证输出正则能否从 `mu≈4` 拉回有效范围 | **直接训练 500 iter** |
| **R2** | A+B + 四轮输出头低幅重新初始化，保持原 PPO 损失 | 验证干净轮初始化的学习速度 | **直接训练 500 iter** |
| **R3** | A+B + 同样的轮头低幅初始化 + S2 正则 | 同时恢复控制范围并持续防止再次饱和 | **直接训练 500 iter** |
| R4（补充） | upstream-3999 + S2 正则 | 检验 A+B 是否必要，排除初始化依赖 | 如资源允许再跑 500 iter |

R2/R3 的四个轮输出行应使零命令/低速下的均值靠近 0、`clip1_fraction` 降低；只修改轮输出行及其 bias，不要重置 12 维腿动作行，不要更换 65D 观测合同或改变现有控制前馈。建议先比较两种保守初始化（全零 bias + 极小权重，或将旧行整体缩小至输出尺度 <0.3），做 0 iteration 安全回归后选表现更稳的一种。**零均值轮动作并不代表机器人轮速为零**：固定映射前馈仍存在。

注意：在当前消融中，A+B `no-wheel` 仍达到 8/12、无 unstable，这支持将 A+B 的轮头复位作为可行训练起点，但**不能保证所有新初始化都安全**。

### 3.2 自动执行日程（不再每隔几百轮等待批准）

- 训练前仅核对真实 Trainer、loss 正确挂接、四轮输出行确实可更新，并完成 0-iter 有界动作和安全 smoke test。若失败，修复实现后继续，不要求用户逐项批复。
- R1/R2/R3 各自执行 **500 iteration**（资源允许时额外 R4）；记录 0、100、300、500 的 checkpoint 和完整 metrics。
- 对每组在固定短程 6 场景×至少 2 种子回归；优胜者至少扩展到 **3 个独立种子**，其中包含训练种子外的测试种子。安全和详细报告不要只看 12 次 pass 计数。
- 若 500 轮后控制权指标与轨迹都在改善、未出现非轮接触/失稳，可**自动继续最佳组到 2000 iteration**；如 2000 轮后长转和停车仍持续改善，可**自动继续到 4000 iteration**。不需再回来询问是否允许从 500 到 2000。
- 若 500 后 clip1 已恢复但任务成绩未改善，先切换到定向命令训练与姿态问题修复，不要把“去饱和”当成已完成的运动控制。
- 若 R1、R2、R3 全部无条件响应（随 yaw 变化的学习差动仍为零），则再启动 **M3：健康初始化共享 Actor / 轮腿分支解耦**。M3 是备选，不是本轮第一实验。

### 3.3 推荐记录的训练指标

必须按训练环境与独立评测分别记录：

1. `mu_wheel` 每通道均值、p50/p95/p99、绝对最大值；`sample_action`；`clip6_fraction`、`clip1_fraction`（必须按四个轮分别统计）；`log_std_wheel`（如共享 std 则记录真实分布参数）。
2. 实际 mapper 输出（**归一化动作**）以及 ×16 后的轮目标（**rad/s**）；`u_common/u_yaw/u_front_rear/u_diag`；分别记录策略贡献和固定前馈贡献，注明计算公式，**不要把方差比例直接解释成实际偏航贡献百分比**。
3. 固定状态下扫 `yaw=−0.10,−0.08,−0.04,0,+0.04,+0.08,+0.10`，测学习差动的斜率、左右镜像误差、与 `yaw_cmd` 的符号相关性；另外固定 yaw、变化机身实际 `gyro_z` / 轮速，检验反馈不是纯命令查表。
4. PPO `surrogate/value/entropy/regularizer`、KL、PPO ratio clip fraction、梯度范数、参数实际更新幅度、learning rate；拆分轮头、腿头、共享层监控。
5. `tracked`、`held`、`unstable`、nonwheel、最小膝净空、`vx/yaw` 误差、停车位移、停车后航向漂移、末 5 s 偏航跟踪、腿部关节误差及退出窗口失败次数。

### 3.4 建议阶段门槛（用于自动选择，不是单一通过率指标）

**500 iteration 的候选继续条件**：`clip1_fraction < 0.2`（目标值）、学习差动在相同状态不同转向指令下有可辨别的有符号响应（建议残差 std >0.005，仅作初筛），并且安全无新增 `unstable`/nonwheel。短程得分应接近/不低于 A+B 原始 9/12；如暂时 7–8/12 但控制权显著恢复且后续趋势稳定，可继续到 1000–2000 轮，不要机械按照 9/12 一票否决。最终须以独立种子复核。

**强制停止某个分支**：NaN/Inf、轮目标超过既有物理限幅、持续失稳/非轮接触、严重姿态恶化、训练图与部署映射不一致、或跨多个 checkpoint 的性能与动作控制权同时持续退化。停止**当前分支**不等于结束全部任务；应转入其他分支。

### 3.5 关于轮控制权限不足——第二阶段再改 mapper

原映射用固定差速前馈 `−0.66*wz*side`，同时依赖有限的 learned residual。恢复可训练轮输出后，若残差对偏航仍贡献很小：

- 用 `common/yaw/front_rear/diagonal` 正交分解显式区分四轮动作；保留共同驱动与差动的独立限幅。
- 验证 `yaw_gate`、`yaw_residual_limit` 两条差动通路的符号和响应，`yaw=0` 不应有无解释的偏航残差；检查左右镜像。
- 逐档测试 yaw 残差增益而不是立即把 `0.66` 减小；前馈与残差要分别统计。
- 改 mapper 以后**必须重新训练相应模型**，不能只拿旧 ONNX 对新映射做零训练评测后声称没有收益。

**第一阶段必须坚持动作/训练改动与 mapper 改动解耦**，才能知道最终收益来自哪里。

## 4. 线 B：12 s 转向后的 `joint_pose` / HOLD 失败单独攻关

### 4.1 不要把“joint_pose 100/100 帧失败”直接解释成 12 秒持续积累

最新长合同：12 s 运动 + 10 s 停车，MuJoCo 偏航率约 0.09–0.096 rad/s，并无明显偏航塌缩；但转向与弧线出现 `joint_pose` 约 0.180–0.209 rad，局部膝净空低至约 0.024–0.048 m。这里仍有三件事**没有确认**：

- `joint_pose` 具体是运动中逐渐偏离，还是释放命令时切换姿态导致的跳变？100/100 计数是哪些状态/时间窗口？
- 报告中 `held=False`，但 12 s 合同部分 `hold_displacement_m`、`hold_heading_deg` 为空；是没有进入 HOLD、评测提前拒绝，还是字段计算缺失？
- `fixed_exit_failed` 单字段为 False，但详细 `fixed_exit_failed` 字典里 `joint_pose:100`。两个名字所表示的逻辑并不一致，必须追溯判定代码。

**因此第一步不是再给 PPO 加腿姿态奖励，而是拿到逐帧状态与错误发生点。**

### 4.2 立刻做 4 组定向零训练实验

在同一模型同一种子同一物理和评测条件下：

| 组 | 干预 | 识别的问题 |
|---|---|---|
| L0 | 原始 6 s/12 s；运动到放杆完整记录 | 明确错误时间点 |
| L1 | 保持原轮控制，运动/停止阶段都去除腿 Actor residual，仅保持安全的确定性站姿 | 判断腿 Actor 造成多少关节误差 |
| L2 | 运动阶段原腿 residual，仅进入 BRAKE/SETTLE 后衰减至零，并采用连续渐变 | 判断停车切换导致的姿态跳变 |
| L3 | 在 L0 基础上独立扫描腿目标滤波、参考捕获时机及腿关节跟踪（只在仿真） | 查控制滤波、参考值或执行器延迟 |

逐帧记录 `cmd_vx/cmd_wz`、状态机 ID、每个腿关节 `q/q_des/dq/torque`、动作原始值与裁剪值、滤波腿目标、腿关节误差峰值和发生时间、接触、膝净空、roll/pitch/heading、wheel actual/target，以及从 12 s 放杆到进入 HOLD 的每个门槛判定。

重点检查 `rolling_leg_filter_hz=5`、`rolling_precalibrated_stand`、`rolling_wheel_hold`、固定退出窗口的 joint reference、后腿目标/补偿项、`previous_action` 是否在 reset 时正确复位（报告提出有疑点，尚未证明是 bug）。改变滤波前必须保存可回放的原始轨迹，禁止多项同时改。

**优先修复控制状态切换或参考捕获问题**；若验证是运动阶段持续腿姿态误差，再将 12 s 长转向和 STOP/HOLD 联合训练纳入 RL。

### 4.3 长程训练的时间合同要正确

历史 Isaac 配置中 `episode_length_s=10`，命令驻留约 1.8–3 s；这不足以完整覆盖 **12 s 运动 + 10 s 停车**。最新训练配置未核实，应检查当前 episode length、termination、command resampling、rollout 长度，避免在策略未看到长任务时要求它通过长任务。若当前仍是 10 s，则把用于长转/停车课程的 episode 长度扩到至少 25–30 s，分配持续转、弧线、BRAKE、HOLD，并确保重置逻辑一致。

训练奖励应同时看低速弧线 `vx` 与 `wz`，并将停车后的**积分航向误差/保持位移**单独评估；不能仅以瞬时 `|gyro_z|` 代替 10 s 内累计的航向漂移。需要相对 heading reference 时，应确认 Actor 实际可观测到该误差或由部署时的 STOP 控制器进行补偿，不能只给 Critic 真值却要求 Actor 主动纠正不可观测误差。

### 4.4 停车不是只看一个 `held`

短合同有 `hold_heading_deg` 大于 1° 的样本，12 s 合同又有空缺字段。分别报告：

- 正确进入 BRAKE / SETTLE / HOLD 的比例与进入时刻；
- `brake_distance_m`、`hold_displacement_m`、`hold_heading_deg`、`hold_speed_p95`、`final_body_speed`、关节姿态误差；
- 严格区分 **未进入保持阶段**、**保持阶段超限**、**碰撞/安全失败**、**字段未计算**，不得将空值一律写为 not_held；
- 与短合同同种子相比，12 s 的每项增量及首次越界的时间。

## 5. 线 C：仿真和实机分界，当前不用等待硬件才能训练

1. MuJoCo 12 s 下 yaw 基本保持，不代表真实 AS2-W 的长时程差速塌缩已经解决。前一批实机约 3–4 s 后左右真实轮速差下降，可能来自轮端伺服、轮胎横向滑移、支撑负载或接触模型差异。
2. 不修改真实 `kd=2`、硬件电机限幅或其他服务，也不直接将模拟最优策略推到实机。先在独立仿真回放中做 wheel target vs wheel actual、扭矩限制、姿态/轮载、接触和轮胎摩擦对照。
3. 确认 `wheel_map` 的动作单位：轮输出归一化，经 `×16` 才是轮目标 rad/s；保证训练/评测/ONNX/C++ 一致。
4. 检查 Python 确定性 actor、导出 Hardtanh ±6、ONNX 以及 mapper `clip1` 的数值 parity。若训练后最大绝对输出仍超 ±6，导出可能压缩输出；生产模型必须给出误差和后果评估。
5. 不准用放宽 `held` / `tracked` 阈值掩盖失败。

## 6. 日志和交付要求（一次完整提交，避免反复审批）

在独立目录建议 `runs/e3-p4/` 下提交：

- `ACTIVE_TRAINER_CONTRACT.md`：真正运行的 PPO 框架、入口、算法、更新路径、checkpoint、环境、部署动作链；明确与另一套训练系统的区别。
- `EXP_MATRIX.md` 和运行命令；各组固定配置、种子、初始权重 SHA、源码 SHA、训练损失改动 diff。
- `R0_R1_R2_R3_metrics.csv`：0/100/300/500/1000/2000/4000 视实际运行记录梯度、裁剪、动作响应和奖励；更新点没有达到的必须标 N/A，不能补造。
- `same_state_yaw_sweep.csv`：正负 yaw 响应及镜像检验。
- `short6s_seed_comparison.csv`：六场景、至少 3 种子，逐场景 tracked / held / unstable / knee / pass。
- `long12s_joint_trace.csv`：每关节 q/qdes/滤波/力矩/状态机/首次失败帧，附曲线或图表。
- `hold_failure_audit.md`：解释 12 s 表中 HOLD 空值及 `fixed_exit_failed` 字段的真实含义。
- `RECOMMENDATION.md`：选出一个下一阶段候选及依据，并写明是否可以进入 4000+ iteration；若失败给出已执行的排查和下一次具体补救，不仅仅回复“请求用户批准”。

自动决策规则：如果任一组出现稳定且可测的轮控制权、短程无新增安全退化、长程趋势改善，直接按前述条件扩展 2000/4000 iter；若轮输出正常但任务仍未提升，则开始轮残差 mapper 消融与长转课程；若 3 组都无条件响应，再考虑 M3 独立重训。**除实机动作、修改已有任务共享资源或引入新风险外，不要每个实验都暂停请求批准。**

## 7. 给执行 Agent 的最终指令（可直接作为任务文本）

> 现在进入 E3-P4。停止把旧 `ppo_update_audit.csv` 的 zero-grad 当作 PPO 不可训证据；该探针已被 S2 的 detach 对照证伪。先在实际运行入口确认 R4000 采用 RSL-RL 还是 SB3，避免混用不同项目源码与 loss。随即在独立 `runs/e3-p4/` 以 A+B 为共同起点并行建立 R1（+历史 wheel_saturation 正则）、R2（低幅轮头重置）、R3（两者组合），各执行 500 iteration，留 0/100/300/500 checkpoint。全过程固定 mapper、轮端控制、奖励与命令分布，不覆盖任何基线；用真实 PPO rollout loss 检查梯度。选优依据是 clip1、随 yaw 指令变化的策略轮差动、真实跟踪、held、安全和长转关节状态的共同改善，而不是仅看 h4 存活或一次 9/12。满足安全和改善条件者自动扩训至 2000，再至 4000 iteration，无须逐段请示。
>
> 另外并行做 12 s 长转 joint_pose 诊断，区分运动累积偏差、放杆状态转换跳变、滤波/目标参考和 HOLD 评测字段缺失，输出逐关节和逐帧证据；当短程主线训练完成，优先将已确认的退出/保持缺陷修复到独立分支。严禁修改实机 kd=2 或直接部署实机策略。最后一次性提交训练矩阵、源码 diff、SHA、轨迹、数据表、评测结果及下一阶段自动推进建议。

---

## 8. 报告数据来源与可追溯性说明

本任务书依据 2026-11-14 两份由手机截图转写的服务器报告：

- `AS2W_R4000_E3P3零训练审计与六份交付_手机截图汇总_20261114.md`（含完整零训练审计、消融表和 24 行长转 CSV；131 KB 动作遥测只提供样例和聚合）。
- `AS2W_R4000_S2历史分支复核T0T1T2_手机截图汇总_20261114.md`（含历史梯度探针更正、统一观测集对照和正则项源码差异）。

历史 GitHub 快照用于核实源代码入口与 RSL-RL 自定义 PPO 的存在；**当前服务器工作树、最新 run-contract 和正在运行的分支仍需 Agent 以现场文件为准**。凡属本文建议的阈值与实验设计，都是下一轮工程验收标准，不是已经实测的事实。
