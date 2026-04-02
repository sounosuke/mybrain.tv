# BuddyCamera ハンドオフ — 2026-04-02

前回: `buddy-camera-handoff-20260331.md`

---

## 今日の成果

### 1. アバタードロップダウンの動的化（ViewController.swift）

`asset/arkit/` ディレクトリをスキャンして zipファイルを自動的にリストアップするようにした。背景ドロップダウンと同じ方式。

- `loadAvatarList()` を追加（`loadBackgroundList()` と対称）
- `avatarChanged()` をシンプル化。ハードコードのパス配列を廃止し、`representedObject` からパスを取得
- 新しいzipを `asset/arkit/` に置くだけで次回起動時にドロップダウンに自動で出る

### 2. getUserMediaフック方式の検証

**経緯**: WKWebView/仮想カメラ方式の代わりに、PlaywrightのヘッドレスChromeがgetUserMediaをフックして直接Meetに参加できるか検証した。

**スクリプト**: `buddy-prototype/pipeline/meet_getusermedia_test.py`

**重要な発見**:
- Playwrightバンドル版ChromiumはGoogleにbot検知されてMeetに参加できない
- 通常のGoogle Chrome（`/Applications/Google Chrome.app`）を使えばゲスト参加できる
- `--use-fake-device-for-media-stream` フラグが必要（これがないとgetUserMediaフックが呼ばれないケースがある）
- フックは `MediaDevices.prototype.getUserMedia` をprototypeレベルで上書きする必要がある（インスタンスレベルだけでは不十分）
- 音声も同じフックでAudioContext経由で流せることを確認（440Hz テストトーンをMeetに流すことに成功）

**判明したこと（重要）**:
- 最初にアバターが映ったように見えたのは、**BuddyCameraが選ばれていたから**（仮想カメラが動いていた）
- getUserMediaフック自体が映像を映したわけではない
- `--use-fake-device-for-media-stream` なしだとフックが呼ばれないことがある

**ログで確認できた動作（フックが効いている状態）**:
```
[GUM Hook] getUserMedia called with: {"video":{"deviceId":{"exact":"6fd065..."},...}}
[GUM Hook] AudioContext resumed: running
[GUM Hook] Returning fake stream, video: 1 audio: 0
Frames: 4144  |  FPS: 117.0
```

---

## 次のタスク（優先順）

### 最優先: WebRender映像をgetUserMediaフック経由でMeetに流す

現在の問題: `--webrender` フラグでiframeにWebRenderを埋め込もうとするとCORSでブロックされる。

**解決策**: iframeではなく、**WebRenderを別タブで開いてCDPでスクリーンショットを撮り、MeetのcanvasにpostMessageまたはSharedArrayBufferで送る**。

具体的な実装方針:
1. Playwrightで2ページを開く（WebRenderページ + Meetページ）
2. WebRenderページからCDP `Page.captureScreenshot` でJPEGフレームを取得（`capture_to_vcam.py` と同じ方法）
3. フレームをMeetページのcanvasにpython側で橋渡し（`page.evaluate` でBase64→drawImage）
4. getUserMediaフックがそのcanvasを映像ソースとして返す

これは `capture_to_vcam.py` + 仮想カメラの組み合わせを、仮想カメラなしで実現するもの。

**参考**: `capture_to_vcam.py` の CDP captureScreenshot部分をそのまま流用できる。

### 音声（TTS → Meet）

TTS出力をgetUserMediaフックの音声トラックに流す。仕組みは確認済み:
- `window.__buddyAudioCtx`（AudioContext）と `window.__buddyAudioDest`（MediaStreamDestination）を公開済み
- PythonからBase64でWAVを送り、Chromeの AudioContextでデコード → Destinationノードに接続するだけ
- `getUserMedia` が呼ばれたタイミングでAudioContextをresumeする処理を実装済み

### Xcode ビルド（ViewController.swift の変更）

今日変更した内容（アバタードロップダウン動的化）をXcodeでビルドして動作確認する。

---

## アーキテクチャの将来設計（今日議論したこと）

### アバター配布基盤
- zipファイルの構造: 4ファイル（animation.glb / offset.ply / vertex_order.json / skin.glb）
- 固有なのは `skin.glb` だけ、残り3つは全モデル共通
- 将来: `template.zip`（共通アセット）からのフォールバックローダーをWebRenderに実装
  - zipの中になければ `template.zip` から取得する
  - アバター作者は `skin.glb` だけのzipを作ればよい

### マルチエージェント会議
- Sonosuke（生カメラ）+ 複数エージェント（アバター）でMeet会議
- 各エージェントが独立したChromeインスタンスでMeetにゲスト参加（getUserMediaフック方式）
- emotion_wsにアバターIDを追加: `{"type": "set_emotion", "emotion": "happy", "avatar": "buddy"}`
- 各エージェントのGoogleアカウントは不要（ゲスト参加 + 主催者承認）

### エージェントの個性（顔 + 声 + 性格）
- 顔: `skin.glb` で個別のアバター
- 声: Qwen3-TTS の voice cloning でエージェントごとに固有の声（リファレンス音声を差し替えるだけ）
- 会議でアバターが喋る声がMeetに流れる

### リモートレンダリング
- M5 MaxをレンダリングサーバーとしてWebRenderを動かす
- 非力な端末（Mac Mini等）はストリームを受信してgetUserMediaフック経由でMeetに参加
- レンダリングのGPU負荷はM5 Maxに集約

---

## 既知の問題・注意事項

### getUserMediaフックのCORS問題
- MeetページにlocalHostのiframeを埋め込むとCORSでブロックされる
- → 別ページで開いてCDP経由でフレームを橋渡しする方式に変更必要

### AudioContextのautoplay policy
- init script内でAudioContextを作成すると `suspended` 状態になる
- `getUserMedia` が呼ばれたタイミング（ユーザー操作起因）でresumeするとうまくいく
- フックの中で `await audioCtx.resume()` を呼ぶことで対応済み

### `--use-fake-device-for-media-stream` の必要性
- このフラグがないと、Meetが特定のカメラデバイスIDを指定してgetUserMediaを呼ぶ
- フラグがあると fake device IDでの呼び出しが保証される
- フックはそれをキャッチして canvas ストリームを返す

### BuddyCameraのアバタードロップダウン（ViewController.swift）
- ビルドがまだ。変更点: `loadAvatarList()` の追加と `avatarChanged()` のシンプル化
- Xcodeでビルドして確認が必要

---

## 実行コマンド

```bash
# getUserMedia フックテスト（Canvas テストパターン）
cd ~/Documents/mybrain.tv/buddy-prototype/pipeline
source ../../venv/bin/activate
python3 meet_getusermedia_test.py --meet-url "https://meet.google.com/xxx-xxxx-xxx"

# getUserMedia フックテスト（音声テストトーン付き）
python3 meet_getusermedia_test.py --meet-url "https://meet.google.com/xxx-xxxx-xxx" --audio-test

# 従来の仮想カメラパイプライン（引き続き動作確認済み）
python3 capture_to_vcam.py --backend buddy
```

---

## 今日の反省

- ゴールを見据えずに目の前の技術的問題を解くことに終始していた
- getUserMediaフックの前に「仮想カメラ自体が不要では？」という問いを立てるべきだった
- 次回からタスクに入る前に「このタスクの先にあるゴールは何か」を確認する

---

*作成: 2026-04-02 by Claude*
