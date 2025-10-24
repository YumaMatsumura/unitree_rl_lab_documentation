# 環境設定パラメータ（velocity_env_cfg.py）

**ファイルパス**: `source/unitree_rl_lab/unitree_rl_lab/tasks/locomotion/robots/{robot_name}/velocity_env_cfg.py`

ロボットの速度追従タスクの環境設定を定義します。このファイルでは、シミュレーション環境、地形、報酬関数、観測、イベントなど、強化学習に必要な全ての要素を設定します。

## シーン設定 (RobotSceneCfg)

並列シミュレーション環境の数と配置を設定します。

```python
scene: RobotSceneCfg = RobotSceneCfg(
    num_envs=4096,      # 並列環境数（GPUメモリに応じて調整）
    env_spacing=2.5      # 環境間の間隔 [m]
)
```

**調整のポイント**:

- `num_envs`: GPUメモリに応じて調整。多いほど学習が高速化されるが、メモリ不足に注意。
  - 2の累乗でないと、「FPSが10〜30%程度低下」「端数処理が入りwarp効率が落ちる」といった問題が起こるため、2の累乗で与えるのが良い。
  - `num_envs`の値が小さいほど速いわけではなく、総合的には4096が最速になることが多い。
  - ただし、「GPUメモリが足りない」「学習が重い」などの場合は2の累乗で値を小さくするのが良い。
  - RTX 3060 (12GB): 2048-4096
  - RTX 4090 (24GB): 8192-16384

## 地形設定 (COBBLESTONE_ROAD_CFG)

トレーニング用の地形を生成します。

```python
COBBLESTONE_ROAD_CFG = terrain_gen.TerrainGeneratorCfg(
    size=(8.0, 8.0),                    # 地形のサイズ [m]
    num_rows=10,                        # 地形の行数
    num_cols=20,                        # 地形の列数
    horizontal_scale=0.1,               # 水平方向のスケール
    vertical_scale=0.005,               # 垂直方向のスケール（地形の起伏）
    difficulty_range=(0.0, 1.0),        # 難易度の範囲
    sub_terrains={                      # サブ地形の定義
        "flat": terrain_gen.MeshPlaneTerrainCfg(proportion=0.1),
        # コメントアウトされた地形を有効化することで、階段、ボックス、傾斜などを追加可能
    }
)
```

**サブ地形のオプション**:

```python
# ランダムな凹凸
"random_rough": terrain_gen.HfRandomUniformTerrainCfg(
    proportion=0.1,
    noise_range=(0.01, 0.06),
    noise_step=0.01,
    border_width=0.25
),

# ピラミッド型の傾斜
"hf_pyramid_slope": terrain_gen.HfPyramidSlopedTerrainCfg(
    proportion=0.1,
    slope_range=(0.0, 0.4),
    platform_width=2.0,
    border_width=0.25
),

# ランダムボックス
"boxes": terrain_gen.MeshRandomGridTerrainCfg(
    proportion=0.2,
    grid_width=0.45,
    grid_height_range=(0.05, 0.2),
    platform_width=2.0
),

# 階段
"pyramid_stairs": terrain_gen.MeshPyramidStairsTerrainCfg(
    proportion=0.2,
    step_height_range=(0.05, 0.23),
    step_width=0.3,
    platform_width=3.0,
    border_width=1.0,
    holes=False,
),
```

## 物理マテリアル設定

地形との接触時の物理特性を設定します。

```python
physics_material=sim_utils.RigidBodyMaterialCfg(
    friction_combine_mode="multiply",   # 摩擦の結合モード
    restitution_combine_mode="multiply", # 反発係数の結合モード
    static_friction=1.0,                # 静止摩擦係数
    dynamic_friction=1.0,               # 動摩擦係数
)
```

## コマンド設定 (CommandsCfg)

ロボットに与える速度コマンドの範囲を設定します。

