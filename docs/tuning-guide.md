# パラメータチューニングガイド

このページでは、目的別のパラメータ調整方法を紹介します。強化学習の学習効率を向上させたり、より複雑な動作を実現したり、実機転用性を高めるためのテクニックを解説します。

## 学習を安定化させるには

学習曲線が不安定だったり、報酬が発散する場合の対処法です。

### 1. PPOアルゴリズムのパラメータ調整

**learning_rateを下げる**

```python
# rsl_rl_ppo_cfg.py
algorithm = RslRlPpoAlgorithmCfg(
    learning_rate=5.0e-4,  # デフォルト: 1.0e-3
)
```

**clip_paramを小さくする**

```python
algorithm = RslRlPpoAlgorithmCfg(
    clip_param=0.1,  # デフォルト: 0.2
)
```

**max_grad_normを小さくする**

```python
algorithm = RslRlPpoAlgorithmCfg(
    max_grad_norm=0.5,  # デフォルト: 1.0
)
```

### 2. 環境設定のパラメータ調整

**並列環境数を増やす**

```python
# velocity_env_cfg.py
scene: RobotSceneCfg = RobotSceneCfg(
    num_envs=8192,  # デフォルト: 4096（GPUメモリが許す限り）
)
```

**報酬のweightを調整**

タスク報酬とペナルティのバランスを取ります。

```python
# velocity_env_cfg.py
class RewardsCfg:
    # タスク報酬を強化
    track_lin_vel_xy = RewTerm(weight=2.0)  # デフォルト: 1.5
    track_ang_vel_z = RewTerm(weight=1.0)   # デフォルト: 0.75

    # ペナルティを弱める（必要に応じて）
    action_rate = RewTerm(weight=-0.05)     # デフォルト: -0.1
```

### 3. 初期条件のランダム化を緩和

```python
# velocity_env_cfg.py
reset_base = EventTerm(
    params={
        "pose_range": {
            "x": (-0.2, 0.2),    # デフォルト: (-0.5, 0.5)
            "y": (-0.2, 0.2),    # デフォルト: (-0.5, 0.5)
            "yaw": (-1.57, 1.57) # デフォルト: (-3.14, 3.14)
        },
    }
)
```

## 学習を高速化するには

学習時間を短縮したい場合の最適化手法です。

### 1. 効率的なサンプリング

**num_steps_per_envを増やす**

```python
# rsl_rl_ppo_cfg.py
num_steps_per_env = 48  # デフォルト: 24
```

**num_learning_epochsを増やす**

```python
algorithm = RslRlPpoAlgorithmCfg(
    num_learning_epochs=8,  # デフォルト: 5
)
```

**num_mini_batchesを増やす**

```python
algorithm = RslRlPpoAlgorithmCfg(
    num_mini_batches=8,  # デフォルト: 4
)
```

### 2. 制御周波数の調整

**decimationを大きくする**

```python
# velocity_env_cfg.py
def __post_init__(self):
    self.decimation = 8  # デフォルト: 4（制御周波数: 25Hz）
```

制御周波数が下がる（50Hz → 25Hz）ため、タスクによっては性能が低下する可能性があります。

### 3. カリキュラム学習の活用

カリキュラム学習を有効にして、簡単なタスクから徐々に難易度を上げます。

```python
# velocity_env_cfg.py
class CurriculumCfg:
    terrain_levels = CurrTerm(func=mdp.terrain_levels_vel)
    lin_vel_cmd_levels = CurrTerm(mdp.lin_vel_cmd_levels)
```

## より複雑な動作を学習させるには

階段や障害物など、複雑な地形での歩行を実現する方法です。

### 1. 地形の複雑さを増やす

コメントアウトされている地形タイプを有効化します。

```python
# velocity_env_cfg.py
COBBLESTONE_ROAD_CFG = terrain_gen.TerrainGeneratorCfg(
    sub_terrains={
        "flat": terrain_gen.MeshPlaneTerrainCfg(proportion=0.1),
        "random_rough": terrain_gen.HfRandomUniformTerrainCfg(
            proportion=0.2,
            noise_range=(0.01, 0.06),
            noise_step=0.01,
            border_width=0.25
        ),
        "pyramid_stairs": terrain_gen.MeshPyramidStairsTerrainCfg(
            proportion=0.3,
            step_height_range=(0.05, 0.23),
            step_width=0.3,
            platform_width=3.0,
            border_width=1.0,
            holes=False,
        ),
        "boxes": terrain_gen.MeshRandomGridTerrainCfg(
            proportion=0.2,
            grid_width=0.45,
            grid_height_range=(0.05, 0.2),
            platform_width=2.0
        ),
    }
)
```

