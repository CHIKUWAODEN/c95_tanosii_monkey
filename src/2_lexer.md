字句解析
========
{: .counting }

　まずはじめに行うのは、字句解析周りの修正です。
class を宣言するための class や this といったキーワードや、 '.'（ドット）などの演算子を、この度新たにトークンとして扱えるようにするために手を入れていきます。

トークンタイプの追加
--------
{: .counting }

　字句解析器周りの回収は実に簡単です。
これも、元本の実装がちゃんとしていることの現れなのか、実にスマートにいきます。
さて、まずはトークンタイプを追加しないことには始まりません。


    // token.go
    const (
        ...
        CLASS = "CLASS"
        THIS  = "THIS"
        DOT   = "."
        ...
    )

　そして、キーワードとトークンタイプのテーブルもこのように追加します。
ドットはキーワードではなく演算子として扱うため、他の演算子と同じように、ここで出てくることはありません。


    // token.go
    var keywords = map[string]TokenType{
        ...
        "class":  CLASS, // 
        "this":   THIS,
    }


字句解析器の修正
--------
{: .counting }

　字句解析器（Lexer）の修正は、たったの二行で済みました。
下記のようにするだけです。
実に簡単でいいですね。

    // lexer.go
    func (l *Lexer) NextToken() token.Token {
        ...
        switch l.ch {
        ...
        // ここからの2行を追加
        case '.':
            tok = newToken(token.DOT, l.ch)
        ...
        }
        ...
    }


　class と this についてはどうなのかと思われるかもしれませんが、手付かずですでに対応済みです。
具体的には default 節に流れると、class や this はここで一つのトークンとなるように字句解析が行われます。