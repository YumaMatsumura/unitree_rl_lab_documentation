# ディレクトリ構成

リポジトリの全体構造と各ディレクトリの役割を説明します。

## トップレベル構成

```
unitree_rl_lab/
├─ deploy/                                                 # 学習後の検証/実機化( Sim2Sim → Sim2Real )で使うC++コントローラ群
│  └─ robots/
│     └─ go2/                                              # Go2向けコントローラ
│        ├─ config/
│        │  └─ config.yaml                                 # 状態機械(FSM)用のYAML設定
│        ├─ include/
│        ├─ src/
│        ├─ CMakeLists.txt
│        └─ main.cpp
│
├─ doc/
│  └─ licenses/                                            # ライセンス表記関係
│
├─ docker/                                                 # Docker でIsaac Lab＋依存一式を再現するための設定
│                                                          # 参照: ルート一覧（Docker関連フォルダの存在）
│
├─ scripts/                                                # 学習・推論の実行スクリプト置き場
│  ├─ mimic/
│  │  ├─ csv_to_npz.py                                     # CSVからnpzへの変換スクリプト
│  │  └─ replay_npz.py                                     # モーションデータの再生
│  ├─ rsl_rl/
│  │  ├─ train.py                                          # 学習の実行スクリプト
│  │  ├─ play.py                                           # 学習済みポリシーの再生
│  │  └─ cli_args.py                                       # コマンドライン引数のパーサー
│  └─ list_envs.py
│                                                          # 使い方例:
│                                                          #   python scripts/rsl_rl/train.py --headless --task Unitree-Go2-Velocity
│                                                          #   python scripts/rsl_rl/play.py  --task Unitree-Go2-Velocity
│                                                          #   （同等のショートカットは unitree_rl_lab.sh 経由でも可）
│                                                          # 参照: README の Installation/Verify/Running の例コマンド
│
├─ source/
│  └─ unitree_rl_lab/
│     ├─ config/
│     │  └─ extension.toml
│     ├─ docs/
│     └─ unitree_rl_lab/
│        ├─ assets/
│        │  └─ robots/
│        │     ├─ unitree.py                               # モデル参照パスの設定スクリプト
│        │     │                                           #   ・USD を使う場合: UNITREE_MODEL_DIR を設定（Hugging Faceのunitree_modelを取得）
│        │     │                                           #   ・URDFを使う場合: UNITREE_ROS_DIR を設定（unitree_rosを取得; IsaacSim 5.0+ 推奨）
│        │     │                                           #   ・必要に応じて spawn 設定をURDFモードに切替
│        │     │                                           #   参照: README の「Download unitree robot description files」節
│        │     └─ unitree_actuators.py                     # アクチュエーターモデルの定義
│        ├─ tasks/
│        │  ├─ locomotion/                                 # ロコモーションタスク（速度追従など）
│        │  │  ├─ agents/
│        │  │  │  └─ rsl_rl_ppo_cfg.py                     # PPOアルゴリズムの設定
│        │  │  ├─ mdp/
│        │  │  │  ├─ commands/                             # コマンド生成関数
│        │  │  │  ├─ rewards.py                            # 報酬関数の定義
│        │  │  │  ├─ observations.py                       # 観測関数の定義
│        │  │  │  └─ curriculums.py                        # カリキュラム学習の定義
│        │  │  └─ robots/
│        │  │     ├─ go2/
│        │  │     │  └─ velocity_env_cfg.py                # Go2の速度追従環境設定
│        │  │     ├─ h1/
│        │  │     │  └─ velocity_env_cfg.py                # H1の速度追従環境設定
│        │  │     └─ g1/
│        │  │        └─ 29dof/
│        │  │           └─ velocity_env_cfg.py             # G1 29DOFの速度追従環境設定
│        │  └─ mimic/                                      # モーショントラッキングタスク
│        │     ├─ agents/
│        │     │  └─ rsl_rl_ppo_cfg.py                     # PPOアルゴリズムの設定
│        │     ├─ mdp/
│        │     │  ├─ commands.py                           # モーションコマンド生成
│        │     │  ├─ rewards.py                            # 報酬関数の定義
│        │     │  ├─ observations.py                       # 観測関数の定義
│        │     │  ├─ terminations.py                       # 終了条件の定義
│        │     │  └─ events.py                             # イベントハンドラ
│        │     └─ robots/
│        │        └─ g1_29dof/
│        │           ├─ gangnanm_style/
│        │           │  ├─ tracking_env_cfg.py             # 江南スタイルのトラッキング環境設定
│        │           │  └─ G1_gangnam_style_V01.bvh_60hz.npz  # モーションデータ
│        │           └─ dance_102/
│        │              ├─ tracking_env_cfg.py             # ダンス102のトラッキング環境設定
│        │              └─ G1_dance_102.bvh_60hz.npz      # モーションデータ
│        └─ utils/
│           ├─ parser_cfg.py                               # 設定ファイルのパーサー
│           └─ export_deploy_cfg.py                        # デプロイ設定のエクスポート
│
└─ unitree_rl_lab.sh                                       # セットアップ＆実行ショートカット
                                                           #   -i : ライブラリを editable でインストール（Isaac Lab環境のPythonで）
                                                           #   -l : タスク一覧表示（IsaacLab本体より高速版のリスト）
                                                           #   -t : 学習実行（train.py相当）
                                                           #   -p : 再生実行（play.py相当）
                                                           #   参照: README の Installation/Verify の手順
```

