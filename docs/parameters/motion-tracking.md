# モーショントラッキング設定（tracking_env_cfg.py）

**ファイルパス**: `source/unitree_rl_lab/unitree_rl_lab/tasks/mimic/robots/g1_29dof/*/tracking_env_cfg.py`

モーションキャプチャデータを追従するタスクの設定です。BVH形式のモーションデータをnpz形式に変換したファイルを読み込み、ロボットに再現させることができます。

## モーションコマンド設定 (MotionCommandCfg)

トラッキングするモーションデータと、ランダム化の範囲を設定します。

```python
motion = mdp.MotionCommandCfg(
    asset_name="robot",
    motion_file="path/to/motion.npz",                    # モーションデータファイル
    anchor_body_name="torso_link",                       # アンカーとなるボディ
    resampling_time_range=(1.0e9, 1.0e9),               # リサンプリング時間（実質無限大）
    debug_vis=True,                                      # デバッグビジュアライゼーション
    pose_range={                                         # 姿勢のランダム化範囲
        "x": (-0.05, 0.05),
        "y": (-0.05, 0.05),
        "z": (-0.01, 0.01),
        "roll": (-0.1, 0.1),
        "pitch": (-0.1, 0.1),
        "yaw": (-0.2, 0.2)
    },
    velocity_range={                                     # 速度のランダム化範囲
        "x": (-0.5, 0.5),
        "y": (-0.5, 0.5),
        "z": (-0.2, 0.2),
        "roll": (-0.52, 0.52),
        "pitch": (-0.52, 0.52),
        "yaw": (-0.78, 0.78)
    },
    joint_position_range=(-0.1, 0.1),                    # 関節位置のランダム化範囲
    body_names=[                                         # トラッキング対象のボディリスト
        "pelvis",
        "left_hip_roll_link",
        "left_knee_link",
        "left_ankle_roll_link",
        "right_hip_roll_link",
        "right_knee_link",
        "right_ankle_roll_link",
        "torso_link",
        "left_shoulder_roll_link",
        "left_elbow_link",
        "left_wrist_yaw_link",
        "right_shoulder_roll_link",
        "right_elbow_link",
        "right_wrist_yaw_link",
    ]
)
```

### パラメータの詳細

#### motion_file
npz形式のモーションデータファイルのパスです。

- BVHファイルは`scripts/mimic/csv_to_npz.py`を使ってnpz形式に変換する必要があります
- ファイルには以下の情報が含まれます：
  - 各ボディの位置・姿勢の時系列データ
  - 各関節の角度の時系列データ
  - 線速度・角速度の時系列データ

#### anchor_body_name
モーションのアンカー（基準）となるボディ名です。

- 通常は胴体（`"torso_link"`）を指定
- このボディの位置・姿勢がモーションデータと一致するように学習

#### resampling_time_range
モーションをリサンプリングする時間範囲です。

- デフォルトでは`(1.0e9, 1.0e9)`（実質無限大）に設定し、モーション全体を1エピソードで実行
- 短く設定すると、モーションの一部をランダムに切り出して学習

#### debug_vis
デバッグ用の可視化を有効にするかどうかです。

- `True`: モーションの目標位置が球体として表示される
- `False`: 可視化なし（学習時は無効化推奨）

#### pose_range
リセット時に初期姿勢に加えるランダムなオフセットの範囲です。

- ロバスト性を向上させるためのドメインランダマイゼーション
- 値が大きすぎると、初期姿勢が不自然になり学習が困難になる

#### velocity_range
リセット時に初期速度に加えるランダムなオフセットの範囲です。

- 動的なモーション（ダンスなど）では適度なランダム化が有効
- 静的なモーションでは小さく設定

#### joint_position_range
リセット時に各関節位置に加えるランダムなオフセットの範囲 [rad] です。

- 推奨値: -0.1 ~ 0.1 [rad]
- 大きすぎると不自然な姿勢から開始する可能性

#### body_names
トラッキングの対象とするボディのリストです。

- リストに含まれるボディの位置・姿勢がモーションデータと一致するように報酬が与えられる
- 重要なボディ（足首、手首など）を優先的に追加

## モーショントラッキング報酬 (RewardsCfg)

モーションを正確に再現するための報酬関数です。

### アンカーボディの追従報酬

```python
motion_global_anchor_pos = RewTerm(
    func=mdp.motion_global_anchor_position_error_exp,
    weight=0.5,
    params={"command_name": "motion", "std": 0.3}
)

motion_global_anchor_ori = RewTerm(
    func=mdp.motion_global_anchor_orientation_error_exp,
    weight=0.5,
    params={"command_name": "motion", "std": 0.4}
)
```

**説明**:

- アンカーボディのグローバル位置・姿勢をモーションデータに一致させる
- `std`パラメータ: 小さいほど厳密な一致を要求

### ボディの相対位置・姿勢の追従報酬

```python
motion_body_pos = RewTerm(
    func=mdp.motion_relative_body_position_error_exp,
    weight=1.0,
    params={"command_name": "motion", "std": 0.3}
)

motion_body_ori = RewTerm(
    func=mdp.motion_relative_body_orientation_error_exp,
    weight=1.0,
    params={"command_name": "motion", "std": 0.4}
)
```

**説明**:

- アンカーボディに対する相対的な位置・姿勢を一致させる
- 手足の位置関係を正確に再現するために重要

### ボディの速度追従報酬

```python
motion_body_lin_vel = RewTerm(
    func=mdp.motion_global_body_linear_velocity_error_exp,
    weight=1.0,
    params={"command_name": "motion", "std": 1.0}
)

motion_body_ang_vel = RewTerm(
    func=mdp.motion_global_body_angular_velocity_error_exp,
    weight=1.0,
    params={"command_name": "motion", "std": 3.14}
)
```