```python
base_velocity = mdp.UniformLevelVelocityCommandCfg(
    resampling_time_range=(10.0, 10.0),  # コマンドの再サンプリング時間 [s]
    rel_standing_envs=0.1,                # 静止環境の割合
    ranges=mdp.UniformLevelVelocityCommandCfg.Ranges(
        lin_vel_x=(-0.1, 0.1),            # 初期の前後方向の速度範囲 [m/s]
        lin_vel_y=(-0.1, 0.1),            # 初期の左右方向の速度範囲 [m/s]
        ang_vel_z=(-1, 1)                 # 初期の回転速度範囲 [rad/s]
    ),
    limit_ranges=mdp.UniformLevelVelocityCommandCfg.Ranges(
        lin_vel_x=(-1.0, 1.0),            # 最終的な前後方向の速度範囲
        lin_vel_y=(-0.4, 0.4),            # 最終的な左右方向の速度範囲
        ang_vel_z=(-1.0, 1.0)             # 最終的な回転速度範囲
    ),
)
```

**パラメータの意味**:

- `ranges`: カリキュラム学習の初期段階での速度範囲
- `limit_ranges`: カリキュラム学習の最終段階での速度範囲
- `rel_standing_envs`: 静止コマンド（速度0）を与える環境の割合

## アクション設定 (ActionsCfg)

ポリシーネットワークの出力をロボットの制御に変換する設定です。

```python
JointPositionAction = mdp.JointPositionActionCfg(
    asset_name="robot",
    joint_names=[".*"],                  # 対象となる関節（正規表現）
    scale=0.25,                          # アクションのスケール
    use_default_offset=True,             # デフォルトオフセットを使用
    clip={".*": (-100.0, 100.0)}         # アクションのクリッピング範囲
)
```

**調整のポイント**:

- `scale`: 値が大きいほど、ポリシーの出力が大きな関節角度変化を引き起こす

## 観測設定 (ObservationsCfg)

### PolicyCfg - ポリシーネットワーク用の観測

実機でも取得可能な観測値のみを使用します。

```python
base_ang_vel = ObsTerm(
    func=mdp.base_ang_vel,
    scale=0.2,                           # スケール係数
    clip=(-100, 100),                    # クリッピング範囲
    noise=Unoise(n_min=-0.2, n_max=0.2)  # ノイズ範囲
)
projected_gravity = ObsTerm(
    func=mdp.projected_gravity,
    clip=(-100, 100),
    noise=Unoise(n_min=-0.05, n_max=0.05)
)
velocity_commands = ObsTerm(
    func=mdp.generated_commands,
    clip=(-100, 100),
    params={"command_name": "base_velocity"}
)
joint_pos_rel = ObsTerm(
    func=mdp.joint_pos_rel,
    clip=(-100, 100),
    noise=Unoise(n_min=-0.01, n_max=0.01)
)
joint_vel_rel = ObsTerm(
    func=mdp.joint_vel_rel,
    scale=0.05,
    clip=(-100, 100),
    noise=Unoise(n_min=-1.5, n_max=1.5)
)
last_action = ObsTerm(
    func=mdp.last_action,
    clip=(-100, 100)
)
```

**観測項目の説明**:

- `base_ang_vel`: 胴体の角速度（IMUから取得）
- `projected_gravity`: 投影された重力ベクトル（IMUから計算）
- `velocity_commands`: 速度コマンド
- `joint_pos_rel`: 関節位置（デフォルト位置からの相対値）
- `joint_vel_rel`: 関節速度
- `last_action`: 前ステップのアクション

### CriticCfg - Criticネットワーク用の観測（特権情報）

シミュレーションでのみ利用可能な情報を含みます。

