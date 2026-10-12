# oss-observations

OSSのコード・公開API・ドキュメント・設計判断・保守状況を観察し、自分の設計に活かせる学びを残す個人用リポジトリ。

時間を区切って複数のOSSに触れ、共通の問いで比較する。1回の観察では問いを1つに絞り、関連する実装やテストを根拠まで確かめる。

## 運用の基本

- 初回の観察は、1つのOSSにつき累計90分まで。準備・環境構築・実験・記録を含める。
- ブラウザから観察を始める。クローンは、検索・実行・変更などで必要になった場合に行う。
- 観察時点は、日付・バージョン・コミットSHA・参照箇所の固定リンクで残す。
- 初回観察を終えた異なるOSSが4つ増えるごとに、再訪枠を1つ使える。再訪は追加90分まで、同じOSSの合計は180分まで。
- 時間になったら、分かったこと・根拠・未確認のことを書いて終了する。

詳しくは [RULES.md](RULES.md) を読む。

## 観察対象と記録

[観察対象の一覧](projects/README.md)に、対象・所要時間・状態・記録へのリンクをまとめる。

対象ごとのディレクトリ名は、上流リポジトリの `owner-repo` 形式にする。例えば `golang/go` は `projects/golang-go/`。同じリポジトリの別パッケージや別バージョンを観察しても、時間は同じ対象に合算する。

```text
oss-observations/
├── README.md
├── RULES.md
├── docs/
│   └── oss_code_reading_guide.md # コードの読み方のガイド
├── templates/
│   ├── project.md             # 対象ごとの案内
│   ├── observation.md         # 1つの問いの観察記録
│   └── experiment.md          # 実験の目的・実行方法・結果
└── projects/
    ├── README.md             # 観察対象・時間・再訪枠の一覧
    └── <owner>-<repo>/        # 対象を決めたら作る
        ├── README.md
        ├── observations/
        │   └── YYYYMMDD-topic.md
        └── experiments/      # 実験が必要になったら作る
            └── <topic>/
                ├── README.md
                └── 実験コード
```

## 観察を始める

コードを読む手順は [コードリーディングガイド](docs/oss_code_reading_guide.md) を参照する。

例えば `golang/go` を対象にする場合、次のように用意する。観察対象のソースコードをクローンする必要はない。

```sh
mkdir -p projects/golang-go/observations
cp templates/project.md projects/golang-go/README.md
cp templates/observation.md projects/golang-go/observations/YYYYMMDD-topic.md
```

日付・テーマに合わせてファイル名を変え、対象の情報と今回の問いを記入する。[観察対象の一覧](projects/README.md)にも対象を登録する。

問いは「タイムアウト設定をどの部品が持つか」「公開APIと内部実装の境界を何で守っているか」「実装固有のエラーをどこで翻訳するか」など、調べる範囲が分かるものにする。

観察が終わったら、メモ・対象ごとのREADME・一覧に時間と状態を反映する。次のOSSでは、前の対象と比べられる問いを選ぶ。

## 記録の考え方

記録は、問い → 仮説 → 調べた場所 → 確認できた事実 → 自分の設計に活かせること → 次の問い、の順に残す。

作者の意図は、issue・PR・設計文書などの根拠と結び付ける。自分の推測は、そのように明記する。短時間で読んだ範囲と未確認の範囲も残す。

実験の手順と結果には [実験テンプレート](templates/experiment.md)を使える。クローンする場合の作業場所は `_local/<owner>-<repo>/` とし、Git管理から除外する。