**説明**:

- ボディの線速度・角速度をモーションデータに一致させる
- 動的なモーションでは重要度が高い

### 正則化項目

```python
joint_acc = RewTerm(func=mdp.joint_acc_l2, weight=-2.5e-7)      # 関節加速度ペナルティ
joint_torque = RewTerm(func=mdp.joint_torques_l2, weight=-1e-5) # 関節トルクペナルティ
action_rate_l2 = RewTerm(func=mdp.action_rate_l2, weight=-1e-1) # アクション変化率ペナルティ
joint_limit = RewTerm(
    func=mdp.joint_pos_limits,
    weight=-10.0,
    params={"asset_cfg": SceneEntityCfg("robot", joint_names=[".*"])}
)
```

### 望ましくない接触のペナルティ

```python
undesired_contacts = RewTerm(
    func=mdp.undesired_contacts,
    weight=-0.1,
    params={
        "sensor_cfg": SceneEntityCfg(
            "contact_forces",
            body_names=[
                r"^(?!left_ankle_roll_link$)(?!right_ankle_roll_link$)(?!left_wrist_yaw_link$)(?!right_wrist_yaw_link$).+$"
            ]
        ),
        "threshold": 1.0
    }
)
```

**説明**:

- 正規表現で、足首と手首以外のボディの接触をペナルティ
- ダンスなどでは手をつくこともあるため、重みを小さく設定

## モーショントラッキング終了条件 (TerminationsCfg)

トラッキングが失敗した場合の終了条件を設定します。

```python
time_out = DoneTerm(func=mdp.time_out, time_out=True)

anchor_pos = DoneTerm(
    func=mdp.bad_anchor_pos_z_only,
    params={"command_name": "motion", "threshold": 0.25}
)

anchor_ori = DoneTerm(
    func=mdp.bad_anchor_ori,
    params={
        "asset_cfg": SceneEntityCfg("robot"),
        "command_name": "motion",
        "threshold": 0.8
    }
)

ee_body_pos = DoneTerm(
    func=mdp.bad_motion_body_pos_z_only,
    params={
        "command_name": "motion",
        "threshold": 0.25,
        "body_names": [
            "left_ankle_roll_link",
            "right_ankle_roll_link",
            "left_wrist_yaw_link",
            "right_wrist_yaw_link"
        ]
    }
)
```

### パラメータの詳細

#### anchor_pos
アンカーボディの高さがモーションデータから一定以上離れたらエピソードを終了します。

- `threshold`: 許容誤差 [m]（デフォルト: 0.25m）

#### anchor_ori
アンカーボディの姿勢が一定以上傾いたらエピソードを終了します。

- `threshold`: 許容角度 [rad]（デフォルト: 0.8 rad ≈ 45度）

#### ee_body_pos
エンドエフェクタ（手首・足首）の高さがモーションデータから大きく離れたら終了します。

- 重要なボディの位置を厳密にチェック

## イベント設定 (EventCfg)

ドメインランダマイゼーションを設定します。

### 起動時のランダム化

```python
physics_material = EventTerm(
    func=mdp.randomize_rigid_body_material,
    mode="startup",
    params={
        "asset_cfg": SceneEntityCfg("robot", body_names=".*"),
        "static_friction_range": (0.3, 1.6),
        "dynamic_friction_range": (0.3, 1.2),
        "restitution_range": (0.0, 0.5),
        "num_buckets": 64
    }
)

add_joint_default_pos = EventTerm(
    func=mdp.randomize_joint_default_pos,
    mode="startup",
    params={
        "asset_cfg": SceneEntityCfg("robot", joint_names=[".*"]),
        "pos_distribution_params": (-0.01, 0.01),
        "operation": "add"
    }
)

base_com = EventTerm(
    func=mdp.randomize_rigid_body_com,
    mode="startup",
    params={
        "asset_cfg": SceneEntityCfg("robot", body_names="torso_link"),
        "com_range": {"x": (-0.025, 0.025), "y": (-0.05, 0.05), "z": (-0.05, 0.05)}
    }
)
```

### 定期的な外乱

```python
push_robot = EventTerm(
    func=mdp.push_by_setting_velocity,
    mode="interval",
    interval_range_s=(1.0, 3.0),
    params={"velocity_range": VELOCITY_RANGE}
)
```

## 全体設定

```python
decimation = 4                          # 制御周期: 50Hz（4 × 0.005s = 0.02s）
episode_length_s = 30.0                 # エピソード長（モーションの長さに合わせて調整）
sim.dt = 0.005                          # シミュレーションのタイムステップ
```

## モーションデータの準備

### BVHファイルからnpzへの変換

```bash
python scripts/mimic/csv_to_npz.py \
    --bvh_file path/to/motion.bvh \
    --output_file path/to/motion.npz \
    --fps 60
```

### モーションデータの再生・確認

```bash
python scripts/mimic/replay_npz.py \
    --motion_file path/to/motion.npz \
    --robot g1_29dof
```

## パラメータ調整のコツ

### トラッキング精度を上げるには

1. 報酬の`weight`を調整して、位置・姿勢・速度のバランスを取る
2. `std`パラメータを小さくして、より厳密な一致を要求
3. `body_names`に重要なボディを追加

### 学習を安定化させるには

1. 初期のランダム化範囲（`pose_range`、`velocity_range`）を小さくする
2. 終了条件の`threshold`を緩くする
3. ドメインランダマイゼーションを弱める

### 動的なモーションを再現するには

1. 速度追従報酬（`motion_body_lin_vel`、`motion_body_ang_vel`）の重みを大きくする
2. `episode_length_s`を長くして、モーション全体を学習
3. アクション変化率ペナルティを適度に設定
