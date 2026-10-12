# 全体テーマ：グローバルな設定状態によるdata raceと既存APIの扱い

- 作成日：2026-10-12
- 対象OSS：tidwall/gjson
- 観察予定のコミットSHA：690362d6edf4bcbde1e9f54d552d3814f1cd5bcb
- 対象バージョン：未確認（観察時に確認）

## 分割前の問い

gjsonは、グローバルな設定状態によるdata raceをどのように解消し、既存APIとの互換性をどのように維持したか？

この問いを、処理の流れから順に読める小さな問いに分けた。最初に選ぶ問いと所要時間は [対象のREADME](../README.md) で管理し、観察結果は問いごとの文書に記録する。

## 分けた問い

| 番号 | 問い |
| --- | --- |
| 01 | [SetEscapeHTMLを呼ぶと、設定値はどこに保存され、どこで読まれるか？](20261012-01-escape-html-state-flow.md) |
| 02 | [AddModifierで登録した関数は、どこに保存され、どこから呼ばれるか？](20261012-02-modifier-registration-flow.md) |
| 03 | [変更前の共有状態では、どの読み取りと書き込みが競合し得るか？](20261012-03-conflicting-state-access.md) |
| 04 | [atomic.Boolとsync.RWMutexは、それぞれどの状態の読み書きを保護しているか？](20261012-04-synchronization-scope.md) |
| 05 | [Deprecatedとされた既存APIは、変更後にどのように扱われるか？](20261012-05-deprecated-api-behavior.md) |
| 06 | [設定またはmodifierに関するテストを1つ選ぶと、どんな入力と期待値を確認しているか？](20261012-06-test-input-and-expectation.md) |

初回は1つを選ぶ。残りは再訪時の候補として残せる。この文書は全体テーマの案内であり、所要時間は各問いの観察メモに記録する。

## 調べる前の仮説

元の計画にあった仮説を残す。いずれも観察前の予想であり、未検証。

- 問題の中心は、複数goroutineから読み書きされるパッケージレベルの可変状態にあるのではないか。
- 既存シンボルをDeprecatedとして残し、新しい安全なAPIへの移行を案内しているのではないか。名前と動作の互換性は、それぞれ確認する。
- 真偽値の設定と、動的に追加されるmodifierでは性質が異なるため、atomicとmutexを使い分けているのではないか。

## 参照候補

内容と各問いとの関係は、観察時に確認する。

- [対象コミット](https://github.com/tidwall/gjson/commit/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb)
- [gjson.go](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go)
- [gjson_test.go](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson_test.go)
- [Issue #311](https://github.com/tidwall/gjson/issues/311)
- [Issue #387](https://github.com/tidwall/gjson/issues/387)
- [Issue #401](https://github.com/tidwall/gjson/issues/401)

パーサ全体、クエリ構文、高速化手法などに疑問が出たら、次の問いとして記録する。