```python
base_lin_vel = ObsTerm(func=mdp.base_lin_vel, clip=(-100, 100))
base_ang_vel = ObsTerm(func=mdp.base_ang_vel, scale=0.2, clip=(-100, 100))
projected_gravity = ObsTerm(func=mdp.projected_gravity, clip=(-100, 100))
velocity_commands = ObsTerm(
    func=mdp.generated_commands,
    clip=(-100, 100),
    params={"command_name": "base_velocity"}
)
joint_pos_rel = ObsTerm(func=mdp.joint_pos_rel, clip=(-100, 100))
joint_vel_rel = ObsTerm(func=mdp.joint_vel_rel, scale=0.05, clip=(-100, 100))
joint_effort = ObsTerm(func=mdp.joint_effort, scale=0.01, clip=(-100, 100))
last_action = ObsTerm(func=mdp.last_action, clip=(-100, 100))
```

**追加の観測**:

- `base_lin_vel`: 胴体の並進速度（実機では直接測定困難）
- `joint_effort`: 関節トルク

## 報酬設定 (RewardsCfg)

報酬関数は学習の成否を決定する最も重要な要素です。

### タスク報酬

```python
track_lin_vel_xy = RewTerm(
    func=mdp.track_lin_vel_xy_exp,
    weight=1.5,                          # 重み係数
    params={"command_name": "base_velocity", "std": math.sqrt(0.25)}
)
track_ang_vel_z = RewTerm(
    func=mdp.track_ang_vel_z_exp,
    weight=0.75,
    params={"command_name": "base_velocity", "std": math.sqrt(0.25)}
)
```

### ペナルティ項目

```python
base_linear_velocity = RewTerm(
    func=mdp.lin_vel_z_l2,
    weight=-2.0
)  # Z軸方向の並進速度ペナルティ

base_angular_velocity = RewTerm(
    func=mdp.ang_vel_xy_l2,
    weight=-0.05
)  # XY軸周りの角速度ペナルティ

joint_vel = RewTerm(
    func=mdp.joint_vel_l2,
    weight=-0.001
)  # 関節速度ペナルティ

joint_acc = RewTerm(
    func=mdp.joint_acc_l2,
    weight=-2.5e-7
)  # 関節加速度ペナルティ

joint_torques = RewTerm(
    func=mdp.joint_torques_l2,
    weight=-2e-4
)  # 関節トルクペナルティ

action_rate = RewTerm(
    func=mdp.action_rate_l2,
    weight=-0.1
)  # アクション変化率ペナルティ

dof_pos_limits = RewTerm(
    func=mdp.joint_pos_limits,
    weight=-10.0
)  # 関節角度制限ペナルティ

energy = RewTerm(
    func=mdp.energy,
    weight=-2e-5
)  # エネルギー消費ペナルティ

flat_orientation_l2 = RewTerm(
    func=mdp.flat_orientation_l2,
    weight=-2.5
)  # 姿勢維持ペナルティ
```

### 足の接触報酬

```python
feet_air_time = RewTerm(
    func=mdp.feet_air_time,
    weight=0.1,
    params={
        "sensor_cfg": SceneEntityCfg("contact_forces", body_names=".*_foot"),
        "command_name": "base_velocity",
        "threshold": 0.5
    }
)

air_time_variance = RewTerm(
    func=mdp.air_time_variance_penalty,
    weight=-1.0,
    params={"sensor_cfg": SceneEntityCfg("contact_forces", body_names=".*_foot")}
)

feet_slide = RewTerm(
    func=mdp.feet_slide,
    weight=-0.1,
    params={
        "asset_cfg": SceneEntityCfg("robot", body_names=".*_foot"),
        "sensor_cfg": SceneEntityCfg("contact_forces", body_names=".*_foot")
    }
)
```

### 望ましくない接触のペナルティ

```python
undesired_contacts = RewTerm(
    func=mdp.undesired_contacts,
    weight=-1,
    params={
        "threshold": 1,
        "sensor_cfg": SceneEntityCfg(
            "contact_forces",
            body_names=["Head_.*", ".*_hip", ".*_thigh", ".*_calf"]
        )
    }
)
```

