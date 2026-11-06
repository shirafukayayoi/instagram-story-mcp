# ソースと帰属

MIT Licenseの著作権表示は `LICENSE` を参照してください。

このMCPは [shirafukayayoi/instagram_api_safe_relay](https://github.com/shirafukayayoi/instagram_api_safe_relay) の `e2c4b9c0edbe518d3cdc9ac798463ea97fd581ba` から分離しました。

- `src/instagram_story_mcp/server.py`：元の `scripts/instagram_mcp.py` を、独立パッケージと明示的なRelay配置先の指定に対応させています。
- `src/instagram_story_mcp/workflow/`：元の `skills/instagram-story-from-url/scripts/` にある4つのURL処理モジュールを含みます。`relay_runtime.py` の配置先解決は、このパッケージ向けに変更しています。
- `tests/test_server.py`：元のMCPテストをパッケージ構成に対応させ、配置先未設定時のテストを追加しています。

元のRelayは [fa0311/twitter_api_safe_relay](https://github.com/fa0311/twitter_api_safe_relay) の構造と実装方針を参考にしています。元の著作権表示を保持しています。
