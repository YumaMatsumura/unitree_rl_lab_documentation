# unitree_rl_lab Documentation

## 概要

**unitree_rl_lab** は、Unitree社のロボット（Go2、H1、G1など）向けの強化学習環境です。Isaac Labをベースに、ロコモーション（歩行制御）やモーショントラッキング（モーションキャプチャデータの再現）など、様々なタスクに対応しています。

## リポジトリ

[https://github.com/YumaMatsumura/unitree_rl_lab.git](https://github.com/YumaMatsumura/unitree_rl_lab.git)

## 主な機能

### ロコモーションタスク (Locomotion)
速度コマンドに従ってロボットを歩行させるタスクです。カリキュラム学習により、段階的に複雑な地形での歩行を学習できます。

**対応ロボット**: Go2, H1, G1 (29DOF)

### モーショントラッキングタスク (Mimic)
BVH形式のモーションキャプチャデータを読み込み、ロボットに再現させるタスクです。ダンスなどの複雑な全身動作の学習に適しています。

**対応ロボット**: G1 (29DOF)

## 使い方の流れ

1. **環境構築**: Isaac Labのインストールとunitree_rl_labのセットアップ
2. **学習の実行**: タスクとロボットを選択して強化学習を開始
3. **パラメータ調整**: 報酬関数や学習パラメータをカスタマイズ
4. **実機展開**: 学習済みポリシーをC++コントローラに組み込んで実機で動作

## クイックスタート

```bash
# ライブラリのインストール
./unitree_rl_lab.sh -i

# タスク一覧の表示
./unitree_rl_lab.sh -l

# 学習の実行（Go2の速度追従タスク）
./unitree_rl_lab.sh -t Unitree-Go2-Velocity

# 学習済みポリシーの再生
./unitree_rl_lab.sh -p Unitree-Go2-Velocity
```

または、直接Pythonスクリプトを実行：

```bash
# 学習
python scripts/rsl_rl/train.py --headless --task Unitree-Go2-Velocity

# 再生
python scripts/rsl_rl/play.py --task Unitree-Go2-Velocity
```

## ドキュメント構成

このドキュメントでは、以下の内容を説明します：

- **ディレクトリ構成**: リポジトリの全体構造と各ディレクトリの役割
- **パラメータ設定**: 学習や動作をカスタマイズするための設定ファイルの詳細
  - 環境設定（地形、報酬、観測など）
  - PPOアルゴリズム設定
  - モーショントラッキング設定
  - デプロイ設定（実機用FSM）
- **チューニングガイド**: 目的別のパラメータ調整方法

## 必要な環境

- **OS**: Ubuntu 22.04 LTS
- **GPU**: NVIDIA GPU (RTX 3060以上推奨)
- **Isaac Lab**: 最新版
- **Python**: 3.10+
- **CUDA**: 12.x

## ライセンス

このプロジェクトのライセンスについては、リポジトリの[LICENCE](https://github.com/YumaMatsumura/unitree_rl_lab/blob/main/LICENCE)ファイルを参照してください。
