# 議論：parseContextを呼び出し先で書き換える設計は良いか

- 日付：2026-10-12
- きっかけ：[問い00の観察メモ](../observations/20261012-00-basic-get-flow.md)に書いた、parseContextについての疑問
- 関連：[問い00のレビュー](../reviews/20261012-00-basic-get-flow.md)
- 照合したコード：`gjson.go`（コミット `690362d6edf4bcbde1e9f54d552d3814f1cd5bcb`、タグ `v1.20.0`）
- 議論の相手：Claude（AI）

「AIの考え」はAIによる説明で、観察メモで読んだ範囲（`Get`）の外の話を含みます。[運用ルール](../../../RULES.md)の「6. 観察の正確さ」に従い、「自分で確かめること」で確認するまでは、観察メモに事実として書きません。

## この議論で分かること

- 関数が結果を呼び出し元に渡す方法には、「戻り値で返す」と「引数のポインタに書き込む」の2つがあります。
- ふだんのコードでは、戻り値で返すほうが追いやすく、テストもしやすくなります。
- gjsonのparse関数は、ポインタに書き込む形です。結果が深い再帰の奥で決まるので、この形が選ばれたと考えられます（推測）。ただし、どのフィールドが出力なのかがコードに示されていないため、読みにくくなっています。
- ポインタに書き込む形を使う場合も、「値を作る関数は戻り値で返し、入れ物への書き込みは1か所にまとめる」と読みやすくなります。gjsonには、そのために使える関数（`parseAny`）がすでにあります。

## 自分の疑問と意見

### 最初の疑問

> 確かに、どちらも直後に break するので、1回のGetで呼ばれるのは、parseObjectまたはparseArrayのどちらかです。
> しかし、この書き方は個人的に気に食わないです。なぜなら、変数cのフィールドの内部がどのように書き換えられているのかが見えないからです。また、cの内部をバラバラに書き換えているところもよくわかりません。
> また、そのせいか、Getの関数の最後にc.valueで返していますが、いつcのvalueに値が書き込まれたのかがこのコードからはわかりません。これで良いのでしょうか？個人的には、以下のように書くかと思います。
>
> ```go
> c.pipe, c.piped = parseArray(c, i, path)
> ```
>
> このように書くことで、parseArrayの中でparseContextのフィールドの値を書き換えるのではなく、呼び出し元で書き換わっているのがわかります。直前の`c.lines = true`でもGetの中で変数cのフィールドを書き換えています。

### 追加の疑問

#### 1. 「戻り値で返す」とは、parseContextを返すことか

> 「戻り値で返すほうが、テストもしやすく追いやすい。」とありますが、これはパース関数`parseObject`および`parseArray`の戻り値として、parseContextを返すということですか？

→ 答えは「よくある疑問」の1つ目。

#### 2. Getに返すのは、パースが終わった後の結果ではないか

> 「Bの形（戻り値で返す）にすると、結果を Get まで届けるために、途中のすべての階層が結果を受け取って返し直す必要があります。」ここがよくわかりません。
> Getに返すのは、パースの途中の結果ではなくて、パースが終わった後の結果ですよね？

→ 答えは「よくある疑問」の2つ目。説明に使った「途中の階層」という言葉が分かりにくかったため、「途中で呼ばれている関数」に言い換えた。

#### 3. Aでも、途中の関数はcを渡すだけではないか

> Bの場合がバケツリレーとなり、意味のない関数が出来上がってしまい、冗長であることは何となくわかりますが、Aに関してもlevel1の関数がlevel2にcを渡すだけのような気がします。

`level1`／`level2` は、議論の途中でAIが出した例。途中の関数（`level1`）に自分の仕事がなかったので、AもBも中継しているだけに見えた。分かりにくい例だったため、「3. 理由2」の `findUser`／`findName` の例に置き換えた。答えは「よくある疑問」の3つ目。

#### 4. 書き込みは1か所にしたほうがよいのではないか（自分の考え）

