# Instagram Story MCP

YouTube・TikTok・Instagram投稿をInstagramストーリーズへ共有するための、ローカルstdio MCPです。アカウント確認、プレビュー、親しい友達への公開、既存ハイライトへの追加、処理結果の照会を提供します。

URLの取得・動画変換・MCPをこのパッケージにまとめています。Instagramへの通信、認証、公開の検証と投稿台帳には、別に配置した [Instagram HTTP Relay](https://github.com/shirafukayayoi/instagram_api_safe_relay) を使います。普段の操作でブラウザーは起動しません。

## 必要な環境

- Python 3.11以降
- Node.js 24、pnpm 11.5.0
- `ffmpeg` と `ffprobe` がPATHから実行できること
- HTTPモードで設定・ビルド済みのRelay

Python用のMCP SDK、yt-dlp、curl-cffi、Pillowはインストール時に導入します。MCP SDKは検証済みの1.26.0へ固定しています。

## 導入

```powershell
git clone https://github.com/shirafukayayoi/instagram-story-mcp.git
cd instagram-story-mcp
python -m venv .venv
.venv\Scripts\python.exe -m pip install .
```

Linuxでは `.venv/Scripts/python.exe` を `.venv/bin/python` に読み替えてください。開発時は `pip install -e .` を使えます。

Relayは `feat/account-switching` ブランチの `e2c4b9c` 以降を利用してください。この版には文面なし動画のモバイル形式での公開と、投稿時の本人コメントの照合が含まれます。

```powershell
git clone --branch feat/account-switching https://github.com/shirafukayayoi/instagram_api_safe_relay.git
cd instagram_api_safe_relay
pnpm install --frozen-lockfile
pnpm build
```

Relayの `settings.json`、`.env`、専用のセッションをローカルで設定します。[HTTPモードの導入手順](https://github.com/shirafukayayoi/instagram_api_safe_relay/blob/feat/account-switching/docs/http-mode.md)を参照してください。

- 全プロファイルの `browser.type` を `http` にする。
- `stories.enabled` を有効にし、Relayの `work/story-smoke` を素材許可ディレクトリへ含める。
- Instagram投稿のカード共有を使う場合は `stories.feedPostSharingEnabled` も有効にする。
- HTTPモードではDMとメンション受付を無効にする。
- Relayの `.env` へ `INSTAGRAM_RELAY_API_TOKEN` を保存する。

初回ログイン・追加認証は手動で行います。既存のセッションや台帳を削除しません。認証ファイルはこのMCPリポジトリへコピーせず、Relay側のGit管理外に保持します。

## Codexへの登録

MCPリポジトリのルートから、配置先を実環境の絶対パスに置き換えて実行します。

```powershell
$mcpPython = (Resolve-Path '.venv\Scripts\python.exe').Path
codex mcp add instagram --env 'INSTAGRAM_RELAY_REPOSITORY=C:\path\instagram_api_safe_relay' --env PYTHONUTF8=1 -- $mcpPython -m instagram_story_mcp
```

`~/.codex/config.toml` の登録にタイムアウトを指定します。BearerやCookieはここへ書きません。

```toml
[mcp_servers.instagram]
command = 'C:\path\instagram-story-mcp\.venv\Scripts\python.exe'
args = ["-m", "instagram_story_mcp"]
startup_timeout_sec = 30
tool_timeout_sec = 600
env = { INSTAGRAM_RELAY_REPOSITORY = 'C:\path\instagram_api_safe_relay', PYTHONUTF8 = "1" }
```

MCPクライアントを再接続し、`instagram_status` で `ready: true`、選択中のアカウントを確認します。停止中のRelayは必要時に起動し、稼働中ならそのまま利用します。配置先の環境変数が未設定でもツール一覧は取得でき、実際の操作は設定を求めて停止します。

## ツール

| ツール | 処理 |
| --- | --- |
| `instagram_status` | 接続・必要なツール・選択中アカウントの診断 |
| `instagram_accounts` | 登録済みアカウントと状態の一覧 |
| `instagram_select_account` | 実ログイン先を照合して今後の投稿先を選択 |
| `instagram_preview_url` | URLの素材を準備し、投稿先・公開範囲をプレビュー |
| `instagram_publish_operation` | プレビュー済みの同じ操作を公開 |
| `instagram_operation_status` | 投稿とハイライト処理の状態を照会 |
| `instagram_highlights` | 既存ハイライトを一覧取得 |
| `instagram_resume_highlight` | 初回指定済みの動画のハイライト登録を再開 |

## 投稿の流れ

1. 接続と選択中のアカウントを確認する。
2. `instagram_preview_url` へURLと公開範囲を渡す。親しい友達は `close_friends`、通常公開は `all`。コメント、動画の切り出し、ハイライトは初回から指定する。
3. プレビューの投稿先、公開範囲、素材を確認する。プレビューではInstagramへアップロード・公開しない。
4. ユーザーの明示的な投稿依頼がある場合に、返された `operationId`、`preview.account.id`、`preview.audience` を `instagram_publish_operation` へ渡す。
5. 投稿は `result.state: published` とメディアID、アカウント、公開範囲を確認する。ハイライトは別に `highlightResult.state: completed` を確認する。

楽曲紹介のコメントは、確認できた「曲名 / アーティスト名」を基本にします。投稿者名からアーティストを推定しません。会話動画などへのコメントは任意です。文面は投稿時の本人コメントとして送り、画像・動画へ文字を焼き込みません。

動画はH.264/AACまたは音声なしのMP4、1〜60秒、100MiB以下が対象です。横動画は縦のキャンバスへ変換し、原本を保存します。勝手な音声差し替えや自動分割は行いません。60秒を超える動画は、ユーザーが指定した範囲を切り出します。

## 再開と結果不明

操作IDを省略した初回プレビューではUUIDを作ります。再開時は同じIDと引数を指定します。プレビューの有効期限は10分で、期限切れは同じ操作IDで更新します。

公開要求は確認済みプレビューのIDで1回だけ送り、送信前に台帳へ記録します。受理済み・公開済み・結果不明の操作で公開ツールを再び呼んだ場合は状態照会だけを行います。接続が切れても、新しいIDで自動再投稿しません。

ハイライトの `unknown` も再追加しません。対象の動画がハイライトに存在しても、登録前後の既存一覧の維持を確認できなければ、厳密な完了状態は `unknown` のまま保持します。投稿とハイライトの成否を分けて報告してください。

子プロセスの標準入力・出力・エラーをMCPの通信パイプから分離しています。Windowsで子Pythonが起動時に停止する問題の回帰テストを含みます。子プロセスの生出力や、署名付き素材URLをMCPへ返しません。

## テスト

インストール後、ネットワークや実アカウントを使わないテストを実行できます。

```powershell
.venv\Scripts\python.exe tests/test_server.py
.venv\Scripts\python.exe -m pip wheel --no-deps . --wheel-dir dist
```

設定済みRelayの接続確認だけを行う場合：

```powershell
$env:INSTAGRAM_RELAY_REPOSITORY = 'C:\path\instagram_api_safe_relay'
.venv\Scripts\python.exe tests/test_server.py --live
```

`--live` でも投稿やハイライト追加は行いません。CIはLinuxのPython 3.11/3.14とWindowsのPython 3.14で、境界テスト、stdio接続、wheelビルドを実行します。

## ライセンスと元の実装

MIT License。元の実装と変更点は [NOTICE.md](NOTICE.md) を参照してください。
