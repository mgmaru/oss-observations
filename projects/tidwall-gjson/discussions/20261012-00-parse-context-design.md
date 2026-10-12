# 議論：parseContextを呼び出し先で書き換える設計は良いか

- 日付：2026-10-12
- きっかけ：[問い00の観察メモ](../observations/20261012-00-basic-get-flow.md)に書いた、parseContextについての疑問
- 関連：[問い00のレビュー](../reviews/20261012-00-basic-get-flow.md)
- 照合したコード：`gjson.go`（コミット `690362d6edf4bcbde1e9f54d552d3814f1cd5bcb`、タグ `v1.20.0`）
- 議論の相手：Claude（AI）

「AIの考え」はAIによる説明で、観察メモで読んだ範囲（`Get`）の外の話を含む。[運用ルール](../../../RULES.md)の「6. 観察の正確さ」に従い、「自分で確かめること」で確認するまでは、観察メモに事実として書かない。

## 自分の疑問と意見

> 確かに、どちらも直後に break するので、1回のGetで呼ばれるのは、parseObjectまたはparseArrayのどちらかです。
> しかし、この書き方は個人的に気に食わないです。なぜなら、変数cのフィールドの内部がどのように書き換えられているのかが見えないからです。また、cの内部をバラバラに書き換えているところもよくわかりません。
> また、そのせいか、Getの関数の最後にc.valueで返していますが、いつcのvalueに値が書き込まれたのかがこのコードからはわかりません。これで良いのでしょうか？個人的には、以下のように書くかと思います。
>
> ```go
> c.pipe, c.piped = parseArray(c, i, path)
> ```
>
> このように書くことで、parseArrayの中でparseContextのフィールドの値を書き換えるのではなく、呼び出し元で書き換わっているのがわかります。直前の`c.lines = true`でもGetの中で変数cのフィールドを書き換えています。

## AIの考え

### 結論

「`c` の中身がいつ、どこで書き換わるのかが見えない」という違和感はもっともで、コードにも裏付けがある。ただし、提案の書き方はそのままでは成り立たない。「結果の書き込み先を渡す」設計自体は妥当で、問題は「何が出力で、どこで書かれるか」をコードで示していない点にある。

### 違和感を裏付ける事実

| フィールド | 役割 | 書いている場所 | 読んでいる場所 |
| --- | --- | --- | --- |
| `json` | 入力 | `Get` [2261行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L2261) | parse関数 |
| `lines` | 入力（設定） | `Get` [2263行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L2263) | どこからも読まれていない |
| `value` | 出力（結果） | `parseObject` [1321-1385行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L1321-L1385)、`parseArray` [1610-1843行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L1610-L1843)の約30か所 | `Get` [2280・2285行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L2279-L2285)、`fillIndex` |
| `pipe` / `piped` | 出力（後処理の指示） | `parseObject` [1237行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L1236-L1239)、`parseArray` 1562・1590・1760行目 | `Get` 2279-2280行目 |
| `calcd` | 出力（フラグ） | `parseArray` 1808行目 | `fillIndex` [3442行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L3441-L3450) |

- 入力と出力が1つの構造体に混ざっている。`c.lines = true` は入力側、`value` や `pipe` は出力側の書き換えで、役割が違う。
- `value` はフィールドごとに分けて書かれている（例：1321-1326行目の `c.value.Str`、`c.value.Raw`、`c.value.Type`）。いつ書かれるかを知るには、2つの関数の約30か所を読む必要がある。
- 値が見つからなければ `c.value` はゼロ値のままで、`Exists()`（694-696行目）が false を返す。「見つからない」という結果も、コード上では見えない形で表現されている。
- `c.lines` は2263行目で書かれるが、`gjson.go` と `gjson_test.go` のどこからも読まれていない。中身の見えない状態は、使われなくなったフィールドが残っても気づきにくい。

読み方のヒント：`Get` の2264・2269・2274行目では parse関数の戻り値をすべて捨てているのに、後で `c.value` を使っている。戻り値を捨てているのに、引数に渡したポインタのフィールドを後で使っていたら、その関数がそこに書き込んでいると考えられる。

### 提案の書き方が成り立たない理由