### 2. エピソード長を延長

```python
# velocity_env_cfg.py
def __post_init__(self):
    self.episode_length_s = 30.0  # デフォルト: 20.0
```

長いエピソードで、より長期的な戦略を学習できます。

### 3. 速度コマンドの範囲を拡大

```python
# velocity_env_cfg.py
base_velocity = mdp.UniformLevelVelocityCommandCfg(
    limit_ranges=mdp.UniformLevelVelocityCommandCfg.Ranges(
        lin_vel_x=(-1.5, 1.5),  # デフォルト: (-1.0, 1.0)
        lin_vel_y=(-0.6, 0.6),  # デフォルト: (-0.4, 0.4)
        ang_vel_z=(-1.5, 1.5)   # デフォルト: (-1.0, 1.0)
    ),
)
```

### 4. 報酬関数の追加

新しい報酬項目を追加して、特定の動作を促進します。

```python
# velocity_env_cfg.py
class RewardsCfg:
    # 既存の報酬...

    # 新しい報酬項目
    stand_still = RewTerm(
        func=mdp.stand_still_penalty,
        weight=-0.5,
        params={"threshold": 0.1}
    )
```

## 実機転用性を高めるには

Sim2Realギャップを小さくし、実機での性能を向上させる方法です。

### 1. ドメインランダマイゼーションの強化

**物理パラメータの範囲を広げる**

```python
# velocity_env_cfg.py
physics_material = EventTerm(
    params={
        "static_friction_range": (0.2, 1.8),   # デフォルト: (0.3, 1.2)
        "dynamic_friction_range": (0.2, 1.5),  # デフォルト: (0.3, 1.2)
        "restitution_range": (0.0, 0.3),       # デフォルト: (0.0, 0.15)
    }
)

add_base_mass = EventTerm(
    params={
        "mass_distribution_params": (-2.0, 5.0),  # デフォルト: (-1.0, 3.0)
    }
)
```

**外乱を強化**

```python
push_robot = EventTerm(
    mode="interval",
    interval_range_s=(3.0, 8.0),  # デフォルト: (5.0, 10.0)
    params={
        "velocity_range": {
            "x": (-1.0, 1.0),     # デフォルト: (-0.5, 0.5)
            "y": (-1.0, 1.0)      # デフォルト: (-0.5, 0.5)
        }
    }
)
```

### 2. 観測ノイズの追加

実機のセンサーノイズを模擬します。

```python
# velocity_env_cfg.py
class ObservationsCfg:
    class PolicyCfg(ObsGroup):
        base_ang_vel = ObsTerm(
            func=mdp.base_ang_vel,
            scale=0.2,
            clip=(-100, 100),
            noise=Unoise(n_min=-0.3, n_max=0.3)  # デフォルト: (-0.2, 0.2)
        )

        joint_pos_rel = ObsTerm(
            func=mdp.joint_pos_rel,
            clip=(-100, 100),
            noise=Unoise(n_min=-0.02, n_max=0.02)  # デフォルト: (-0.01, 0.01)
        )
```

### 3. アクションの滑らかさを重視

**action_rateペナルティの重みを大きくする**

```python
# velocity_env_cfg.py
action_rate = RewTerm(
    func=mdp.action_rate_l2,
    weight=-0.2  # デフォルト: -0.1
)
```

**アクションのスケールを小さくする**

```python
# velocity_env_cfg.py
JointPositionAction = mdp.JointPositionActionCfg(
    scale=0.2,  # デフォルト: 0.25
)
```

### 4. 制御周波数を実機に合わせる

実機の制御周波数（通常50Hz）に合わせます。

```python
# velocity_env_cfg.py
def __post_init__(self):
    self.decimation = 4    # 4 × 0.005s = 0.02s = 50Hz
    self.sim.dt = 0.005
```

### 5. 特権情報の削除

Policyの観測に実機で取得できない情報が含まれていないか確認します。

```python
# velocity_env_cfg.py
class ObservationsCfg:
    class PolicyCfg(ObsGroup):
        # base_lin_velは含めない（実機で直接測定困難）
        base_ang_vel = ObsTerm(...)      # OK: IMUから取得可能
        projected_gravity = ObsTerm(...) # OK: IMUから計算可能
        joint_pos_rel = ObsTerm(...)     # OK: エンコーダから取得可能
        joint_vel_rel = ObsTerm(...)     # OK: エンコーダから計算可能
```

