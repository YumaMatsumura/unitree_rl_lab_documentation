# PPOアルゴリズム設定（rsl_rl_ppo_cfg.py）

**ファイルパス**: `source/unitree_rl_lab/unitree_rl_lab/tasks/{task_type}/agents/rsl_rl_ppo_cfg.py`

PPO（Proximal Policy Optimization）アルゴリズムの学習パラメータを定義します。このファイルでは、学習率、ネットワークアーキテクチャ、ミニバッチ数など、学習の挙動を決定する重要なパラメータを設定します。

## ランナー設定 (BasePPORunnerCfg)

学習の全体的な流れを制御するパラメータです。

```python
num_steps_per_env = 24                  # 各環境でのステップ数（ロールアウト長）
max_iterations = 50000                  # 最大イテレーション数
save_interval = 100                     # モデル保存間隔
experiment_name = ""                    # 実験名（タスク名と同じ）
empirical_normalization = False         # 経験的正規化の使用
```

### パラメータの詳細

#### num_steps_per_env
各環境で何ステップ実行してからポリシーを更新するかを指定します。

- **計算**: 合計サンプル数 = `num_steps_per_env` × `num_envs`
- **例**: 24ステップ × 4096環境 = 98,304サンプル/イテレーション
- **推奨値**: 12-48
- **影響**:
  - 大きい値: より安定した学習だが、更新頻度が低い
  - 小さい値: 高頻度の更新だが、ノイズが多い

#### max_iterations
学習を実行する最大イテレーション数です。

- **推奨値**: 20,000-100,000
- **学習時間の目安**:
  - RTX 4090: 50,000イテレーションで約6-12時間（環境数による）
  - RTX 3060: 同じ設定で約12-24時間

#### save_interval
モデルをチェックポイントとして保存する間隔（イテレーション単位）です。

- **推奨値**: 50-500
- **注意**: 保存間隔が短すぎるとディスクI/Oが学習を遅くする可能性がある

#### empirical_normalization
観測値を経験的に正規化するかどうかを指定します。

- **デフォルト**: `False`
- `True`にすると、実行時の統計に基づいて観測を正規化

## ポリシー・Critic設定 (RslRlPpoActorCriticCfg)

ニューラルネットワークのアーキテクチャを定義します。

```python
policy = RslRlPpoActorCriticCfg(
    init_noise_std=1.0,                 # 初期ノイズの標準偏差
    actor_hidden_dims=[512, 256, 128],  # Actorネットワークの隠れ層次元
    critic_hidden_dims=[512, 256, 128], # Criticネットワークの隠れ層次元
    activation="elu"                     # 活性化関数
)
```

### パラメータの詳細

#### init_noise_std
ポリシーの出力に加えられる初期探索ノイズの標準偏差です。

- **デフォルト**: 1.0
- **推奨値**: 0.5-1.5
- **影響**:
  - 大きい値: より大胆な探索、初期学習が不安定になる可能性
  - 小さい値: 保守的な探索、局所解に陥りやすい

#### actor_hidden_dims
Actorネットワーク（ポリシー）の隠れ層のニューロン数を指定します。

- **デフォルト**: `[512, 256, 128]`
- **他の選択肢**:
  - 軽量版: `[256, 128, 64]` - 高速だが表現力が低い
  - 標準版: `[512, 256, 128]` - バランスが良い
  - 重量版: `[1024, 512, 256]` - 表現力が高いが学習が遅い

#### critic_hidden_dims
Criticネットワーク（価値関数）の隠れ層のニューロン数を指定します。

- **デフォルト**: `[512, 256, 128]`
- 通常はActorと同じ構造にする

#### activation
ネットワークの活性化関数を指定します。

- **デフォルト**: `"elu"`
- **選択肢**:
  - `"elu"`: Exponential Linear Unit（推奨）
  - `"relu"`: Rectified Linear Unit
  - `"tanh"`: 双曲線正接関数
  - `"leaky_relu"`: Leaky ReLU

## PPOアルゴリズム設定 (RslRlPpoAlgorithmCfg)

PPOアルゴリズムの詳細なパラメータを設定します。

```python
algorithm = RslRlPpoAlgorithmCfg(
    value_loss_coef=1.0,                # Value損失の係数
    use_clipped_value_loss=True,        # クリップされたValue損失を使用
    clip_param=0.2,                     # PPOクリッピングパラメータ（ε）
    entropy_coef=0.01,                  # エントロピーボーナスの係数
    num_learning_epochs=5,              # 学習エポック数
    num_mini_batches=4,                 # ミニバッチ数
    learning_rate=1.0e-3,               # 学習率
    schedule="adaptive",                # 学習率スケジュール（"adaptive" or "fixed"）
    gamma=0.99,                         # 割引率
    lam=0.95,                           # GAEのλパラメータ
    desired_kl=0.01,                    # 目標KLダイバージェンス（adaptive使用時）
    max_grad_norm=1.0                   # 勾配クリッピングの最大ノルム
)
```

### パラメータの詳細

