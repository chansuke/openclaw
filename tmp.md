# レビュー結果（2026-02-21）

`tmp.md` の5項目を実コードで検証し、必要な修正を反映済み。

## 1. MS Teams: 4MB 超ファイルの Resumable Upload Session 対応

**状態:** ✅ 実装済み  
**反映先:** `extensions/msteams/src/graph-upload.ts`

- `uploadToOneDrive` / `uploadToSharePoint` を共通化し、`4MB` を境に simple upload と resumable upload を自動分岐
- `POST .../createUploadSession` で `uploadUrl` を取得し、`5MB`（320KB の倍数）チャンクで `PUT` 送信
- 最終チャンクレスポンスから `id`, `webUrl`, `name` を検証して返却
- OneDrive (`/me/drive`) と SharePoint (`/sites/{siteId}/drive`) の両方を同一ロジックで対応

---

## 2. Nostr: `dispatchReplyWithBufferedBlockDispatcher` への置き換え

**状態:** ✅ 実装済み  
**反映先:** `extensions/nostr/src/channel.ts`

- 非公式 `handleInboundMessage` キャスト呼び出しを削除
- `resolveAgentRoute` → `finalizeInboundContext` → `recordInboundSession` → `dispatchReplyWithBufferedBlockDispatcher` の正式経路に移行
- `createReplyPrefixOptions` を利用した reply prefix / model selected ハンドリングを追加
- `ctx` は `channel/provider/sender/chat` を Nostr DM に合わせて明示設定

**補足（元メモの修正点）:**

- 元メモにある `createCtxPayload` / `buildSessionTracker` は現行 plugin-sdk には存在せず、実装では `finalizeInboundContext` / `recordInboundSession` を使用するのが正しい

---

## 3. model-usage スキル: Linux CLI サポートガイドの追加

**状態:** ⏳ 未対応（要前提確認）  
**対象:** `skills/model-usage/SKILL.md`

- 現在の frontmatter は `"os": ["darwin"]` で、本文にも Linux 対応は TODO のまま
- Linux バイナリ配布経路（公式 install path）が確認できてから `os` / `install` / 本文手順を更新するのが妥当

---

## 4. skill-creator: テンプレートスクリプトの実ロジック実装

**状態:** ✅ 実装済み  
**反映先:** `skills/skill-creator/scripts/init_skill.py`

- `EXAMPLE_SCRIPT` を実用的な CLI 雛形に更新
- `argparse` による `--input` / `--output` / `--format(text|json)` を追加
- stdin/ファイル入力、stdout/ファイル出力、例外時 `stderr` + 非0終了を追加

---

## 5. MS Teams: SharePoint アップロードのエラーリカバリ強化

**状態:** ✅ 実装済み  
**反映先:** `extensions/msteams/src/graph-upload.ts`

- 429/503/504 と一時的 fetch 失敗を対象に再試行ラッパーを追加
- `Retry-After` ヘッダー尊重 + 指数バックオフ（1s/2s/4s）
- 適用対象:
  - upload（simple/resumable chunk）
  - `createUploadSession`
  - `createSharingLink`
  - `createSharePointSharingLink`
  - `getDriveItemProperties`
  - `getChatMembers`
- リトライログを `console.warn` で出力

---

## 追加テスト

- `extensions/msteams/src/graph-upload.test.ts` を追加
  - simple/resumable 分岐
  - 429 リトライ
- `extensions/nostr/src/channel.dispatch.test.ts` を追加
  - Nostr inbound が正式 dispatcher 経由で配信されること
