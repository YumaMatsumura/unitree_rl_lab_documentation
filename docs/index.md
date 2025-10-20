# unitree_rl_lab Documentation

## unitree_rl_lab

https://github.com/YumaMatsumura/unitree_rl_lab.git

## ディレクトリ構成

重要な所のみ。


```
unitree_rl_lab/
├─ deploy/                                  # 学習後の検証/実機化( Sim2Sim → Sim2Real )で使うC++コントローラ群
│  └─ robots/
│     └─ g1_29dof/                          # G1-29DOF向けコントローラ
│        └─ (CMakeプロジェクト)            # README指示に従い「mkdir build && cd build && cmake .. && make」でビルド
│                                            # → 生成バイナリ ./g1_ctrl を Sim2Sim/Sim2Real で使用
│                                            # 依存: unitree_sdk2 を /usr/local にインストール
│                                            # 使い方(例):
│                                            #   Sim2Sim: ./g1_ctrl
│                                            #   実機:   ./g1_ctrl --network eth0
│                                            #   操作:  [L2+Up] 立ち上がり → [R1+X] 実行 など
│                                            # 参照: README の Deploy セクション（手順/キー操作が明記）
│                                            #      （Sim2Sim側の unitree_mujoco の設定も README に記載）
│
├─ doc/
│  └─ licenses/                              # ライセンス表記関係
│
├─ docker/                                   # Docker でIsaac Lab＋依存一式を再現するための設定
│                                            # 参照: ルート一覧（Docker関連フォルダの存在）
│
├─ scripts/                                 # 学習・推論の実行スクリプト置き場
│  └─ rsl_rl/
│     ├─ train.py                           # 学習の実行スクリプト
│     └─ play.py                            # 学習済みポリシーの再生
│                                            # 使い方例:
│                                            #   python scripts/rsl_rl/train.py --headless --task Unitree-G1-29dof-Velocity
│                                            #   python scripts/rsl_rl/play.py  --task Unitree-G1-29dof-Velocity
│                                            #   （同等のショートカットは unitree_rl_lab.sh 経由でも可）
│                                            # 参照: README の Installation/Verify/Running の例コマンド
│
├─ source/
│  └─ unitree_rl_lab/
│     └─ unitree_rl_lab/
│        └─ assets/
│           └─ robots/
│              └─ unitree.py                 # モデル参照パスの設定スクリプト
│                                            #   ・USD を使う場合: UNITREE_MODEL_DIR を設定（Hugging Faceのunitree_modelを取得）
│                                            #   ・URDFを使う場合: UNITREE_ROS_DIR を設定（unitree_rosを取得; IsaacSim 5.0+ 推奨）
│                                            #   ・必要に応じて spawn 設定をURDFモードに切替
│                                            # 参照: README の「Download unitree robot description files」節
│
└─ unitree_rl_lab.sh                         # セットアップ＆実行ショートカット
                                             #   -i : ライブラリを editable でインストール（Isaac Lab環境のPythonで）
                                             #   -l : タスク一覧表示（IsaacLab本体より高速版のリスト）
                                             #   -t : 学習実行（train.py相当）
                                             #   -p : 再生実行（play.py相当）
                                             # 参照: README の Installation/Verify の手順
```