- `parseArray` の戻り値 `(int, bool)` は「次に読む位置」と「値が見つかったか」で、`pipe` の情報ではない。この2つは再帰で欠かせない（例：1331行目の `i, hit = parseObject(c, i+1, rp.path)`）。
- 結果が決まるのは再帰の一番深いところ（`"user.name"` なら `name` を探している階層）。戻り値で `Get` まで届けるには、途中のすべての階層が結果を受け取って返し直す必要がある。返す値には `value`・`pipe`・`piped`・`calcd` が必要で、結局 `parseContext` の出力部分と同じ構造体になる。

  ```go
  // 戻り値で返す場合（再帰呼び出しの4か所すべてで必要）
  i, hit, res = parseObject(json, i+1, rp.path)
  if hit {
      return i, true, res
  }
  ```

- `c` を渡している限り、呼び出し先で書き換えることは防げない。呼び出し元でだけ書き換わることを保証するには、`c` を渡すのをやめる必要がある。

### なぜこう書いたのか（推測）

作者が理由を書いた記録は見つけていない。

- 戻り値では制御に必要な「位置」と「見つかったか」だけを返し、最終結果は書き込み先に直接置く。こうすると、途中の階層は `hit` を返すだけで済む。これが一番の理由だと考える。
- gjsonのREADMEは「fast」を掲げ、`Get` のベンチマークは「0 allocs/op」。ただ、`Result`（64ビット環境で約80バイト）を戻り値で返しても割り当ては増えないはずで、違いは各階層でのコピー程度。測定していないので、性能が決め手とは言えない。
- Goの標準ライブラリにも同じ形がある。`encoding/json` の `decodeState.unmarshal` は、`d.value(rv)` を呼んだ後、深いところで書かれた `d.savedError` を返す（[decode.go 179-194行目](https://github.com/golang/go/blob/2c3c1caad2079458fa7248ea1814797fc2e94e8a/src/encoding/json/decode.go#L179-L194)、2026-10-12閲覧）。

### 代わりの案

「書き込み先を渡す」設計は残し、性能を変えずに、見えない部分を減らす。

```go
// parseContext は parseObject / parseArray の作業領域。
type parseContext struct {
	json string // 入力：検索対象のJSON

	// 出力：parseObject / parseArray が書き込み、Get が読む
	value Result // 見つかった値。見つからなければゼロ値のまま
	pipe  string // パイプ以降のパス（piped が true のとき）
	piped bool
	calcd bool // value が元のJSONの一部ではなく、計算で作った値か
}
```

1. 入力と出力をコメントで分け、使われていない `lines` を消す。
2. 出力はフィールドごとではなく、構造体ごと代入する（例：`c.value = Result{Type: String, Raw: val, Str: s}`）。`c.value =` を検索するだけで、書き込んでいる箇所がすべて見つかる。ただし、ほかのフィールドがゼロに戻ることで動作が変わらないかは、1か所ずつ確かめる必要がある。
3. `Get` の呼び出し箇所に、「結果は `c.value` に入る」とコメントを書く。

「呼び出し元で書き換えが見えるようにしたい」という感覚は、ふだんのアプリケーションのコードでは正しい原則。戻り値で返すほうが、テストもしやすく追いやすい。書き込み先を渡す形を選ぶのは、深い再帰や何層もの呼び出しの奥で複数の出力が決まるときだけでよい。その場合も、何が出力なのかを型やコメントで明示する。

## 自分で確かめること

新しい範囲のコードを読むので、再訪のときに確かめる。確かめたら `[x]` にし、結果を再訪メモに書く。

- [ ] parseContextの各フィールドは入力か出力か。どこで書かれ、どこで読まれるか
- [ ] `c.lines` は書かれるだけで読まれていないか。そうなら、いつからか（`git log -S "c.lines"`）
- [ ] 戻り値を `(int, bool)` にしているのは、再帰で「位置」と「見つかったか」を使うためか（1331行目など）
- [ ] `encoding/json` の `decodeState` も同じ形か

## 自分の結論

議論を読んで、納得したこと・納得できないことを自分の言葉で書く。書けたら、観察メモの「自分の設計に活かせること」に移す。

- 納得したこと：
- 納得できないこと・まだ分からないこと：
- 自分の設計に活かせること：
