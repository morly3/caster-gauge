# Caster Gauge

スマートフォンのジャイロセンサーを使ってキャスター角を測定するWebアプリです。  
3Dプリント製アライメントゲージと組み合わせて使用します。

## デモ

GitHub Pages: `https://<your-username>.github.io/caster-gauge/`

## 使い方

1. アライメントゲージをハブボルトに固定し、ゲージ面を垂直にする
2. スマートフォンをゲージ面に貼り付ける
3. ブラウザでアプリを開く（iOS: センサー許可が必要）
4. **ゼロリセット** を押す
5. **記録開始** を押す
6. ステアリングをゆっくり左右に切る
7. **記録停止** → **キャスター算出**

## 測定原理

ステアリングを切るとホイールはキングピン軸まわりに回転します。  
キャスター角が存在する場合、転舵に伴いキャンバー角が変化します。

スマホのジャイロで転舵角（ヨー）とキャンバー変化（ロール）を同時記録し、  
左右最大転舵時のキャンバー差からキャスター角を算出します。

```
キャスター角 = (左切りキャンバー − 右切りキャンバー) / (2 × sin(据え切り角))
```

ジャイロで転舵角を直接測定するため、据え切り角のバラつきが結果に影響しません。

## 機能

- ヨー・ピッチ・ロール リアルタイム表示（EMA平滑化）
- ゼロリセット（初回自動設定）
- 10Hz データ記録
- キャスター角自動算出
- CSV保存（タイムスタンプ付き）
- iOS Safari 対応（DeviceOrientationEvent.requestPermission）

## 動作要件

- HTTPS環境（GitHub Pages、ローカルサーバー等）
- DeviceOrientation API 対応ブラウザ
  - iOS Safari
  - Android Chrome

## セットアップ

```bash
git clone https://github.com/<your-username>/caster-gauge.git
cd caster-gauge
# GitHub Pages: Settings → Pages → Source: main branch
```

ファイルは `index.html` の1枚のみです。

## ライセンス

MIT

## 関連

- [簡易アライメントゲージ by 3D Printing｜mk](https://note.com/)