> その例で何となくわかってきました。Bだとfindnameの戻り値としてnameとfoundがありますね。
> 一方、Bだとfindnameにcを渡すだけで結果を書き換えてくれますね。
> そう考えるとBの方がシンプルな感じになるかもしれません。
> 今回はfindUser -> findNameの2段ですが、これが5段とかになった場合は、返り値が多くなり、大変なことになりそうですね。それであれば、構造体を渡してしまって、その関数に書き込みをお願いした方が良い気がします。しかし、その場合、構造体のフィールドに１箇所で書き込んだ方が良い気がしました。じゃないと、このコードではどこで構造体に書き込まれているのかがバラバラで良くわかりません。どうでしょうか？

2文目の「B」は、cを渡す形（A）のこと。3文目の「B」も、文の流れからAのことと読んだ。この考えへのAIの意見は「6. 書き込みを1か所にまとめる」。

## AIの考え

### 1. 前提：関数が結果を渡す2つの方法

関数が結果を呼び出し元に渡す方法は、大きく2つあります。JSONの文字列 `"Alice"` から、前後の `"` を外して名前を取り出す関数で比べます。

**A：引数のポインタに書き込む（gjsonと同じ形）**

```go
type ctx struct {
	name string
	ok   bool
}

// s は "Alice" のように、前後に " が付いた文字列とする
func parseName(c *ctx, s string) {
	c.name = s[1 : len(s)-1]
	c.ok = true
}
```

**B：戻り値で返す**

```go
// s は "Alice" のように、前後に " が付いた文字列とする
func parseName(s string) (name string, ok bool) {
	return s[1 : len(s)-1], true
}
```

#### 呼び出し元から見た違い（追いやすさ）

```go
// A
c := &ctx{}
parseName(c, s)
fmt.Println(c.name) // c.name はいつ書かれた？ → parseName の中を読まないと分からない

// B
name, ok := parseName(s) // この行で name と ok に値が入ったと分かる
fmt.Println(name)
```

Aでは、呼び出し元に `parseName(c, s)` としか書かれません。`c` のどのフィールドが変わったのかは、`parseName` の中を読まないと分かりません。最初の疑問で感じた「いつ `c.value` に値が書き込まれたのか分からない」は、まさにこの状態です。

Bでは、`name, ok := parseName(s)` の1行で、どこで値が入ったのかが分かります。

#### テストから見た違い（テストしやすさ）

```go
// A
func TestParseName(t *testing.T) {
	c := &ctx{} // 先に入れ物を用意する
	parseName(c, `"Alice"`)
	// name と ok が書かれることは、実装を読んで知っておく必要がある
	if c.name != "Alice" || !c.ok {
		t.Errorf("got name=%q ok=%v", c.name, c.ok)
	}
}

// B
func TestParseName(t *testing.T) {
	name, ok := parseName(`"Alice"`)
	if name != "Alice" || !ok {
		t.Errorf("got name=%q ok=%v", name, ok)
	}
}
```

Bでは、関数のシグネチャ（関数名・引数・戻り値の並び）を見るだけで、「何を受け取り、何を返すか」が分かります。シグネチャがそのまま関数の約束になります。

Aのシグネチャから分かるのは、「`c` が書き換えられるかもしれない」ということだけです。どのフィールドが書き換えられるかは分かりません。たとえば `parseName` がほかのフィールドまで書き換えていても、テストで調べていなければ気づけません。

この例は小さいので、差はわずかです。構造体のフィールドが増えたり、入力と出力が同じ構造体に混ざったりするほど、差は大きくなります。gjsonの `parseContext` がその例です。

| | A：ポインタに書き込む | B：戻り値で返す |
| --- | --- | --- |
| 呼び出し元で、値が入る場所が分かるか | 分からない（中を読む必要がある） | 分かる |
| シグネチャから出力が分かるか | 「書き換えるかもしれない」ことだけ分かる | 分かる |
| テストの準備 | 入れ物を作る必要がある | 引数を渡すだけ |

### 2. gjsonはAの形になっている

`Get` の中で `c` に関わる行だけを抜き出すと、次のようになります。

```go
var c = &parseContext{json: json} // 2261行目：c を作る
// ...
parseObject(c, i, path)           // 2269行目：戻り値を使っていない
// ...
return c.value                    // 2285行目：c.value を返す
```

`parseObject` の戻り値を捨てているのに、最後に `c.value` を返しています。つまり、`parseObject` が `c.value` に結果を書き込んでいます。これは1のAと同じ形です。

