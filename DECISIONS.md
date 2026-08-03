# DECISIONS.md — appreview-insight 実装判断ログ

このファイルは仕様書 §16「判断指針」に従い、実装AIが仕様の曖昧な点で判断した内容を時系列で記録する。

---

## 2026-06-03: 実装開始

**背景:** 仕様書 v1.0 に基づいてゼロから実装を開始。

**実装環境:**
- Python 3.13.13（仕様書では 3.11+ を前提としているが、3.13 は後方互換あり。型ヒント等は 3.11 互換で記述）
- uv 0.11.18（パッケージ管理に採用）
- OS: Linux (sandbox)

**使用するパッケージマネージャ:** uv
- 理由: hatch より高速で、`pyproject.toml` の PEP 621 形式に素直に対応。`uv pip install -e .` でローカル開発が容易。

---

## 2026-06-03: ロギングライブラリの選択

**背景:** 仕様書 §14 では `structlog` または標準 `logging` から選択し記録するよう指示あり。

**検討した選択肢:**
1. 標準 `logging` — 追加依存なし、シンプル
2. `structlog` — JSON Lines 対応が容易、構造化ログに優れる

**決定:** `structlog` を採用

**根拠:** `--log-format json` オプションのサポートが仕様要件にあり、structlog は JSON Lines 出力への切り替えが設定1行で済む。依存追加のコストより実装の明瞭さを優先（仕様書 §16 第五指針）。

---

## 2026-06-03: SQLAlchemy async vs sync の選択

**背景:** 仕様書では SQLAlchemy 2.0+ (async) と明記されているが、SQLite の async ドライバ（aiosqlite）が必要。

**決定:** `aiosqlite` を依存に追加

**根拠:** 仕様書で async SQLAlchemy が明記されており、aiosqlite はそのデファクト。追加する理由を DECISIONS.md に記録（仕様書 §3 の指示に従う）。

---

## 2026-06-03: Alembic非採用・migrations.py 管理

**背景:** 仕様書 §5.2 で「Alembicは使わず、`migrations.py` 内で起動時に冪等にCREATE TABLE IF NOT EXISTSで管理する」と明記。

**決定:** 仕様書通り。v0.1.0 では `migrations.py` のみで管理。

---

## 2026-06-03: Jinja2 の依存追加

**背景:** 仕様書 §7.3 でプロンプトテンプレートに Jinja2 を使用すると明記。

**決定:** `jinja2` を依存に追加。

**根拠:** 仕様書で明示指示あり。

---

## 2026-06-03: httpx async 使用時の google-auth との統合

**背景:** 仕様書では httpx（async）で HTTP 通信を行うが、google-auth は requests ベースが標準。

**決定:** Google Auth の token refresh 部分は google-auth の `Credentials.refresh()` を `asyncio.to_thread()` でラップして async コンテキストで呼び出す。

**根拠:** google-auth-httplib2 は同期前提。httpx との統合のために `google-auth-httplib2` は使わず、トークン取得のみ同期→スレッドで実行する方式が最もシンプルで testable（§16 第四指針）。

---

## 2026-06-03: lingua-language-detector のモデルサイズ

**背景:** lingua は low/high accuracy モードがあり、依存サイズが大きく異なる。

**決定:** `LanguageDetectorBuilder.from_all_languages().build()` ではなく、主要言語（日英中韓独仏西伊葡）のみを対象とした軽量ビルドを採用。ユーザーが設定で言語セットを拡張できるようにする。

**根拠:** モバイルアプリのレビューは限られた言語に集中する。全言語モデルは起動時間・メモリを大きく消費する（§15 パフォーマンス要件）。

---

## 2026-06-03: openai SDK バージョンと responses.create vs chat.completions.create

**背景:** 仕様書で「`responses.create` または `chat.completions.create` の structured outputs」と記載。

**決定:** `chat.completions.create` + `response_format={"type": "json_schema", ...}` を採用。

**根拠:** `responses.create` は OpenAI の新 Responses API で、現時点での SDK 互換性が安定的。`chat.completions.create` の structured outputs は widely available で、テスト・モックが容易（§16 第四指針）。

---

## 2026-06-03: asyncio event loop の管理

**背景:** Typer は同期関数ベース。async 処理を CLI から呼ぶために asyncio.run() が必要。

**決定:** 各 CLI コマンド関数は同期で、内部で `asyncio.run()` を呼ぶラッパーパターンを採用。

**根拠:** anyio/asyncer の追加より標準ライブラリのみで解決する方が依存が少ない（§16 第五指針）。

---

## 2026-08-03: セキュリティ監査・強化

**背景:** `bandit` による静的解析と `pip-audit` による依存パッケージ脆弱性チェックを実施。

**実施内容と判断:**

1. **シークレット漏洩チェック（クリア）**
   - git 履歴・全ファイルを対象にスキャン（`*.p8`, `*.pem`, `*.key`, `.env`, `secrets/`）
   - `sk-`, `AKIA`, `ghp_`, `AIza`, `-----BEGIN` パターンを全ファイルで検索
   - 漏洩なし。`README.md` の `sk-...` は例示文字列のみ

2. **bandit B101 (assert 使用) → 型ガードに置換**
   - `cli.py` の `_get_llm_provider()` と `_create_source()` 内の `assert isinstance(...)` を `if not isinstance(...): raise TypeError(...)` に変更
   - 根拠: `assert` は `-O` オプション（最適化ビルド）で除去されるため、型安全の保証にならない

3. **bandit B105 (ハードコードパスワード誤検知) → nosec コメント**
   - `pii.py` の `_TOKEN_MASK = r"\1[REDACTED_TOKEN]"` は regex のバックリファレンス置換文字列であり、パスワードではない
   - `# nosec B105` コメントを追加して意図を明示（§16 第三指針: 誤検知の抑制には根拠を記録）

4. **`.gitignore` 強化**
   - `*.p8` (Apple Auth Key), `*.pem`, `*.key`, `service-account*.json` を追加
   - セクションコメントで「絶対コミットしない」旨を明記

5. **依存パッケージの脆弱性対応**
   - `jinja2`: `>=3.1` → `>=3.1.6` に引き上げ (PYSEC-2026-1471: Server-Side Template Injection, PYSEC-2026-1475)
   - `aiohttp` の CVE は本プロジェクトの直接依存ではないためスコープ外。将来的に依存が追加された場合は `>=3.14.1` を指定すること

