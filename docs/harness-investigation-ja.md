# OpenCode ハーネス調査レポート

## 1. まず結論（ハーネスの全体像）

OpenCode のハーネスは、ざっくり次の 6 層で動きます。

1. **SessionPrompt** がユーザー入力をメッセージ化し、実行ループを回す
2. **Agent** が「build / plan / explore など」の権限・性格を決める
3. **SystemPrompt + InstructionPrompt** がシステムプロンプトを合成する
4. **ToolRegistry + PermissionNext** が使えるツールと許可判定を決める
5. **LLM.stream** が provider 向けに最終変換してストリーム実行する
6. **SessionProcessor** が tool-call / text / reasoning を逐次処理して履歴へ反映する

---

## 2. ハーネスの実行フロー（コード起点）

### 2.1 エントリ

- `SessionPrompt.prompt()` がユーザーメッセージを作成し、必要なら `loop()` を開始します。
- ここで旧式 `tools` 指定は permission へ変換されます。

### 2.2 ループ内でやっていること

- 既存メッセージ列を収集し、必要なら「途中のユーザー追加入力」を `<system-reminder>` で包んでモデルに再提示します。
- `SystemPrompt.environment()` と `InstructionPrompt.system()` を足し、最終的な system 群を作ります。
- `resolveTools()` で ToolRegistry 由来のツールと MCP ツールをまとめ、各 execute に plugin hook / permission ask / truncation を挿入します。
- `SessionProcessor.process()` を呼び、モデルのストリームを処理します。
- tool-calls が続く限りループを継続し、終了理由が stop 系なら抜けます。

### 2.3 ストリーム処理

`SessionProcessor` は以下を担当します。

- reasoning-start/delta/end を `reasoning` パートとして保存
- tool-input/tool-call イベントを `tool` パートとして保存
- ツール実行後の結果（output, metadata, attachments）をメッセージに反映
- 同じ失敗が連続する doom loop を検知して停止

---

## 3. 「プロンプトは何が裏で動くか」

### 3.1 ベースの provider 別 system prompt

`SystemPrompt.provider()` がモデル ID で振り分けます。

- `gpt-5` 系 → `codex_header.txt`
- `gpt-* / o1 / o3` → `beast.txt`
- `gemini-*` → `gemini.txt`
- `claude` → `anthropic.txt`
- `trinity` → `trinity.txt`
- その他 → `qwen.txt`

加えて `SystemPrompt.environment()` が実行環境情報（ディレクトリ、git 判定、OS、日付など）を system に注入します。

### 3.2 AGENTS.md などローカル指示の注入

`InstructionPrompt.system()` が次を読み込み、system prompt に追記します。

- リポジトリ階層の `AGENTS.md / CLAUDE.md / CONTEXT.md`
- グローバル設定ディレクトリ側の `AGENTS.md`
- config.instructions のローカルファイルや URL

この仕組みで「実行時の repo 固有ルール」が毎ターン system 側に混ざります。

### 3.3 エージェント固有 prompt

`Agent` 定義で prompt が差し込まれるものがあります。

- `explore` → `agent/prompt/explore.txt`
- `compaction` → `agent/prompt/compaction.txt`
- `summary` → `agent/prompt/summary.txt`
- `title` → `agent/prompt/title.txt`

`build`/`plan` は主に permission と mode の違いで振る舞いを変えます。

### 3.4 plan モード時の特殊リマインダ

`SessionPrompt` は plan への遷移時、巨大な `<system-reminder>` を synthetic user part として注入します。
内容は「plan ファイル以外は編集禁止」「explore を並列で使う」「最後に plan_exit」など、実行手順そのものです。

### 3.5 tool 側の説明プロンプト

各ツールにも説明テキスト（例: `tool/bash.txt`, `tool/task.txt`）があり、Tool 定義の `description` としてモデルへ渡されます。
つまり、実際は「system prompt + 会話履歴 + ツール説明＋schema」の合成で行動が決まります。

---

## 4. ハーネスの制御ポイント（重要）

### 4.1 権限制御

- `PermissionNext.ask()` が rule を評価し、`allow/deny/ask` を決定
- `ask` の場合は Bus イベントで承認待ち
- `always` 承認時は同セッションの pending をまとめて解放

### 4.2 ツール有効化

- `ToolRegistry.tools()` がモデルやフラグでツールを出し分け
  - `gpt-*` の一部では `apply_patch` を使い `edit/write` を抑制
  - `websearch/codesearch` は provider/flag 条件で有効化
- `LLM.resolveTools()` で agent permission により最終除外

### 4.3 provider 変換

`LLM.stream()` は provider options / headers / max tokens / prompt 変換ミドルウェアを挟み、最終的に AI SDK `streamText` を呼びます。

---

## 5. 読む順番（実践的ロードマップ）

「ハーネスを深く理解する」目的なら、次の順で読むのが効率的です。

1. `packages/opencode/src/session/prompt.ts`
   - 入口・ループ・plan/build 切り替え・tools 解決が一番集約されています。
2. `packages/opencode/src/session/processor.ts`
   - ストリームイベントがメッセージ/パートへどう落ちるかを理解できます。
3. `packages/opencode/src/session/llm.ts`
   - provider への最終 payload 化、system 合成、toolChoice、headers を追います。
4. `packages/opencode/src/session/system.ts`
   - provider 別システムプロンプト分岐を把握します。
5. `packages/opencode/src/session/instruction.ts`
   - AGENTS.md 等の注入仕様（スコープ探索）を把握します。
6. `packages/opencode/src/agent/agent.ts`
   - build/plan/explore の性格差（permission/mode/prompt）を確認します。
7. `packages/opencode/src/tool/registry.ts`
   - 実際に何のツールが露出されるかを確認します。
8. `packages/opencode/src/tool/bash.ts`
   - 代表ツールの permission ask、abort/timeout、metadata 更新の実装を読みます。
9. `packages/opencode/src/permission/next.ts`
   - allow/deny/ask の評価順と pending 解放ロジックを読みます。
10. 最後に prompt テキスト群（`session/prompt/*.txt`, `tool/*.txt`）
    - コードを読んだ後に文面を読むと、どの場面で効く文か理解しやすいです。

---

## 6. 追加メモ（あなたの用途向け）

AI アシスタント開発でハーネス設計を真似るなら、OpenCode から特に学ぶべきは次です。

- **Prompt layering**（provider別 + 環境 + ローカル指示 + 一時 reminder）
- **Permission as policy engine**（tool 実行前に統一 API で ask）
- **Event-sourced っぽいメッセージ分解**（assistant text / reasoning / tool を part で分離）
- **Tool contract の一元化**（description + json schema + execute + truncation）
- **モード遷移を prompt で強制**（plan/build で実行可能範囲を切り替え）