## イベント設定 (EventCfg)

ドメインランダマイゼーションと環境のリセット処理を設定します。

### 起動時のランダム化

```python
physics_material = EventTerm(
    func=mdp.randomize_rigid_body_material,
    mode="startup",
    params={
        "asset_cfg": SceneEntityCfg("robot", body_names=".*"),
        "static_friction_range": (0.3, 1.2),    # 静止摩擦係数の範囲
        "dynamic_friction_range": (0.3, 1.2),   # 動摩擦係数の範囲
        "restitution_range": (0.0, 0.15),       # 反発係数の範囲
        "num_buckets": 64                        # バケット数
    }
)

add_base_mass = EventTerm(
    func=mdp.randomize_rigid_body_mass,
    mode="startup",
    params={
        "asset_cfg": SceneEntityCfg("robot", body_names="base"),
        "mass_distribution_params": (-1.0, 3.0),  # 質量の変動範囲 [kg]
        "operation": "add"
    }
)
```

### リセット時の設定

```python
reset_base = EventTerm(
    func=mdp.reset_root_state_uniform,
    mode="reset",
    params={
        "pose_range": {
            "x": (-0.5, 0.5),
            "y": (-0.5, 0.5),
            "yaw": (-3.14, 3.14)
        },
        "velocity_range": {
            "x": (0.0, 0.0),
            "y": (0.0, 0.0),
            "z": (0.0, 0.0),
            "roll": (0.0, 0.0),
            "pitch": (0.0, 0.0),
            "yaw": (0.0, 0.0)
        }
    }
)

reset_robot_joints = EventTerm(
    func=mdp.reset_joints_by_scale,
    mode="reset",
    params={
        "position_range": (1.0, 1.0),            # 関節位置の範囲
        "velocity_range": (-1.0, 1.0)            # 関節速度の範囲
    }
)
```

### 定期的なイベント

```python
push_robot = EventTerm(
    func=mdp.push_by_setting_velocity,
    mode="interval",
    interval_range_s=(5.0, 10.0),                # イベント発生間隔 [s]
    params={
        "velocity_range": {
            "x": (-0.5, 0.5),
            "y": (-0.5, 0.5)
        }
    }
)
```

## 終了条件 (TerminationsCfg)

エピソードを終了させる条件を設定します。

```python
time_out = DoneTerm(func=mdp.time_out, time_out=True)

base_contact = DoneTerm(
    func=mdp.illegal_contact,
    params={
        "sensor_cfg": SceneEntityCfg("contact_forces", body_names="base"),
        "threshold": 1.0
    }
)

bad_orientation = DoneTerm(
    func=mdp.bad_orientation,
    params={"limit_angle": 0.8}          # 姿勢の許容角度 [rad]
)
```

## カリキュラム学習 (CurriculumCfg)

学習の進行に応じて難易度を調整します。

```python
terrain_levels = CurrTerm(func=mdp.terrain_levels_vel)      # 地形の難易度を段階的に上昇
lin_vel_cmd_levels = CurrTerm(mdp.lin_vel_cmd_levels)       # 速度コマンドの範囲を段階的に拡大
```

## 全体設定 (RobotEnvCfg)

```python
decimation = 4                          # 制御周期のデシメーション（4 × 0.005s = 0.02s = 50Hz）
episode_length_s = 20.0                 # エピソード長 [s]
sim.dt = 0.005                          # シミュレーションのタイムステップ [s]
sim.render_interval = decimation        # レンダリング間隔
```

**パラメータの説明**:

- `decimation`: シミュレーションステップごとに何回に1回制御を実行するか
- `episode_length_s`: 1エピソードの長さ（秒）
- `sim.dt`: 物理シミュレーションのタイムステップ
- 制御周波数 = 1 / (decimation × sim.dt) = 1 / (4 × 0.005) = 50 Hz