**読み方のヒント**：戻り値を捨てているのに、引数に渡したポインタのフィールドを後で使っていたら、その関数がそこに書き込んでいると考えられます。

`parseContext` のフィールドを、役割ごとに整理すると次のようになります。

| フィールド | 役割 | 書いている場所 | 読んでいる場所 |
| --- | --- | --- | --- |
| `json` | 入力 | `Get` [2261行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L2261) | parse関数 |
| `lines` | 入力（設定） | `Get` [2263行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L2263) | どこからも読まれていない |
| `value` | 出力（結果） | `parseObject` [1321-1385行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L1321-L1385)、`parseArray` [1610-1843行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L1610-L1843)の約30か所 | `Get` [2280・2285行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L2279-L2285)、`fillIndex` |
| `pipe` / `piped` | 出力（後処理の指示） | `parseObject` [1237行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L1236-L1239)、`parseArray` 1562・1590・1760行目 | `Get` 2279-2280行目 |
| `calcd` | 出力（フラグ） | `parseArray` 1808行目 | `fillIndex` [3442行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L3441-L3450) |

この表から、最初の疑問を裏付ける事実が4つ分かります。

- **入力と出力が1つの構造体に混ざっています。** `c.lines = true` は入力側、`value` や `pipe` は出力側の書き換えで、役割が違います。
- **`value` はフィールドごとにバラバラに書かれています。** たとえば1321-1326行目では、`c.value.Str`・`c.value.Raw`・`c.value.Type` を別々の行で書いています。いつ書かれるかを知るには、2つの関数の約30か所を読む必要があります。
- **「何も書かれなかったこと」にも意味があります。** 値が見つからなければ `c.value` はゼロ値（Goで初期化しないときの値。構造体なら全フィールドが `0` や `""`）のままです。その結果、`Exists()`（694-696行目）が false を返します。「見つからない」という結果も、コード上では見えない形で表されています。
- **`c.lines` は書かれるだけで、どこからも読まれていません。** `gjson.go` と `gjson_test.go` を検索しても、読んでいる箇所はありません。中身の見えない状態は、使われなくなったフィールドが残っても気づきにくい、という例です。

### 3. 提案の書き方が、そのままでは成り立たない理由

```go
c.pipe, c.piped = parseArray(c, i, path)
```

この書き方の狙い（呼び出し元で書き換えが見えるようにする）は、1のBと同じで正しい方向です。ただし、gjsonではそのままでは成り立ちません。理由は3つあります。

**理由1：戻り値の意味が違う**

`parseArray` の戻り値 `(int, bool)` は、「次に読む位置」と「値が見つかったか」です。`pipe` の情報ではありません。この2つは再帰（関数が自分自身を呼ぶこと）で欠かせません。たとえば1331行目の `i, hit = parseObject(c, i+1, rp.path)` では、`i` で続きの読み取り位置を受け取り、`hit` で探索を打ち切るかを決めています。

**理由2：結果が決まるのは、再帰の一番深いところ**

`{"user":{"name":"Alice"}}` から `user.name` を取り出すとき、処理はおおよそ次のように進みます。

```text
Get
 └─ ① parseObject(c, 1, "user.name")   … 一番外側で "user" を探す
     └─ ② parseObject(c, 9, "name")     … user の中で "name" を探す
         └─ "Alice" を見つける → c.value に書き込む → hit=true を返す
     ← ① は hit=true を受け取り、そのまま true を返す
 ← Get は c.value を返す
```

`Get` が受け取るのは、パースが終わった後の最終結果（`Alice`）だけです。途中の結果を返すわけではありません。

問題は、その最終結果を誰が `Get` まで運ぶかです。`Alice` を見つけるのは ② ですが、`return` は自分を呼んだ関数にしか戻りません。② が `return` すると ① に戻り、`Get` には直接届きません。

**小さな例：途中の関数にも仕事がある場合**

`{"user":{"name":"Alice"}}` を（パースを省くために）map で表し、`user` → `name` の順に探す例で比べます。どちらも `go test` で動くことを確かめています。

B：戻り値で返す

