# StepAI-Ver.A

OpenAI の gpt-realtime で音声入力→LLM 応答（STT/LLM）は WebRTC、最終的な音声合成（TTS）は ElevenLabs を使うリアルタイム音声アシスタントです。OpenAI 側の音声出力は使わず、テキスト相当（audio_transcript）を受けて ElevenLabs に読み上げさせます。

- STT/LLM: OpenAI gpt-realtime（WebRTC 直結、ephemeral key）
- TTS: ElevenLabs（サーバの `/tts` プロキシ経由）
- 発話区切り: サーバ VAD（turn_detection）を使用

## 構成とデータフロー
1. ブラウザでマイク許可 → WebRTC で OpenAI Realtime に接続（`/session` で ephemeral key を取得）
2. 音声は OpenAI へ送信（`modalities: ['text','audio']`）。応答は audio_transcript（テキスト）として受信
3. 受け取ったテキストをサーバの `/tts` に送信 → ElevenLabs が音声（MP3）をストリーミング返却 → ブラウザで再生

## 必要要件
- Node.js 18+（推奨）と npm
- モダンブラウザ（マイク許可必須）
- OpenAI API キー（gpt-realtime へのアクセスが有効なアカウント）
- ElevenLabs API キー（任意の声での TTS に必要）

## インストールと起動
1) 依存をインストール
```
npm install
```

2) 環境変数を設定（プロジェクト直下に `.env` を作成）
```
OPENAI_API_KEY=sk-...
# 既定で gpt-realtime を使用（不要なら明示）
REALTIME_MODEL=gpt-realtime
# ElevenLabs TTS を使うために必須
ELEVENLABS_API_KEY=xi-...
# /voices が使えない/失敗する環境のフォールバック（任意）
ELEVENLABS_VOICE_ID=xxxxxxxxxxxxxxxxxxxxxxxx
# 任意
PORT=3000
```

3) 起動
- HTTP で起動（localhost での検証）
```
npm start
```
- HTTPS で起動（推奨）
```
npm run generate-ssl
npm run https
```

4) ブラウザでアクセス
- `http://localhost:3000`（もしくは `https://localhost:3000`）
- 初回に ElevenLabs のボイス一覧を取得しドロップダウンに反映します（API キー必須）
- 「🎤 話す」を押す → 接続→録音開始（Listening…）→ 話すと Transcript に AI 文→ ElevenLabs で音声再生

## 主要スクリプト
- `npm start`: HTTP サーバ起動
- `npm run https`: HTTPS サーバ起動（事前に `npm run generate-ssl`）
- `npm run generate-ssl`: ローカル用の自己署名証明書を作成（`localhost.pem`/`localhost-key.pem`）

## サーバ API（ローカル）

### GET `/health`
- 用途: ヘルスチェック
- レスポンス例: `{ "ok": true }`

### POST `/session`
- 用途: OpenAI Realtime の ephemeral key を取得（WebRTC 用）
- リクエスト（例）
```
POST /session
Content-Type: application/json

{
  "modalities": ["text","audio"],
  "instructions": "あなたは丁寧で簡潔な日本語の音声アシスタントです。"
}
```
- 備考: サーバ側で `turn_detection: { type: 'server_vad' }` を付与します
- レスポンス（例）
```
{
  "client_secret": "ephemeral_...",
  "model": "gpt-realtime",
  "voice": null
}
```

### POST `/tts`
- 用途: ElevenLabs のストリーミング TTS プロキシ
- リクエスト（例）
```
POST /tts
Content-Type: application/json

{
  "text": "こんにちは、今日はどのようにお手伝いできますか？",
  "voiceId": "<任意: 指定がなければ .env もしくは /voices の先頭を使用>"
}
```
- レスポンス: `audio/mpeg`（MP3 ストリーミング）
- エラー: `401/400/5xx`（API キー無効、voice 未指定など）

### GET `/voices`
- 用途: ElevenLabs のボイス一覧を取得（5 分キャッシュ）
- レスポンス（例）
```
{
  "voices": [
    { "id": "sQXIzk...", "name": "Bella" },
    { "id": "abc123...", "name": "Josh" }
  ]
}
```
- 備考: `ELEVENLABS_API_KEY` が必要。401 の場合はキーまたは権限を確認してください。

## ブラウザ UI について
- Transcript: 受信した `response.audio_transcript.delta` をテキストとして表示
- TTS: Transcript 確定時（`response.audio_transcript.done` or `response.done`）に `/tts` を呼び出し音声を再生
- プロンプト: 画面のテキストエリアを編集→「適用」で `session.update`
- ボイス: ページロード時に `/voices` で取得→ドロップダウンへ。選択は localStorage に保存

## よくあるトラブルと対処
- `/voices` が 401/500
  - サーバログが `ElevenLabs voices error: 401` → ELEVENLABS_API_KEY が無効/権限不足
  - 有効なキーに差し替え。もしくは `.env` に `ELEVENLABS_VOICE_ID` を指定
- 文字は出るが音声が出ない
  - ブラウザの Network で `/tts` のレスポンスコードを確認
  - ドロップダウンが空 → キー/権限、あるいは `.env` の VOICE_ID を確認
- 接続できない/反応がない
  - DevTools Console の状態ログを確認: `pc.signalingState` / `pc.iceGatheringState` / `pc.iceConnectionState` / `pc.connectionState` / `dc.onopen`
  - HTTPS で試す（`npm run https`）。企業ネットワーク・プロキシ環境では TURN が必要な場合があります
- マイクが動かない
  - ブラウザのマイク許可、他アプリの独占を確認

## 実装上の注意
- モデルは `gpt-realtime` を既定使用（フォールバック無し）。必要なら `.env` で上書き
- WebRTC は非トリクル ICE で SDP を HTTP 交換（`OpenAI-Beta: realtime=v1`）
- STUN: `stun:stun.l.google.com:19302`
- OpenAI の音声トラックは受け取らない（`sendonly`）。テキスト相当（audio_transcript）のみを扱う
- ElevenLabs へは `audio/mpeg` でストリーミングプロキシ

## セキュリティ
- `.env` は Git 管理外（`.gitignore` 済み）。秘密情報はコミットしない
- OpenAI の ephemeral key は短期（≒1 分）
- 本番運用は HTTPS 推奨。プロキシ/WAF 配下ではヘッダや圧縮の扱いに注意

## ライセンス
- 未指定（必要に応じて追記してください）

---
不明点や追加の機能（WebSocket フォールバック、TURN サーバ導入、音声バー表示など）が必要であればお知らせください。