## モーショントラッキング特有の調整

モーショントラッキングタスクで高品質な動作を実現する方法です。

### 1. トラッキング精度の向上

**報酬の重みを調整**

```python
# tracking_env_cfg.py
motion_body_pos = RewTerm(weight=1.5)  # デフォルト: 1.0
motion_body_ori = RewTerm(weight=1.5)  # デフォルト: 1.0
```

**stdパラメータを小さくする**

```python
motion_body_pos = RewTerm(
    params={"std": 0.2}  # デフォルト: 0.3
)
motion_body_ori = RewTerm(
    params={"std": 0.3}  # デフォルト: 0.4
)
```

### 2. 動的なモーションの再現

**速度追従報酬の重みを大きくする**

```python
# tracking_env_cfg.py
motion_body_lin_vel = RewTerm(weight=1.5)  # デフォルト: 1.0
motion_body_ang_vel = RewTerm(weight=1.5)  # デフォルト: 1.0
```

**アクション変化率ペナルティを適度に設定**

```python
action_rate_l2 = RewTerm(weight=-0.05)  # デフォルト: -1e-1
```

動的なモーションでは、アクションが急激に変化する必要があるため、ペナルティを弱めます。

### 3. 初期化範囲の調整

**学習初期は範囲を小さく**

```python
# tracking_env_cfg.py
motion = mdp.MotionCommandCfg(
    pose_range={
        "x": (-0.02, 0.02),   # 小さい範囲から開始
        "y": (-0.02, 0.02),
        "z": (-0.005, 0.005),
        "roll": (-0.05, 0.05),
        "pitch": (-0.05, 0.05),
        "yaw": (-0.1, 0.1)
    },
)
```

**学習が進んだら範囲を拡大**

トラッキング精度が向上したら、ドメインランダマイゼーションを強化します。

## デバッグとモニタリング

### TensorBoardでの学習曲線の確認

```bash
tensorboard --logdir logs/rsl_rl/
```

以下の指標を確認：

- **Mean Reward**: 上昇傾向にあるか
- **Policy Loss**: 安定しているか（発散していないか）
- **Value Loss**: 減少傾向にあるか
- **KL Divergence**: `desired_kl`付近に収束しているか

### ハイパーパラメータの系統的調整

一度に1つのパラメータのみを変更し、効果を確認します。

1. ベースライン設定で学習
2. 1つのパラメータを変更
3. 結果を比較
4. 改善があれば採用、なければ元に戻す
5. 次のパラメータへ

## 推奨設定のテンプレート

### 高速学習（速度重視）

```python
# rsl_rl_ppo_cfg.py
num_steps_per_env = 16
num_learning_epochs = 3
num_mini_batches = 2
learning_rate = 3.0e-3
```

```python
# velocity_env_cfg.py
scene.num_envs = 8192
self.decimation = 8
```

### 安定学習（精度重視）

```python
# rsl_rl_ppo_cfg.py
num_steps_per_env = 48
num_learning_epochs = 8
num_mini_batches = 8
learning_rate = 5.0e-4
clip_param = 0.1
```

```python
# velocity_env_cfg.py
scene.num_envs = 4096
self.decimation = 4
```

### 実機転用重視

```python
# velocity_env_cfg.py
# 強力なドメインランダマイゼーション
physics_material = EventTerm(
    params={
        "static_friction_range": (0.2, 1.8),
        "dynamic_friction_range": (0.2, 1.5),
    }
)

# 観測ノイズの追加
base_ang_vel = ObsTerm(
    noise=Unoise(n_min=-0.3, n_max=0.3)
)

# 滑らかなアクション
action_rate = RewTerm(weight=-0.2)
```

```python
# rsl_rl_ppo_cfg.py
entropy_coef = 0.02  # 探索を促進
```

## まとめ

パラメータ調整は試行錯誤のプロセスです。以下のステップで進めることをお勧めします：

1. **デフォルト設定で学習**を開始
2. **TensorBoardで学習曲線を観察**
3. **問題を特定**（不安定、遅い、性能不足など）
4. **このガイドの該当セクションを参照**してパラメータを調整
5. **結果を評価**し、必要に応じて再調整

実機展開を目指す場合は、早い段階からドメインランダマイゼーションを強化することが重要です。