```go
func Get(data map[string]any) (string, bool) {
	return findUser(data)
}

func findUser(data map[string]any) (string, bool) {
	user, ok := data["user"].(map[string]any) // 自分の仕事：user を探す
	if !ok {
		return "", false
	}
	name, found := findName(user) // name 探しは findName に任せる
	return name, found            // ← 受け取った結果を、そのまま返す
}

func findName(user map[string]any) (string, bool) {
	name, ok := user["name"].(string) // 自分の仕事：name を探す
	return name, ok
}
```

A：ポインタに書き込む

```go
type ctx struct {
	name  string
	found bool
}

func Get(data map[string]any) (string, bool) {
	c := &ctx{}
	findUser(c, data)
	return c.name, c.found
}

func findUser(c *ctx, data map[string]any) {
	user, ok := data["user"].(map[string]any) // 自分の仕事：user を探す
	if !ok {
		return
	}
	findName(c, user) // ← c を渡すだけ。結果は受け取らない
}

func findName(c *ctx, user map[string]any) {
	name, ok := user["name"].(string) // 自分の仕事：name を探す
	if ok {
		c.name, c.found = name, true // Get が持っている c に直接書く
	}
}
```

どちらの `findUser` も、`user` を探すという自分の仕事を持っています。そのうえで、どちらにも中継があります。違うのは、何を、どちら向きに運ぶかです。

| | A：ポインタに書き込む | B：戻り値で返す |
| --- | --- | --- |
| 呼ぶとき（下向き） | `c` を渡す | `user` を渡す |
| 戻るとき（上向き） | 何も運ばない | `name, found` を受け取って返す |

- 下向きは、どちらも何かを渡すので差はありません。gjsonでも、Aは `json` の代わりに `json` 入りの `c` を渡しているだけで、引数の数は同じです。
- 差が出るのは上向きです。Bの `findUser` は、結果の形（`string, bool`）を知っていて、それを運ぶ必要があります。Aの `findUser` は、結果が何なのかを知らなくても済みます。

**gjsonに当てはめると**

gjsonをBの形にすると、途中で呼ばれている関数（上の図の ①）が、② の結果を受け取って返し直す必要があります。

```go
// 出力だけをまとめた構造体
type parseResult struct {
	value Result
	pipe  string
	piped bool
	calcd bool
}

func parseObject(json string, i int, path string) (int, bool, parseResult)

// 再帰呼び出しの4か所すべてで、受け取って返し直す
i, hit, res = parseObject(json, i+1, rp.path)
if hit {
	return i, true, res
}
```

このとき返すのは、`parseContext` そのものではありません。`parseContext` には `json` などの入力も入っているからです。返すのは、出力だけをまとめた別の構造体（上の `parseResult`）になります。

小さな例より手間が増える理由は3つあります。

1. parse関数は、結果のほかに「次に読む位置（`i`）」と「見つかったか（`hit`）」も返しています。結果はそこに上乗せすることになります。
2. すべての `return` に結果を足す必要があります。`parseObject` には `return` が11か所、`parseArray` には17か所あります（中にある小さな関数の分は除く）。見つからなかった場合も、空の結果（`parseResult{}`）を返します。
3. パスが深いほど、中継が長くなります。`"user.name"` なら2段、`"a.b.c.d"` なら4段です。

ただし、段数が増えても、1つの関数の返り値の数は変わりません。増えるのは「受け取って返す」中継の箇所の数です。結果を構造体にまとめて返せば、Bでも中継は `return findName(user)` の1行で済み、結果に項目を足しても途中の関数は変わりません。そのため、段数が多いというだけでAを選ぶ必要はありません。Bで書くことも十分にできます。gjsonは、手間と読みやすさのトレードオフでAを選んだ、と考えるのが正確です。

**理由3：`c` を渡している限り、中での書き換えは防げない**

`c.pipe, c.piped = parseArray(c, i, path)` と書いても、`parseArray` に `c` を渡しているので、中で `c` を書き換えることはできてしまいます。呼び出し元でだけ書き換わることを保証するには、`c` を渡すのをやめる必要があります。

### 4. なぜgjsonはAを選んだのか（推測）

作者が理由を書いた記録は見つけていません。以下は推測です。

