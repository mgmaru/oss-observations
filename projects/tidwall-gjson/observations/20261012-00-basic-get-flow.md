# 問い00：JSON内のユーザー名を取り出す処理はどう進むか？

- 状態：未着手
- 観察日：2026-10-12（月）
- 観察範囲：ネストしたオブジェクトから文字列を1つ取得するユースケース
- 対象バージョン：
- コミットSHA（観察予定）：690362d6edf4bcbde1e9f54d552d3814f1cd5bcb
- 種別：
- 今回の所要時間：0分
- 観察時間：1時間50分

| 項目 | 内容 |
| --- | --- |
| 入力 | `{"user":{"name":"Alice"}}` |
| 取得したい値 | `user` に含まれる `name` の文字列（`Alice`） |

## 仮説
- 「ネストしたオブジェクトから文字列を1つ取得するユースケース」を追う。
- 処理の流れの予想としては以下。
1. jsonをParseする。
2. ブロックの中の`user`を検索（キー検索）
3. 含まれていたらネストを解体し、`name`を検索（キー検索）
4. その中の値（`Alice`）を抽出

## 観察メモ・根拠
## gjson.go
### ①公開API
- まず、このGet関数にjson文字列と検索するキー（文字列）が入る。
- 返り値にResult型があるので、この関数で結果を返している。
#### 行2177-2286
```go
func Get(json, path string) Result {}
```
- 最初の数行で特殊な修飾子の分岐がある。
- if文の分岐の後で、parseContext構造体にjson文字列を代入している。
  - このparseContextにparseに必要なデータを格納する。
  - 最後にパース関数（`parseObject`または`parseArray`）にparseContextを渡して、値を抽出する。
- 今回のユースケースだと特殊な演算子がない場合についてなのでこれらの分岐には入らず、2265行目の`else`に入る？
- [疑問] 2261行目でparseContextを初期化してその中に、json文字列を格納している。この構造体を複数の関数で書き換えているのは良くない気がした。
  - 2269行目：`parseObject(c, i, path)`
  - 2274行目：`parseArray(c, i, path)`

### ②Result
#### 行66-80
```go
type Result struct {
	// Type is the json type
	Type Type
	// Raw is the raw json
	Raw string
	// Str is the json string
	Str string
	// Num is the json number
	Num float64
	// Index of raw value in original json, zero means index unknown
	Index int
	// Indexes of all the elements that match on a path containing the '#'
	// query character.
	Indexes []int
}
```
### その他
#### 行30-44
```
const (
	// Null is a null json value
	Null Type = iota // 0
	// False is a json false boolean
	False // 1
	// Number is json number
	Number // 2
	// String is a json string
	String // 3
	// True is a json true boolean
	True // 4
	// JSON is a raw block of JSON
	JSON // 5
)
```
- `enum`定義。
- Nullという定数をType型として宣言し、値はiota（=0）とする。
- `Null int = iota`ともできるが、定数のNullをType型にすることで、取り違いを防ぐ。
- [疑問] Typeだと何の型なのかがわからないような気がするがどうか？

## 問いへの回答
1. 公開APIであるGet関数にjson文字列と検索するキーが渡る。
2. parseContextにパース情報を格納する。
3. パース関数にparseContextを渡して値を取り出す。

## 自分の設計に活かせること

## 未確認のこと・次の問い


## 感想
1. 公開APIのGet関数が長く、プログラムを細かく追うことができなかった。
  - 長いプログラムを読むスキルがない。
    - どうすれば良い？？
    - 慣れるしかない？？
2. 分岐のネストが多くて読みづらかった、、、