#### value_loss_coef
価値関数の損失項の重み係数です。

- **デフォルト**: 1.0
- **推奨値**: 0.5-2.0
- 総損失 = ポリシー損失 + `value_loss_coef` × 価値損失 - `entropy_coef` × エントロピー

#### use_clipped_value_loss
価値関数の損失をクリップするかどうかを指定します。

- **デフォルト**: `True`
- `True`にすると、価値関数の更新も制限され、より安定した学習が可能

#### clip_param
PPOのクリッピングパラメータ（ε）です。

- **デフォルト**: 0.2
- **推奨値**: 0.1-0.3
- **影響**:
  - 大きい値: 大きな更新が可能、学習が速いが不安定
  - 小さい値: 保守的な更新、安定だが学習が遅い

#### entropy_coef
エントロピーボーナスの係数です。探索を促進します。

- **デフォルト**: 0.01
- **推奨値**: 0.001-0.05
- **影響**:
  - 大きい値: より多様な行動、探索的
  - 小さい値: 決定的な行動、搾取的

#### num_learning_epochs
1イテレーションで収集したデータを使って何エポック学習するかを指定します。

- **デフォルト**: 5
- **推奨値**: 3-10
- **影響**:
  - 大きい値: データの活用効率が高いが、過学習のリスク
  - 小さい値: 過学習しにくいが、データの無駄

#### num_mini_batches
収集したデータをいくつのミニバッチに分割するかを指定します。

- **デフォルト**: 4
- **計算**: ミニバッチサイズ = (num_steps_per_env × num_envs) / num_mini_batches
- **例**: (24 × 4096) / 4 = 24,576サンプル/バッチ
- **推奨値**: 2-8
- **影響**:
  - 大きい値: 小さいバッチサイズ、ノイズが多いがメモリ効率が良い
  - 小さい値: 大きいバッチサイズ、安定だがメモリを多く消費

#### learning_rate
オプティマイザの学習率です。

- **デフォルト**: 1.0e-3 (0.001)
- **推奨値**: 1.0e-4 ~ 1.0e-3
- **影響**:
  - 大きい値: 高速な学習だが、不安定になりやすい
  - 小さい値: 安定だが、学習が遅い

#### schedule
学習率のスケジュール方式です。

- **デフォルト**: `"adaptive"`
- **選択肢**:
  - `"adaptive"`: KLダイバージェンスに基づいて学習率を調整
  - `"fixed"`: 学習率を固定

#### gamma
割引率（discount factor）です。

- **デフォルト**: 0.99
- **推奨値**: 0.95-0.999
- **意味**: 将来の報酬をどの程度重視するか
- **影響**:
  - 大きい値: 長期的な報酬を重視
  - 小さい値: 短期的な報酬を重視

#### lam
GAE（Generalized Advantage Estimation）のλパラメータです。

- **デフォルト**: 0.95
- **推奨値**: 0.9-0.98
- **意味**: 分散とバイアスのトレードオフ
- **影響**:
  - 大きい値: 低バイアス、高分散
  - 小さい値: 高バイアス、低分散

#### desired_kl
Adaptive学習率スケジュール使用時の目標KLダイバージェンスです。

- **デフォルト**: 0.01
- **推奨値**: 0.005-0.02
- KLダイバージェンスがこの値を超えると学習率が下がり、下回ると学習率が上がる

#### max_grad_norm
勾配クリッピングの最大ノルムです。

- **デフォルト**: 1.0
- **推奨値**: 0.5-2.0
- 勾配が大きすぎる場合にクリップして学習を安定化

## パラメータ調整の指針

### 学習が不安定な場合

1. `learning_rate` を下げる（例: 5e-4）
2. `clip_param` を小さくする（例: 0.1）
3. `num_mini_batches` を増やす（例: 8）
4. `max_grad_norm` を小さくする（例: 0.5）

### 学習が遅い場合

1. `learning_rate` を上げる（例: 3e-3）
2. `num_learning_epochs` を増やす（例: 8）
3. `num_mini_batches` を減らす（例: 2）
4. `num_steps_per_env` を増やす（例: 48）

### 探索が不足している場合

1. `entropy_coef` を上げる（例: 0.05）
2. `init_noise_std` を上げる（例: 1.5）

### 過学習している場合

1. `num_learning_epochs` を減らす（例: 3）
2. `entropy_coef` を上げる（例: 0.02）
3. より多くのドメインランダマイゼーションを追加

## 推奨設定例

### 高速学習（精度より速度重視）

```python
num_steps_per_env = 16
num_learning_epochs = 3
num_mini_batches = 2
learning_rate = 3.0e-3
```

### 安定学習（速度より精度重視）

```python
num_steps_per_env = 48
num_learning_epochs = 8
num_mini_batches = 8
learning_rate = 5.0e-4
clip_param = 0.1
```

### 実機転用重視

```python
num_steps_per_env = 24
num_learning_epochs = 5
entropy_coef = 0.02
# 環境側で強力なドメインランダマイゼーションを設定
```
