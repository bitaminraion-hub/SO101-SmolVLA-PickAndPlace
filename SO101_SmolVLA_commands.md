# SO-101 + SmolVLA — 赤キューブ → 黄色領域 Pick and Place

知能ロボティクス講座 応用編 最終課題で使用した **LeRobot CLIコマンド** の記録。

- ロボット：SO-101 Leader / Follower
- カメラ：InnoMaker U20CAM × 2（手元 `wrist`、俯瞰 `overhead`）
- 実行環境：Windows 11 / WSL2（Ubuntu）、NVIDIA RTX 3060 12GB
- ベースモデル：`lerobot/smolvla_base`
- データセット：[Yu3773/so101_red_cube_yellow_target_v4](https://huggingface.co/datasets/Yu3773/so101_red_cube_yellow_target_v4)
- 学習済みモデル：[Yu3773/smolvla_red_cube_yellow_target_v4](https://huggingface.co/Yu3773/smolvla_red_cube_yellow_target_v4)
>
> USBデバイスパス、カメラ番号、データセット保存先、モデルパスは使用PCに依存する。以下はそのまま他のPCで実行できることを保証しない。

## 1. Record — 教師データ追加収録

既存のv4データセットへ **10エピソード**を追加収録するコマンド（1エピソード15秒、リセット5秒）。`--resume=true` のため、新規作成用ではない。

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B7B013681-if00 \
  --robot.id=home_so101_follower \
  --robot.cameras='{
    "wrist":{
      "type":"opencv",
      "index_or_path":"/dev/video0",
      "width":640,
      "height":480,
      "fps":30,
      "fourcc":"MJPG",
      "backend":"V4L2"
    },
    "overhead":{
      "type":"opencv",
      "index_or_path":"/dev/video2",
      "width":640,
      "height":480,
      "fps":30,
      "fourcc":"MJPG",
      "backend":"V4L2"
    }
  }' \
  --teleop.type=so101_leader \
  --teleop.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B7B014699-if00 \
  --teleop.id=home_so101_leader \
  --dataset.repo_id=Yu3773/so101_red_cube_yellow_target_v4 \
  --dataset.root=/home/sakuta/lerobot_datasets/so101_red_cube_yellow_target_v4 \
  --dataset.num_episodes=10 \
  --dataset.episode_time_s=15 \
  --dataset.reset_time_s=5 \
  --dataset.single_task="Pick up the red cube and place it inside the yellow target area" \
  --resume=true \
  --display_data=true
```

## 2. Train — SmolVLAファインチューニング

モデルの保存済み `train_config.json` から復元した主要設定：20,000 steps、batch size 16、学習率 `1e-4`、checkpoint 5,000 stepsごと、ログ200 stepsごと、空カメラ1つ、画像キーの変換。**原コマンドの完全な記録ではない。**

```bash
lerobot-train \
  --dataset.repo_id=Yu3773/so101_red_cube_yellow_target_v4 \
  --dataset.root=/home/sakuta/lerobot_datasets/so101_red_cube_yellow_target_v4 \
  --policy.path=lerobot/smolvla_base \
  --policy.device=cuda \
  --policy.empty_cameras=1 \
  --rename_map='{"observation.images.overhead":"observation.images.camera1","observation.images.wrist":"observation.images.camera2"}' \
  --output_dir=/home/sakuta/outputs/train/smolvla_red_cube_yellow_target_v4 \
  --job_name=smolvla_red_cube_yellow_target_v4 \
  --batch_size=16 \
  --steps=20000 \
  --save_checkpoint=true \
  --save_freq=5000 \
  --log_freq=200 \
  --wandb.enable=false \
  --policy.push_to_hub=false
```

> 保存済み設定では `policy.push_to_hub=true`。ここでは意図しない再公開を避けるため **`false` に変更**してある。したがって完全同一設定ではない。学習率 `0.0001` 等のその他の詳細パラメータは保存済み設定に従う想定であり、現在のLeRobotの既定値が同一とは限らない。正確な再現には[公開モデルの学習設定](https://huggingface.co/Yu3773/smolvla_red_cube_yellow_target_v4/blob/main/train_config.json)と対応するLeRobotバージョンを参照。

## 3. Rollout — 学習済みポリシーの実機推論

20,000 stepsのチェックポイントを使用し、15秒間、赤キューブを黄色領域へ移すタスクを実行したコマンド。

```bash
lerobot-rollout \
  --strategy.type=base \
  --robot.type=so101_follower \
  --robot.port=/dev/serial/by-id/usb-1a86_USB_Single_Serial_5B7B013681-if00 \
  --robot.id=home_so101_follower \
  --robot.cameras='{
    "wrist":{
      "type":"opencv",
      "index_or_path":"/dev/video0",
      "width":640,
      "height":480,
      "fps":30,
      "fourcc":"MJPG",
      "backend":"V4L2"
    },
    "overhead":{
      "type":"opencv",
      "index_or_path":"/dev/video2",
      "width":640,
      "height":480,
      "fps":30,
      "fourcc":"MJPG",
      "backend":"V4L2"
    }
  }' \
  --policy.path=/home/sakuta/outputs/train/smolvla_red_cube_yellow_target_v4/checkpoints/020000/pretrained_model \
  --policy.device=cuda \
  --policy.empty_cameras=1 \
  --rename_map='{
    "observation.images.overhead":"observation.images.camera1",
    "observation.images.wrist":"observation.images.camera2"
  }' \
  --task="Pick up the red cube and place it inside the yellow target area" \
  --duration=15 \
  --display_data=false
```

## 4. 工夫した点と結果

- **教師データ収録時の動作統一：** 手元カメラで開いたグリッパの中心にキューブを合わせ、接近して把持する動作を意識した。
- **グリッパの改良：** 黒色EPDMの滑り止め材を把持面に貼り、把持安定性と開閉状態の視認性の改善を狙った。
- **USB接続先の識別：** `/dev/ttyACM0` などの変動する番号ではなく、`/dev/serial/by-id/` を指定した。
- **実機結果：** 教師データ収録時に近い初期位置ではPick and Place動作を確認。初期位置を変えると成功しにくい傾向があり、成功率の定量評価は未実施。


