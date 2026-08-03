# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [Unreleased]

### Security

- **bandit B101 解消**: `cli.py` の `assert isinstance(...)` を `TypeError` を送出する明示的な型ガードに置き換え (`_get_llm_provider`, `_create_source`)
- **bandit B105 誤検知を抑制**: `pii.py` の `_TOKEN_MASK` は regex バックリファレンス文字列であることを `# nosec B105` コメントで明示
- **`.gitignore` 強化**: `*.p8`, `*.pem`, `*.key`, `service-account*.json` などの秘密鍵ファイルパターンを追加。コメント付きでセクション分け

### Changed

- **依存バージョン更新**: `jinja2>=3.1.6` に引き上げ (CVE 対応: PYSEC-2026-1471, PYSEC-2026-1475)
- **`pyproject.toml`**: Development Status を `Alpha` → `Beta` に変更

---

## [0.2.0] — 2026-06-21

### Added

- **`appreview export` コマンド**: レビューを CSV 形式でダンプ (`--app`, `--since`, `--output` オプション)
- **`--since` に週指定 `2w` を追加**: `_parse_since` が `w` サフィックスをサポート
- **OpenAI 新モデル対応**: GPT-4.1 / 4.1-mini / 4.1-nano / o3 / o4-mini をコストテーブルに追加
- **Anthropic 新モデル対応**: `claude-3-7-sonnet-20250219` をコストテーブルに追加
- **Google Play 指数バックオフリトライ**: `MAX_RETRIES=5`、`Retry-After` ヘッダー対応、`429` を `RateLimitError` に正しくマッピング
- **言語検出拡張**: タイ語 (th) / インドネシア語 (id) / ベトナム語 (vi) をデフォルト対応言語に追加
- **`get_reviews_by_rating()`**: 評価スコアで範囲フィルタするクエリを `ReviewRepository` に追加
- **テスト追加**: `test_cli_utils.py` (11件, `_parse_since` 全フォーマット + エラーケース)、`test_cost.py` (+7件, GPT-4.1 / Claude-3.7 / o3-mini)

### Changed

- **バージョン**: 0.1.0 → 0.2.0
- **コスト計算**: プレフィックスマッチを `startswith` のみに簡略化、デフォルトフォールバックを `gpt-4.1-mini` に変更

### Fixed

- **Google Play 429 処理**: `429` レスポンスが `RateLimitError` を正しく送出するよう修正

---

## [0.1.1] — 2026-06-09

### Changed

- **Lint 全解消**: `ruff` 129件の自動修正 + 26件の手動修正で全違反をクリア
  - `UP017` — `timezone.utc` → `datetime.UTC` エイリアスに統一
  - `UP035` — `typing.AsyncGenerator/AsyncIterator/Sequence` → `collections.abc` へ移行
  - `UP045` — `Optional[X]` → `X | None` 形式に統一
  - `UP037` — 文字列型アノテーションからクォートを除去
  - `I001` — import ブロックのソート整理
  - `B007` — 未使用ループ変数を `_` プレフィックスに変更
  - `B905` — `zip()` に `strict=False` を明示
  - `RUF100` — 不要な `noqa` コメントを除去
  - `F841` — 未使用変数 (`log`, `total_chars`, `since`, `env`) を除去
  - `E501` — 100文字超の行を全て折り返し
  - `ANN401` — `Any` 型シグネチャを具体型に改善 (report 関数の `app_config: AppConfig`, `google_play.py` の `_get_credentials`)

- **`repository.py` 最適化**
  - `count_reviews()`: 全行フェッチ → `SELECT COUNT(*)` に変更（大量データで O(1) に）
  - `get_unclassified_reviews()`: classified ID を Python セットで保持する方式 → SQL `NOT EXISTS` 相関サブクエリに変更（メモリ効率向上）

- **`cli.py` 改善**
  - `_run_cost_estimate()` の `load_env_settings` 未インポートバグを修正
  - `_run_fetch()` の未使用 `log = get_logger(...)` を除去

---

## [0.1.0] — 2026-06-03

### Added

- **CLI**: `appreview init`, `doctor`, `fetch`, `analyze`, `run`, `report`, `cost-estimate`, `list-runs` コマンド
- **App Store Connect API**: JWT (ES256) 認証、ページネーション、差分取得、Exponential Backoff
- **Google Play Developer API**: Service Account認証、Token Bucket レートリミッタ（200 req/hour）、7日制限の自動クランプ
- **LLMプロバイダ**: OpenAI (structured outputs), Anthropic (tool use), Ollama (ローカル)
- **分類パイプライン**: バッチ処理（最大20件）、カテゴリ別分類、信頼度スコア
- **クラスタリング**: ネガティブレビューのカテゴリ内グループ化、改善提案生成
- **PIIマスキング**: メール・電話番号・クレカ番号・URLトークンの自動マスク
- **言語検出**: lingua-language-detector による多言語対応
- **SQLiteストレージ**: 差分実行対応、実行履歴管理
- **レポート生成**: Markdown と JSON 形式
- **コスト管理**: 実行前コスト見積もり、上限超過時の確認プロンプト
- **ドキュメント**: README, setup guides, API key 取得手順

### Notes

- Google Play APIは過去7日分のみ取得可能。日次実行を推奨。
- PyPI公開は未実施（コードとパッケージング設定のみ）。

---

## [Future]

v0.3 以降での実装を検討中:

- Slack / Discord 通知連携
- GitHub Issues 自動起票
- Web UI (Streamlit / FastAPI)
- 競合アプリレビューとの比較分析
- リアルタイム監視（Webhook対応）
- Docker image 配布
- PyPI 公式公開
- 多言語UI（現在はプロンプト内応答言語制御のみ）
- レビュースコア時系列グラフ（matplotlibまたはmermaid）