- **一番の理由は、再帰を通して結果を届けるのが簡単になることだと考えます。** 戻り値では制御に必要な「位置」と「見つかったか」だけを返し、最終結果は書き込み先に直接置きます。こうすると、途中で呼ばれている関数は `hit` を返すだけで済みます。
- **性能も理由の一つかもしれませんが、決め手とは言えません。** gjsonのREADMEは「fast」を掲げていて、`Get` のベンチマークは「0 allocs/op」（メモリ割り当てが0回）です。ただ、`Result`（64ビット環境で約80バイト）を戻り値で返しても、割り当ては増えないはずです。違いは、関数から戻るたびに結果をコピーする程度です。測定もしていません。
- **Goの標準ライブラリにも同じ形があります。** `encoding/json` の `decodeState.unmarshal` は、`d.value(rv)` を呼んだ後、深いところで書かれた `d.savedError` を返します（[decode.go 179-194行目](https://github.com/golang/go/blob/2c3c1caad2079458fa7248ea1814797fc2e94e8a/src/encoding/json/decode.go#L179-L194)、2026-10-12閲覧）。

### 5. Aのまま読みやすくするには

Aの形は残したまま、性能を変えずに、見えない部分を減らせます。

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

1. **入力と出力をコメントで分ける。** 使われていない `lines` は消す。
2. **出力は、フィールドごとではなく構造体ごと代入する。**

   ```go
   // 今：フィールドごとに書く
   c.value.Str = val[1 : len(val)-1]
   c.value.Raw = val
   c.value.Type = String

   // 案：構造体ごと書く
   c.value = Result{Type: String, Raw: val, Str: val[1 : len(val)-1]}
   ```

   こうすると、`c.value =` で検索するだけで、書き込んでいる箇所がすべて見つかります。ただし、ほかのフィールドがゼロ値に戻るので、動作が変わらないかは1か所ずつ確かめる必要があります。
3. **`Get` の呼び出し箇所に、「結果は `c.value` に入る」とコメントを書く。**

### 6. 書き込みを1か所にまとめる（AとBの組み合わせ）

「自分の疑問と意見」の4で出た、「構造体のフィールドへの書き込みは1か所にしたほうがよい」という考えについてです。AIもこれに賛成です。

**散らばった書き込みが、読みにくさの中心**

例の `findName` では、`c` への書き込みは1か所だけでした。gjsonでは、`c.value` への書き込みが約30か所に散らばっています。見つかった値の種類（文字列、数値、true/false、オブジェクト、配列など）ごとに、別々の分岐で書いているためです。最初の疑問で感じた「バラバラでよく分からない」の原因は、Aの形そのものよりも、この散らばりです。

**gjsonには、Bの形で値を作る関数がすでにある**

`gjson.go` の[2420-2496行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L2420-L2496)に、`parseAny` という関数があります。

```go
// parseAny parses the next value from a json string.
// A Result is returned when the hit param is set.
// The return values are (i int, res Result, ok bool)
func parseAny(json string, i int, hit bool) (int, Result, bool)
```

JSONの値を1つ読み、`Result` を戻り値で返す関数で、Bの形です。`ForEach`（329行目）、`parseArray` の一部（1777行目）、`ForEachLine`（1942行目）で使われています。

ところが `parseObject`（1308-1390行目）は `parseAny` を使っていません。ほぼ同じ処理を自前で持ち、`c.value` にフィールドごとに書き込んでいます。

**`parseAny` を使って書き込みを1か所にする案**

```go
// イメージ：パスの最後まで一致した（hit）ときだけ、値を parseAny で作る
if hit {
	var res Result
	i, res, ok = parseAny(c.json, i, true)
	if ok {
		c.value = res // c.value への書き込みはここだけ
	}
	return i, ok
}
// hit でなければ、今のまま読み飛ばすか、さらに深く探す
```

値を作る処理は `parseAny` に任せ、`c.value` への書き込みは1か所にまとめます。値を作る処理のテストも、`parseAny` 単体でできます。

ただし、そのまま全部を置き換えることはできません。`parseAny` は `hit` が false のとき、true/false/null を読んでも `return` せず、次の値まで読み進めてしまいます（[2467-2479行目](https://github.com/tidwall/gjson/blob/690362d6edf4bcbde1e9f54d552d3814f1cd5bcb/gjson.go#L2467-L2479)）。今の呼び出し元はすべて `hit` に true を渡しているので、問題は起きていません。そのため、上の案のように「hit のときだけ使う」形にする必要があります。それでも、動作がまったく同じになるかは確かめていません。

なぜ `parseObject` が `parseAny` を使わず、同じ処理を自前で持っているのかも分かっていません。性能のためかもしれないし、歴史的な経緯かもしれません（推測）。

**自分の設計に持ち帰るなら**

- **値を作る関数は、戻り値で返す（B）。** テストしやすく、何を返すかがシグネチャで分かります。
- **共有の入れ物への書き込みは、1か所にまとめる（A）。** どこで書き込まれるかを探すとき、1か所を見れば済みます。

### 7. 使い分けのまとめ

| 場面 | おすすめ |
| --- | --- |
| ふだんのコード（ほとんどの場合） | B：戻り値で返す。結果が多ければ構造体にまとめて返す |
| 途中の関数が制御の情報（位置、見つかったかなど）も返していて、結果まで載せると負担が大きい場合（gjsonのparse関数） | A：書き込み先を渡す。ただし、何が出力なのかをコメントや型で示す |
| Aを使う場合 | 値を作る関数は戻り値で返し、入れ物への書き込みは1か所にまとめる |

迷ったら、まずBで書きます。段数が多いというだけでAを選ぶ必要はありません。

### よくある疑問

**Q. 「戻り値で返す」とは、`parseContext` を戻り値にすることですか？**

いいえ。関数の出力を、引数のポインタに書き込むのではなく、`return` で返すことです。`parseContext` には入力（`json`）も入っているので、そのまま返すことにはなりません。返すのは、出力だけをまとめた値や構造体です（3の `parseResult`）。

**Q. Bにすると、`Get` にパースの途中の結果を返すのですか？**

いいえ。`Get` が受け取るのは、パースが終わった後の最終結果だけです。「受け取って返し直す」のは、その最終結果を運ぶためです。`return` は自分を呼んだ関数にしか戻らないので、一番深い関数で決まった結果は、途中で呼ばれている関数を順に通って `Get` に届きます（3の理由2）。

**Q. Aでも、途中の関数は `c` を渡しているだけではないですか？**

はい。AにもBにも中継はあります。違うのは、何を、どちら向きに運ぶかです。Aは入れ物（`c`）を下向きに渡し、結果は上向きには運びません。Bは結果を上向きに運びます（3の理由2の表）。また、実際のコードでは途中の関数にも自分の仕事（`user` を探すなど）があるので、意味のない関数にはなりません。

**Q. 引数がポインタなら、必ず書き換えられるのですか？**

必ずではありません。シグネチャから分かるのは「書き換えられるかもしれない」ことだけです。どのフィールドが書き換えられるのかは、中を読まないと分かりません。

**Q. Aは悪い書き方ですか？**

いいえ。`encoding/json` のように、標準ライブラリでも使われています。問題はAの形そのものではありません。何が出力で、どこで書かれるのかを、コードやコメントで示していないことです。

## 自分で確かめること

新しい範囲のコードを読むので、再訪のときに確かめます。確かめたら `[x]` にし、結果を再訪メモに書きます。

- [ ] parseContextの各フィールドは入力か出力か。どこで書かれ、どこで読まれるか
- [ ] `c.lines` は書かれるだけで読まれていないか。そうなら、いつからか（`git log -S "c.lines"`）
- [ ] 戻り値を `(int, bool)` にしているのは、再帰で「位置」と「見つかったか」を使うためか（1331行目など）
- [ ] `user.name` の処理が、「3. 理由2」の図のとおりに進むか
- [ ] `encoding/json` の `decodeState` も同じ形か
- [ ] なぜ `parseObject` は `parseAny` を使わず、同じような処理を自前で持っているのか（`git blame`、`git log -S "parseAny"`）
- [ ] `parseAny` は `hit` が false のとき、true/false/null で `return` しないか（2467-2479行目）。呼び出し元がすべて true を渡しているので問題にならないのか

## 自分の結論

議論を読んで、納得したこと・納得できないことを自分の言葉で書きます。書けたら、観察メモの「自分の設計に活かせること」に移します。

- 納得したこと：
- 納得できないこと・まだ分からないこと：
- 自分の設計に活かせること：
