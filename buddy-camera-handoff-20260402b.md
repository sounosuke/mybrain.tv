# セッション引き継ぎ — 2026-04-02（夜）

## 状況サマリー

ローカルLLM（Cline + Gemini指示）による作業がコンテキスト不足で混乱を引き起こした。
フォルダ構造の再編が途中で壊れた状態。次のセッションでClaude（Cowork）と一緒に整理する。

---

## 1. フォルダ構造（現在の実体）

### Mac上のパス対応表

| パス | 中身 | 状態 |
|------|------|------|
| `~/Documents/mybrain.tv/` | ウェブサイト（旧gh-pages）独立gitリポジトリ | ✅ 綺麗 |
| `~/Documents/mybrain-tv-workspace/` | メイン作業リポジトリ `sounosuke/mybrain-tv-workspace.git` | ⚠️ 再編途中 |
| `~/Documents/mybrain.tv.bak/` | 元の `mybrain.tv/` 一枚岩フォルダ（バックアップ） | ✅ 最も完全 |
| `~/Documents/mybrain.tv.bk2/` | ローカルLLMの移行失敗後にSonosukeが手動退避 | ⚠️ 不完全 |

### Coworkマウント対応（このセッション）

| マウント名 | 実体 |
|---|---|
| `mybrain.tv` | `~/Documents/mybrain.tv/` |
| `mybrain-tv-workspace` | `~/Documents/mybrain-tv-workspace/` |
| `mybrain.tv.bak` | `~/Documents/mybrain.tv.bak/` |
| `mybrain.tv.bk2` | `~/Documents/mybrain.tv.bk2/`（重複あり → `Documents--mybrain.tv.bk2` を外す） |

---

## 2. mybrain-tv-workspace の状態

### 正常にある

```
buddy-prototype/      # IPCステートマシン、音声エージェント、memory-v2実装（ローカルLLMが大量実装）
memory/               # ← 内容が不完全（下記参照）
buddy-blog/
buddy-agent-orchestration.md
buddy-architecture-deep-dive.md
buddy-memory-v2.md
buddy-oss-scaffold-analysis.md
buddy-technology-review-log.md
mybrain-tv-way.md
CLAUDE.md
```

### 問題：未追跡の `gh-pages/` フォルダが残っている

`mybrain-tv-workspace/gh-pages/` に `privacy.html` だけが入った空フォルダがある。
`.gitignore` にも入っていない。処分が必要。

### 問題：CLAUDE.md のパスが古い

CLAUDE.md の「プロジェクト構造」セクションがまだ `~/Documents/mybrain.tv/` を指している。
`~/Documents/mybrain-tv-workspace/` に更新が必要。

---

## 3. memory の損傷状況

`mybrain.tv.bak/memory/` を正として、`mybrain-tv-workspace/memory/` との差分：

### workspace に**ない**もの（bak から復元が必要）

```
memory/mybrain.tv/           # プロジェクト固有メモリ（context/ decisions/ investigations/ etc.）
memory/artifacts/
memory/dent/
memory/tablecheck/
memory/shared/decisions/
memory/shared/summaries/
memory/shared/sessions/sonosuke/    ※ workspace には sessions/mybrain.tv/ に改名されている
memory/shared/sessions/dent/
memory/shared/sessions/tablecheck/
memory/shared/tasks/active/
memory/shared/tasks/cancelled/
memory/shared/tasks/completed/
memory/shared/principles/x-post-management-SKILL.md
memory/shared/principles/note-daily-engagement-SKILL.md
memory/shared/principles/buddy-tech-stack-watch-SKILL.md
memory/shared/principles/x-post-reminder-SKILL.md
memory/shared/okr/2026-Q2-Feel-the-AGI/key-results.md
memory/_index/（embeddings.npy含む）
```

### workspace に**あって** bak にないもの（保持すべき新ファイル）

```
memory/archive/ai-virtual-worker-demo-handover.md
memory/archive/ai-virtual-worker-research-report.md
memory/archive/m5-max-verification-plan.md
memory/shared/sessions/mybrain.tv/     ← sonosuke/ の改名版か？要確認
memory/shared/archive/shadow_index.md  ← ローカルLLMが作成した新ファイル
memory/shared/archives/architecture.md ← ローカルLLMが移行
```

### 方針（未決定・次セッションで実施）

- bak の不足ファイルを workspace にコピー
- workspace の新ファイルは上書きしない
- `sessions/sonosuke/` vs `sessions/mybrain.tv/` の命名は要確認

