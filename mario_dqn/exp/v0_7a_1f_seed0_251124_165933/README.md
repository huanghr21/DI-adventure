# Mario DQN Training Results

## 训练配置
- **游戏版本**: SuperMarioBros-1-1-v0
- **动作空间**: 7 actions (SIMPLE_MOVEMENT)
- **观测空间**: 1 frame (84x84)
- **环境数量**: 4 collectors, 4 evaluators
- **总训练步数**: 3M envsteps (296,000 iterations)

## 训练结果
- **平均奖励**: 1564.0
- **最高奖励**: 2361.0
- **目标分数**: 3000 (通关)

## 模型文件
- **最佳模型**: `ckpt/ckpt_best.pth.tar`
- **最新模型**: `ckpt/iteration_290000.pth.tar`

## 查看训练曲线
```bash
tensorboard --logdir=./exp/v0_7a_1f_seed0_251124_165933/log/serial
```
然后在浏览器打开 http://localhost:6006

## 超参数
- Learning rate: 0.0001
- Batch size: 32
- Discount factor (gamma): 0.99
- n-step: 3
- Replay buffer size: 100,000
- Update per collect: 10
- Target update frequency: 500
- Epsilon decay: 250,000 steps (1.0 → 0.05)

## 评估
评估在第 296,000 次迭代进行,所有 8 个评估回合均获得 1564 分。