## 主要ディレクトリの詳細

### deploy/
学習後のポリシーを実機やSim2Simで動作させるためのC++コントローラが含まれています。各ロボット向けにディレクトリが分かれており、FSM（有限状態機械）による状態管理を行います。

### scripts/
学習と推論を実行するためのPythonスクリプトが格納されています。

- `rsl_rl/`: RSL-RLライブラリを使用した強化学習スクリプト
  - `train.py`: 学習実行
  - `play.py`: 学習済みポリシーの再生
- `mimic/`: モーションデータ関連のユーティリティ

### source/unitree_rl_lab/unitree_rl_lab/

#### assets/robots/
ロボットモデルの設定とアクチュエーターモデルの定義が含まれています。

#### tasks/
タスクごとに環境設定が定義されています。

**locomotion/**: 速度追従タスク

- `agents/`: PPOアルゴリズムの設定
- `mdp/`: MDP（マルコフ決定過程）の定義
  - `commands/`: コマンド生成
  - `rewards.py`: 報酬関数
  - `observations.py`: 観測関数
  - `curriculums.py`: カリキュラム学習
- `robots/`: ロボットごとの環境設定ファイル

**mimic/**: モーショントラッキングタスク

- `agents/`: PPOアルゴリズムの設定
- `mdp/`: MDPの定義
  - `commands.py`: モーションコマンド生成
  - `rewards.py`: トラッキング報酬関数
  - `observations.py`: 観測関数
  - `terminations.py`: 終了条件
  - `events.py`: イベント処理
- `robots/`: ロボットとモーションごとの設定ファイル

#### utils/
設定ファイルのパース処理やデプロイ設定のエクスポート機能など、ユーティリティ関数が含まれています。

## ファイルの役割

### 設定ファイル

| ファイル | 説明 |
|---------|------|
| `velocity_env_cfg.py` | 速度追従タスクの環境設定（地形、報酬、観測、イベントなど） |
| `tracking_env_cfg.py` | モーショントラッキングタスクの環境設定 |
| `rsl_rl_ppo_cfg.py` | PPOアルゴリズムの学習パラメータ |
| `config.yaml` | デプロイ時のFSM設定 |

### スクリプトファイル

| ファイル | 説明 |
|---------|------|
| `train.py` | 学習を実行するメインスクリプト |
| `play.py` | 学習済みポリシーを再生するスクリプト |
| `csv_to_npz.py` | CSVモーションデータをnpz形式に変換 |
| `replay_npz.py` | npzモーションデータを再生・可視化 |
| `unitree_rl_lab.sh` | セットアップと実行のショートカットスクリプト |
