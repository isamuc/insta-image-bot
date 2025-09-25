# insta-image-bot アーキテクチャ案

## 1. コンセプト
- Google Drive への画像アップロードをトリガーに、Instagram 投稿用の素材（画像・キャプション）を自動生成。
- 投稿前レビューや失敗時のリトライができる柔軟なフローを維持しつつ、最小限の手作業に抑える。

## 2. システム構成（ローカル簡易版）
- **Google Drive**: 入力画像の保存先。フォルダ ID 単位で監視。
- **Drive Poller / Watcher**: 新規ファイルを検知。ローカル PC でのバッチ実行を想定。
- **Image Processor**: Pillow 等でリサイズ・テンプレ適用を行うモジュール。
- **Caption Generator**: テンプレ埋め込み＋（必要に応じて）LLM API を呼び出す層。
- **Post Scheduler**: Meta Graph API 経由で下書きまたは投稿。API 認証情報を安全に保持。
- **Notification & Logging**: Slack/メール通知、処理ログ記録。

```
[Google Drive] --(新規画像)--> [Drive Watcher Script]
     |                                   |
     |                                   v
     |                            [Image Processor]
     |                                   |
     |                                   v
     |                            [Caption Generator]
     |                                   |
     |                                   v
     |                            [Post Scheduler]
     |                                   |
     v                                   v
[通知先 (Slack 等)] <--- [Notification & Logging]
```

## 3. クラウド運用への拡張案
- **Cloud Storage/Drive → Cloud Functions**: Google Drive API webhook で Cloud Functions (GCP) または Cloud Run をトリガー。
- **処理キュー（Pub/Sub / SQS）**: 同時実行を制御し、リトライ制御を明確化。
- **Secrets Manager**: Instagram / OpenAI 等の認証情報を管理。
- **監視基盤**: Cloud Logging + Alerting。失敗件数閾値で Slack 通知。

## 4. データフロー
1. ユーザーが Drive の `投稿待ち` フォルダに画像をアップロード。
2. Watcher が新規ファイルリストを取得し、未処理ファイルのみキューへ登録。
3. Image Processor が 1080x1350/1080x1080 などの比率に合わせて加工。
4. Caption Generator がテンプレ＋AI で本文案・ハッシュタグ・CTA を生成し、投稿用 JSON を組み立て。
5. Post Scheduler が Meta Graph API に対し下書き登録 or 予約投稿 API を呼び出す。
6. 処理結果（成功 / 失敗）をログに保存し、通知を送信。失敗時はフォールバック（手動投稿用ファイル生成など）。

## 5. シーケンス案（主要パス）
1. `Drive Watcher` → `Drive API`: 新規ファイル一覧取得。
2. `Drive Watcher` → `Local Storage`: 対象ファイルを一時保存。
3. `Image Processor` → `Caption Generator`: 加工済み画像パスとメタ情報を渡す。
4. `Caption Generator` → `Template Store / AI API`: トーン＆マナー設定を読み込み、文章生成。
5. `Post Scheduler` → `Meta Graph API`: 下書き登録。成功レスポンスを受け取る。
6. `Post Scheduler` → `Notification`: Slack / メールで完了報告。

## 6. データ管理
- **テンプレ設定**: JSON / YAML で保管し、キャプションテンプレ・ハッシュタグセットを定義。
- **処理履歴**: SQLite / Google Sheets / Firestore など軽量 DB に `post_id`, `image_id`, `status`, `timestamp`, `error` を記録。
- **秘密情報**: `.env` → `.gitignore` でバージョン管理から除外。クラウド展開時は Secrets Manager を利用。

## 7. 非機能設計メモ
- **可観測性**: 主要イベントごとにログレベルを定義（INFO: 成功、WARN/ERROR: 失敗）。
- **リトライ**: API 呼び出し失敗時は指数バックオフ。3 回失敗で人間に通知。
- **セキュリティ**: OAuth トークンの更新ジョブ、Instagram 二要素認証の扱い検討。
- **テスト観点**: 画像加工の差異、テンプレ変数の欠損、API モックとの結合テスト。

## 8. 次ステップ
1. ユースケースごとの詳細シーケンス図を作成（通常投稿 / 予約投稿 / 失敗リトライ）。
2. 主要モジュールの責務・インターフェイスをクラス図レベルで定義。
3. ローカル簡易版で PoC → 要件次第でクラウド構成に拡張。

## 9. 外部デザイン API 検討（Canva / Figma）
- **導入目的**: テンプレベースの高品質デザインを自動生成し、ブランド統一と加工時間短縮を実現する。
- **共通考慮事項**:
  - API 利用には開発者登録・審査が必要。提供範囲はβ/限定公開の可能性。
  - テンプレ管理・差し替えルールを事前定義し、Image Processor からは抽象化されたインターフェイスで呼び出す。
  - エクスポート待ち時間・失敗時フォールバック（ローカル Pillow 実装等）を併用。

- **Canva API**:
  - 長所: 公式テンプレート資産が豊富。画像差し替えやテキスト更新 → エクスポートまで一連操作が可能。
  - 注意点: 現時点で一部機能は限定公開。処理完了コールバックやジョブ完了待機の実装が必要。
  - 想定フロー: `Image Processor` が Canva Design のテンプレ ID と差し替えパラメータを送信 → Render ジョブ完了 → ダウンロード URL 取得。

- **Figma API**:
  - 長所: デザインの構造を JSON で取得可能。トークンがあれば任意ノードの編集も理論上可能（REST + Plugin/API 組み合わせ）。
  - 注意点: 画像レンダリングは画像エクスポート API を使用。テンプレ差し替えは Plugin（Figma/Dev Mode）側のロジックが必要になるケースが多い。
  - 想定フロー: Figma ファイル内のテンプレレイヤーを Plugin/Widget で更新 → エクスポート API で既定サイズの画像を取得。

- **比較まとめ**:
  - 手軽さ: Canva > Figma（テンプレ差し替えまで公式サポート）。
  - カスタム自由度: Figma > Canva（細かいレイアウト制御、チームでの設計共有）。
  - 実行までの所要時間: いずれもレンダリング待ちが発生。バッチ処理でのジョブ並列・キュー制御が必須。
  - 導入難易度: Canva は公式ワークフローに沿えば比較的簡単だが審査が厳しめ。Figma はPlugin開発が必要で実装負荷が高い。

- **推奨方針**:
  1. 短期 PoC: Pillow ベースで MVP 完成 → ブランド要件が厳しい場合に Canva API を追加検証。
  2. 長期拡張: Figma でテンプレ管理し、Plugin で差し替え自動化 → API 連携で書き出し。
  3. いずれの場合も `Image Processor` を抽象化（`process(image, template_id)` インターフェイス）し、実装差し替えを容易にする。