---

## 4. bak にあって workspace にないもの（未移行）

次のセッションで「どこに置くか」を決める必要がある：

| ファイル/フォルダ | 説明 | 優先度 |
|---|---|---|
| `buddy-camera/` | BuddyCamera Xcodeプロジェクト | 🔴 高（このセッションで変更あり） |
| `LAM_WebRender/` | WebRenderアセット | 🔴 高（Meetパイプラインで使用） |
| `LAM_Audio2Expression/` | 音声→表情MLモデル | 🟡 中 |
| `buddy-infra/` | launchd plists、docker-compose、bore tunnel設定 | 🔴 高 |
| `venv/` → `~/mybrain-venv` symlink | Python仮想環境 | 参照のみでOK |
| `vllm-mlx-venv/` | vllm-mlx用venv | 参照のみでOK |
| `sonosuke_face/` | 音声クローン用顔素材 | 🟡 中 |
| `buddy-blog/` 追加記事 | bak側に追加記事があるか要確認 | 🟡 中 |

---

## 5. 前セッション（3/31）の技術的進捗

### ✅ 完了済み

- **getUserMedia hookでMeetにアバター参加**が動作確認済み（117fps、仮想カメラ不要）
- `meet_getusermedia_test.py` 作成（`buddy-prototype/pipeline/` に配置）
- 音声はAudioContext resumeで動くが**最初だけ出る**問題が残る（後述）
- BuddyCameraのアバタードロップダウンを動的スキャン化（`ViewController.swift` 変更済み、未ビルド）

### ⚠️ 未解決

**（A）WebRender映像をMeetに流せていない**
- 原因：`meet.google.com` のページからlocalhost iframeをcanvasに描画するとCORSブロック
- 次のアプローチ：別タブでWebRenderを開き、CDPの`Page.captureScreenshot`でキャプチャ→canvasに流す
- 実装すべきファイル：`meet_getusermedia_test.py` のPhase 2部分

**（B）音声が最初しか出ない**
- AudioContextがgetUserMedia後にsuspendに戻っている疑い
- `window.__buddyAudioCtx`をTTSが使う形で接続する部分が未完
- 現在は440Hzのテストトーンしか試していない

**（C）ViewController.swift 未ビルド**
- 動的アバタードロップダウンの変更がXcodeでビルドされていない
- `buddy-camera/BuddyCamera/BuddyCamera/BuddyCamera/ViewController.swift` に変更あり

---

## 6. ローカルLLMが実装したもの（buddy-prototype内）

ローカルLLMがかなり実装を進めた。品質は未検証：

```
ipc/ipc_state_machine.py   1397行  IPCステートマシン本体
ipc/mlx_backend.py         1080行  MLXバックエンド
ipc/web_search.py           320行  Web検索
core/buddy_core.py          249行  チャネル非依存頭脳層
voice/agent.py              〜600行 LiveKit音声エージェント（STT/LLM/TTS/VAD）
memory/memory_v2_adapter.py        memory-v2アダプター
memory/build_index.py              embeddingインデックス構築
```

**注意**：Geminiの指示がコンテキスト不足だったため、設計思想（mybrain.tv way）との整合性が不明。
次のセッションで読んでレビューする価値あり。

---

## 7. 次のセッションでやること（優先順）

1. **フォルダ整理方針の確定**
   - `mybrain-tv-workspace` に何を入れるか（buddy-camera、LAM_WebRender、buddy-infra 等）
   - 残す場所が決まったらCLAUDE.mdのパスを更新

2. **memory の復旧**
   - bak から不足ファイルをコピー
   - sessions の命名統一（sonosuke/ vs mybrain.tv/）

3. **ローカルLLM実装のレビュー**
   - ipc_state_machine.py、buddy_core.py の設計確認
   - voice/agent.py が実際に動くか確認

4. **WebRender→Meet CDPブリッジ実装**（技術的な次ステップ）
   - `meet_getusermedia_test.py` Phase 2

---

## 8. Coworkセッションで確認したこと

- `mybrain.tv.bak` が最も完全な状態のバックアップ
- `mybrain.tv.bk2/memory/shared/` は `git` フォルダだけ残る壊れた状態
- `gh-pages/` フォルダが workspace に未追跡で残っている（削除 or .gitignore 追加）
- `mybrain-tv-workspace` の `.gitignore` に `.gitignore.bak` が untracked で残っている

---

*作成：Claude (Cowork) — 2026-04-02*
