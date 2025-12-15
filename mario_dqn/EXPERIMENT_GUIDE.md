# Mario DQN 实验参数与Wrapper完整指南

## 📋 目录
1. [命令行可调参数](#1-命令行可调参数)
2. [配置文件可调参数](#2-配置文件可调参数)
3. [可用Wrapper分类](#3-可用wrapper分类)
4. [推荐实验方向](#4-推荐实验方向)

---

## 1. 命令行可调参数

### 运行命令格式
```bash
python3 -u mario_dqn_main.py -s <SEED> -v <VERSION> -a <ACTION> -o <OBS>
```

### 参数说明

| 参数 | 全称 | 可选值 | 默认值 | 说明 |
|------|------|--------|--------|------|
| `-s` | `--seed` | 任意整数 | 0 | 随机种子，影响训练随机性 |
| `-v` | `--version` | 0/1/2/3 | 0 | 游戏版本，控制图像复杂度 |
| `-a` | `--action` | 2/7/12 | 7 | 动作空间大小 |
| `-o` | `--obs` | 1/4 | 1 | 观测叠帧数量 |

### 游戏版本对比 (`-v`)
- **v0**: 原始图像（最复杂，84x84 彩色转灰度）
- **v1**: 降采样图像（中等复杂度）
- **v2**: 简化背景（去除部分冗余信息）
- **v3**: 极简化（仅保留关键元素）

### 动作空间对比 (`-a`)
- **2动作**: `[['right'], ['right', 'A']]` - 极简，只能右移和跳跃
- **7动作**: `SIMPLE_MOVEMENT` - 平衡，包含基本移动
- **12动作**: `COMPLEX_MOVEMENT` - 复杂，包含所有按键组合

### 观测叠帧 (`-o`)
- **1帧**: 单帧图像 (1, 84, 84) - 缺少速度信息
- **4帧**: 叠帧图像 (4, 84, 84) - 包含运动信息

### 示例命令
```bash
# Baseline
python3 -u mario_dqn_main.py -s 0 -v 0 -a 7 -o 1

# 简化配置（推荐入门）
python3 -u mario_dqn_main.py -s 0 -v 1 -a 2 -o 4

# 复杂配置（探索上限）
python3 -u mario_dqn_main.py -s 0 -v 0 -a 12 -o 4
```

---

## 2. 配置文件可调参数

> **注意**: 本次实验主要通过命令行参数和wrapper进行优化，配置文件参数保持默认即可。

### 环境并行数调整（已有）

如需根据机器性能调整，修改 `mario_dqn_config.py`:

```python
cfg.env.collector_env_num = 4  # 并行环境数，根据机器性能调整 [推荐: 2~8]
```

- **性能较弱**: 设为 2-4
- **性能较强**: 设为 4-8

### 其他参数说明

其他训练参数（学习率、batch size、epsilon等）保持默认配置即可，本次实验重点在于：
- 命令行参数组合优化
- Wrapper设计与验证

---

## 3. 可用Wrapper分类

### 📦 A类：观测空间Wrapper（已内置）
| Wrapper | 功能 | 默认使用 | 位置 |
|---------|------|----------|------|
| `MaxAndSkipWrapper` | 跳帧4帧 | ✅ | DI-engine |
| `WarpFrameWrapper` | 图像resize到84x84 | ✅ | DI-engine |
| `ScaledFloatFrameWrapper` | 归一化到[0,1] | ✅ | DI-engine |
| `FrameStackWrapper` | 叠帧(1或4帧) | ✅ | DI-engine |

**使用方式**: 通过命令行 `-v` 和 `-o` 参数调整

---

### 🎮 B类：动作空间Wrapper
| Wrapper | 功能 | 推荐场景 | 参数 |
|---------|------|----------|------|
| `StickyActionWrapper` | 有概率重复上一动作 | 增加环境随机性 | `p_sticky=0.25` |
| `ActionSmoothWrapper` | 动作平滑，减少抖动 | 让动作更连贯 | `repeat_prob=0.5` |

**使用方式**: 
```python
# 在 mario_dqn_main.py 的 wrapped_mario_env 函数中添加
lambda env: StickyActionWrapper(env, p_sticky=0.25),
```

---

### 🎁 C类：奖励塑形Wrapper
| Wrapper | 功能 | 推荐优先级 | 参数 |
|---------|------|------------|------|
| `SparseRewardWrapper` | 稀疏奖励(只有生死) | ⭐⭐ 实验性 | 无 |
| `CoinRewardWrapper` | 增加金币奖励 | ⭐⭐ 可尝试 | 无 |
| `PositionRewardWrapper` | 向右移动奖励 | ⭐⭐⭐ 强烈推荐 | `reward_scale=0.01` |
| `TimePenaltyWrapper` | 原地不动惩罚 | ⭐⭐⭐ 强烈推荐 | `penalty=-0.01` |
| `RewardClipWrapper` | 奖励裁剪 | ⭐⭐ 稳定训练 | `min_r=-1, max_r=1` |

**使用方式**: 
```python
# 在 mario_dqn_main.py 中添加，需要先导入
from wrapper import PositionRewardWrapper, TimePenaltyWrapper

# 在 env_wrapper 列表中添加
lambda env: PositionRewardWrapper(env, reward_scale=0.01),
```

---

### ⚡ D类：训练效率Wrapper
| Wrapper | 功能 | 推荐优先级 | 参数 |
|---------|------|------------|------|
| `NoProgressWrapper` | 无进展提前结束 | ⭐⭐⭐ 强烈推荐 | `max_steps_no_progress=200` |
| `FinalEvalRewardEnv` | 累计奖励统计 | ✅ 已使用 | 无 |

**使用方式**: 
```python
lambda env: NoProgressWrapper(env, max_steps_no_progress=200),
```

---

### 📊 E类：分析可视化Wrapper
| Wrapper | 功能 | 使用场景 | 位置 |
|---------|------|----------|------|
| `RecordCAM` | 记录CAM激活图 | 评估阶段 | evaluate.py |

**使用方式**: 已在 `evaluate.py` 中使用

---

## 4. 推荐实验方案

### 📋 实验总体设计

**目标**: 通过命令行参数调优和Wrapper设计，找到最优训练配置


**分工原则**: 
- **队友A**: 命令行参数探索
- **队友B**: Wrapper设计验证
- **共同**: 最优配置训练 + 消融实验

---

### 🔬 阶段一：并行探索

#### 队友A：命令行参数优化

**任务**: 通过调整 `-v`, `-a`, `-o` 参数找到最优基础配置

**实验计划**:
```bash
# Step 1: Baseline
baseline:    python -u mario_dqn_main.py -s 0 -v 0 -a 7 -o 1

# Step 2: 测试游戏版本
exp_v1:      python -u mario_dqn_main.py -s 0 -v 1 -a 7 -o 1  # 降采样
exp_v2:      python -u mario_dqn_main.py -s 0 -v 2 -a 7 -o 1  # 简化背景
exp_v3:      python -u mario_dqn_main.py -s 0 -v 3 -a 7 -o 1  # 极简化

# Step 3: 测试动作空间
exp_a2:      python -u mario_dqn_main.py -s 0 -v 0 -a 2 -o 1  # 简化动作
exp_a12:     python -u mario_dqn_main.py -s 0 -v 0 -a 12 -o 1 # 复杂动作

# Step 4: 测试叠帧
exp_stack:   python -u mario_dqn_main.py -s 0 -v 0 -a 7 -o 4  # 4帧叠加

# Step 5: 组合测试（基于前面结果）
exp_combo1:  python -u mario_dqn_main.py -s 0 -v 1 -a 2 -o 1
exp_combo2:  python -u mario_dqn_main.py -s 0 -v 1 -a 2 -o 4
exp_combo3:  python -u mario_dqn_main.py -s 0 -v 0 -a 2 -o 4
exp_combo4:  python -u mario_dqn_main.py -s 0 -v 1 -a 7 -o 4
```

**记录内容**:
- 每个配置的最高分数
- 是否通关及通关步数
- 训练过程中的关键观察（tensorboard曲线）
- 推荐的最优配置（1-2个）

---

#### 队友B：Wrapper设计验证

**任务**: 设计和测试不同wrapper的效果

**准备工作**: 修改 `mario_dqn_main.py`，添加wrapper导入
```python
from wrapper import (
    StickyActionWrapper, CoinRewardWrapper, SparseRewardWrapper,
    PositionRewardWrapper, TimePenaltyWrapper, NoProgressWrapper,
    RewardClipWrapper, ActionSmoothWrapper
)
```

**实验计划**:

**基础配置**: 使用 README 推荐的有效配置 `v1_2a_4f`（降采样 + 简化动作 + 叠帧）
> 💡 **说明**: 这个配置已被证明是较优的组合，可以有效测试wrapper的增益效果。
> B不需要等待A的实验结果，可以直接开始。

```bash
# Step 1: Baseline（v1_2a_4f，无额外wrapper）
baseline:    python -u mario_dqn_main.py -s 0 -v 1 -a 2 -o 4

# Step 2: 单个Wrapper测试
# 每次在 wrapped_mario_env 中添加一个wrapper，都使用 v1_2a_4f 基础配置
```

**Wrapper测试列表**:

```python
# Exp 1: 位置奖励（推荐优先）
lambda env: PositionRewardWrapper(env, reward_scale=0.01)

# Exp 2: 无进展终止（推荐优先）
lambda env: NoProgressWrapper(env, max_steps_no_progress=200)

# Exp 3: 时间惩罚
lambda env: TimePenaltyWrapper(env, penalty=-0.01)

# Exp 4: 金币奖励
lambda env: CoinRewardWrapper(env)

# Exp 5: 稀疏奖励
lambda env: SparseRewardWrapper(env)

# Exp 6: 粘性动作
lambda env: StickyActionWrapper(env, p_sticky=0.25)

# Exp 7: 奖励裁剪
lambda env: RewardClipWrapper(env, min_r=-1, max_r=1)
```

**Wrapper组合测试**（基于单个测试结果）:
```python
# 组合1: 鼓励探索
lambda env: PositionRewardWrapper(env, reward_scale=0.01),
lambda env: NoProgressWrapper(env, max_steps_no_progress=200),

# 组合2: 稳定训练
lambda env: TimePenaltyWrapper(env, penalty=-0.01),
lambda env: RewardClipWrapper(env, min_r=-1, max_r=1),

# 组合3: 多目标激励（谨慎）
lambda env: PositionRewardWrapper(env, reward_scale=0.01),
lambda env: CoinRewardWrapper(env),
```

**记录内容**:
- 每个wrapper的效果对比
- 奖励曲线变化
- 是否改善训练效率或最终性能
- 推荐的最优wrapper组合（1-2个）

---

### 🏆 阶段二：深度优化

#### 队友A：最优配置训练

**任务**: 基于阶段一的发现，训练最优配置

**实施步骤**:

1. **确定最优配置**: 整合两人的实验结果
   ```bash
   # 假设最优配置为: v1_2a_4f + PositionReward + NoProgress
   最优参数: -v 1 -a 2 -o 4
   最优wrapper: PositionRewardWrapper + NoProgressWrapper
   ```

2. **多种子训练**: 验证稳定性
   ```bash
   python -u mario_dqn_main.py -s 0 -v 1 -a 2 -o 4  # seed 0
   python -u mario_dqn_main.py -s 1 -v 1 -a 2 -o 4  # seed 1
   python -u mario_dqn_main.py -s 2 -v 1 -a 2 -o 4  # seed 2
   ```

3. **长时间训练**（可选）: 如果3M步未通关
   ```python
   # 在 mario_dqn_main.py 中修改
   max_env_step = int(5e6)  # 延长到5M步
   ```

4. **记录最终性能**:
   - 3个种子的平均分数和方差
   - 通关率和平均通关步数
   - Tensorboard完整曲线
   - 评估视频和CAM可视化

---

#### 队友B：消融实验

**任务**: 证明最优配置中每个组件的价值

**实验设计**: 基于最优配置，每次去掉一个组件

**假设最优配置**: `v1_2a_4f + PositionReward + NoProgress`

```bash
# 完整配置（基准）
full:           python -u mario_dqn_main.py -s 0 -v 1 -a 2 -o 4
                # + PositionReward + NoProgress

# 消融1: 去掉降采样
ablation_v0:    python -u mario_dqn_main.py -s 0 -v 0 -a 2 -o 4
                # + PositionReward + NoProgress

# 消融2: 去掉动作简化
ablation_a7:    python -u mario_dqn_main.py -s 0 -v 1 -a 7 -o 4
                # + PositionReward + NoProgress

# 消融3: 去掉叠帧
ablation_o1:    python -u mario_dqn_main.py -s 0 -v 1 -a 2 -o 1
                # + PositionReward + NoProgress

# 消融4: 去掉PositionReward
ablation_nopos: python -u mario_dqn_main.py -s 0 -v 1 -a 2 -o 4
                # + NoProgress only

# 消融5: 去掉NoProgress
ablation_nonp:  python -u mario_dqn_main.py -s 0 -v 1 -a 2 -o 4
                # + PositionReward only

# 消融6: 去掉所有wrapper
ablation_nowrap: python -u mario_dqn_main.py -s 0 -v 1 -a 2 -o 4
                 # 无额外wrapper
```

**分析要求**:
- 计算每个组件被移除后的性能下降
- 量化每个组件的贡献百分比
- 判断是否存在冗余组件
- 总结组件之间的协同效应

---

## 📊 实验记录模板

### 阶段一：队友A实验记录

| 实验ID | 负责人 | 配置 | 最高分 | 通关 | 通关步数 | 训练时长 | 关键发现 |
|--------|--------|------|--------|------|----------|----------|----------|
| baseline | A | v0_7a_1f | 1500 | ❌ | - | 5h | 基准 |
| exp_v1 | A | v1_7a_1f | 1800 | ❌ | - | 5h | 降采样有效+300 |
| exp_a2 | A | v0_2a_1f | 1600 | ❌ | - | 4h | 动作简化训练快 |
| exp_stack | A | v0_7a_4f | 2200 | ❌ | - | 6h | 叠帧效果显著+700 |
| exp_combo2 | A | v1_2a_4f | 2800 | ✅ | 2.8M | 6h | **最佳组合** |

**队友A结论**: 推荐配置 `v1_2a_4f`

---

### 阶段一：队友B实验记录

| 实验ID | 负责人 | 基础配置 | Wrapper | 最高分 | 通关 | 训练时长 | 效果评价 |
|--------|--------|----------|---------|--------|------|----------|----------|
| baseline | B | v0_7a_4f | None | 2000 | ❌ | 6h | 基准 |
| wrap_pos | B | v0_7a_4f | PositionReward | 2600 | ❌ | 5.5h | 明显加速探索+600 |
| wrap_np | B | v0_7a_4f | NoProgress | 2300 | ❌ | 4.5h | 训练加速,分数略升 |
| wrap_coin | B | v0_7a_4f | CoinReward | 1900 | ❌ | 6h | 效果不佳-100 |
| wrap_combo1 | B | v0_7a_4f | Pos+NP | 2900 | ✅ | 5h | **组合效果好** |

**队友B结论**: 推荐wrapper组合 `PositionReward + NoProgress`

---

### 阶段二：最优配置训练（队友A）

| 种子 | 配置 | Wrapper | 最高分 | 通关步数 | 备注 |
|------|------|---------|--------|----------|------|
| 0 | v1_2a_4f | Pos+NP | 3100 | 2.6M | 通关 |
| 1 | v1_2a_4f | Pos+NP | 2950 | 2.9M | 通关 |
| 2 | v1_2a_4f | Pos+NP | 3200 | 2.5M | 通关 |
| **平均** | - | - | **3083±102** | **2.67M** | **100%通关率** |

---

### 阶段二：消融实验（队友B）

| 实验ID | 配置差异 | 最高分 | 相比完整版 | 贡献度 | 结论 |
|--------|----------|--------|------------|--------|------|
| full | v1_2a_4f+Pos+NP | 3100 | 0 | - | 完整配置基准 |
| ablation_v0 | **v0**_2a_4f+Pos+NP | 2700 | -400 | 12.9% | 降采样很重要 |
| ablation_a7 | v1_**7a**_4f+Pos+NP | 2500 | -600 | 19.4% | 动作简化关键 |
| ablation_o1 | v1_2a_**1f**+Pos+NP | 2400 | -700 | 22.6% | 叠帧贡献最大 |
| ablation_nopos | v1_2a_4f+**NP** | 2600 | -500 | 16.1% | 位置奖励有效 |
| ablation_nonp | v1_2a_4f+**Pos** | 2800 | -300 | 9.7% | NoProgress加速训练 |
| ablation_nowrap | v1_2a_4f | 2550 | -550 | 17.7% | Wrapper组合价值 |

**关键发现**:
- 叠帧贡献最大（22.6%）
- 动作简化次之（19.4%）
- Wrapper组合贡献17.7%
- 所有组件都有正向作用，无冗余

---

## 📈 协作要点

### 代码管理（统一开发分支）
```bash
# 创建并切换到开发分支（仅第一次）
git checkout -b develop
git push -u origin develop

# 日常工作流程
# 1. 每天开始前拉取最新代码
git pull origin develop

# 2. 进行实验，修改代码（添加wrapper等）
# ...训练过程...

# 3. 提交实验相关修改
git add EXPERIMENT_RECORDS.md  # 更新实验记录到专门文档
git add wrapper.py             # 如果新增wrapper
git commit -m "实验记录: A队友-v1_2a_4f配置, 分数2800"

# 4. 推送到远程开发分支
git push origin develop

# 5. 最终实验全部完成后，合并到主分支
git checkout main
git merge develop
git push origin main
```

**注意事项**:
- ⚠️ **避免同时修改同一个文件**: 如都修改 `mario_dqn_main.py` 的同一部分
- ✅ **实验记录分开**: 在 `EXPERIMENT_RECORDS.md` 中各自更新自己的实验表格区域
- ✅ **Wrapper独立添加**: B新增wrapper添加到 `wrapper.py` 末尾，减少冲突
- ✅ **及时提交**: 完成一组实验就提交推送，避免大量改动积压

**如果遇到冲突**:
```bash
# 拉取时遇到冲突
git pull origin develop  # 提示有冲突

# 查看冲突文件
git status

# 手动解决冲突（编辑标记<<<和>>>的部分）
# 然后标记为已解决
git add <冲突文件>
git commit -m "解决合并冲突"
git push origin develop
```

### 实验文档
- **实验记录**: 使用 `EXPERIMENT_RECORDS.md` 专门记录所有实验数据
- **实验指南**: 本文档 (`EXPERIMENT_GUIDE.md`) 仅作为参考，不频繁修改

### 实验记录
- **Tensorboard**: 统一保存到 `exp/` 目录，子目录命名规范
  ```
  exp/
  ├── A_baseline_v0_7a_1f_seed0/
  ├── A_v1_7a_1f_seed0/
  ├── B_baseline_wrapper_none/
  ├── B_pos_wrapper/
  └── final_v1_2a_4f_pos_np_seed0/
  ```


---

## 🔧 快速上手检查清单

- [ ] 环境安装完成（PyTorch, DI-engine, Mario环境）
- [ ] Baseline能正常运行
- [ ] 理解所有命令行参数含义
- [ ] 知道如何添加wrapper
- [ ] 能查看tensorboard结果
- [ ] 确定实验方向和分工
- [ ] 建立实验记录表格
- [ ] 配置Git进行版本控制

---

**祝实验顺利！有问题随时交流 🚀**
