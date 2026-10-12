# 全体テーマ：共有状態と既存APIの扱い

グローバルな設定状態によるdata raceをどのように解消し、既存APIへの影響をどう扱ったか？

## 分けた問い

- [HTMLエスケープの設定は、処理にどう反映されるか？](20261012-01-escape-html-state-flow.md)
- [登録したmodifierは、どのように呼び出されるか？](20261012-02-modifier-registration-flow.md)
- [変更前の共有状態では、どの操作が競合し得るか？](20261012-03-conflicting-state-access.md)
- [共有状態の読み書きは、どの範囲で同期されるか？](20261012-04-synchronization-scope.md)
- [既存APIは、変更後にどのように扱われるか？](20261012-05-deprecated-api-behavior.md)
- [追った処理に関連するテストは、何を確かめているか？](20261012-06-test-input-and-expectation.md)

[問いの候補一覧](../README.md#問いの候補)
