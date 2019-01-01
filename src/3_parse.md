構文解析
========
{: .counting }

　字句解析器の修正は実に簡単なものでした。
引き続き構文解析器（Parser）を見ていきたいと思います。

　構文解析器の役割は、トークン列という入力から AST<span class="footnote">Abstract Sytax Tree, 仮想構文木<span> を生成することです。
AST の詳細については割愛しますが、ご存知のとおり各トークンに構文上の意味を持たせたものをノードに変換し木構造としたものです。

各種の AST ノードとなる構造体の定義
--------
{: .counting }


### ClassLiteral
{: .counting }

　まずは class キーワードに対応させるための ClassLiteral です。
クラス名となる識別子 `Name` と、クラスの中身を記述するための `Body` をもたせてあります。
ここでミソなのが、BlockStatemet である Body です。
先にいってしまうと、Function などを評価するための関数とまったく同じ評価関数を利用するために、ここでは BlockStatement としています。


    // ast.go
    type ClassLiteral struct {
        Token token.Token
        Body  *BlockStatement
        Name  *Identifier
    }

    func (cl *ClassLiteral) expressionNode()      {}
    func (cl *ClassLiteral) TokenLiteral() string { return cl.Token.Literal }
    func (cl *ClassLiteral) String() string {
        var out bytes.Buffer

        out.WriteString(cl.TokenLiteral())
        out.WriteString(cl.Body.String())

        return out.String()
    }


　今回に限っては、要求をシンプルにまとめただけでもあるのですが、実際にクラスはほぼ関数に近い作りをしています。
関数と違うのは大域的な環境ではなく、まず独自のを持っているだけに過ぎません。
そのあたりについては、評価器のところでもう少し詳しく背名できればと思います。


### DotExpression
{: .counting }

　これはドット演算子による二項式を表現するための AST ノードオブジェクトです。
ドットを二項の演算子と捉え、左辺にはクラスのインスタンスや this などのキーワードを置き、右辺には左辺の環境に束縛された名前となる識別子がくるような形を想定しています。


    // ast.go
    type DotExpression struct {
        Token token.Token
        Left  Expression
        Right *Identifier
    }

    func (de *DotExpression) expressionNode()      {}
    func (de *DotExpression) TokenLiteral() string { return de.Token.Literal }
    func (de *DotExpression) String() string {
        var out bytes.Buffer

        out.WriteString("(")
        out.WriteString(de.Left.String())
        out.WriteString(".")
        out.WriteString(de.Right.String())

        return out.String()
    }


### This
{: .counting }

　`this` キーワードに対応する AST ノードですが、正直なところ、これは必要なかったかもしれません。
後に解説しますが、インスタンスの生成時にそのインスタンスの環境に this という名前でそのインスタンスのオブジェクト自身を束縛するためです。
こうすれば this は単なる名前として表現するだけで今回のところは丸く収まる可能性があったからです。
ただ、今回は試行錯誤のうちにこのようになってしまったので、ありのまま説明してみようと思います。


    // ast.go
    type This struct {
        Token token.Token // token.THIS
        Value string
    }

    func (t *This) expressionNode()      {}
    func (t *This) TokenLiteral() string { return t.Token.Literal }
    func (t *This) String() string       { return t.Value }


### AssignmentExpression
{: .counting }

　代入演算子を表現するための、二項演算子を表現する AST ノードです。
ここで少し工夫してあるのが、左辺となる Left の方を Expression としているところです。
代入演算子の左辺にくるのは、変数名などの識別子ではないのか？と考えられるかもしれませんが、左辺にはドット演算子式が来る可能性があります。
具体的な構文でいうと `foo.bar = 0` であるとか `this.name = "Jhone doe"` などといった具合になります。


    // ast.go
    type AssignmentExpression struct {
        Token token.Token
        Left  Expression
        Right Expression
    }

    func (ae *AssignmentExpression) expressionNode()      {}
    func (ae *AssignmentExpression) TokenLiteral() string { 
        return ae.Token.Literal
    }
    func (ae *AssignmentExpression) String() string {
        var out bytes.Buffer

        out.WriteString("(")
        out.WriteString(ae.Left.String())
        out.WriteString("=")
        out.WriteString(ae.Right.String())
        out.WriteString(")")

        return out.String()
    }


構文解析器（Parser）の修正
--------
{: .counting }


### 新しく追加した演算子の優先順位を定義する
{: .counting }


　各種 AST ノードを表現する構造体の定義ができたので、トークン列から実際にこれらのインスタンスを生成する構文解析器（Parser）の修正に取り組んでいきましょう。
始めにやることは、演算子の優先順位に新しい序列を定義しなおすことと、あたらしく追加したい演算子であるドット演算子と代入演算子について、
その優先順を決めるために優先順位テーブルに新しい要素を追加することです。


    // parser.go
    const (
        _ int = iota
        LOWEST
        ASSIGN      // =（追加) 
        EQUALS      // ==
        LESSGREATER // >, <
        SUM         // +, -
        PRODUCT     // *, /
        PREFIX      // -x, !x
        CALL        // myFunction(x)
        INDEX       // array[index]
        DOT         // foo.name（追加）
    )

　まずは優先順位の定義を行いましょう。
新しく追加した演算子の一つドットは最も高い優先順位を持ち、一方で代入演算子についてはもっとも低い優先順位を持つものとします。
おわかりかと思いますが、代入文に関しては右辺にくる式がすっかり評価し終わったあとの値をもって評価しなければなりません。
一方でドット演算子などは、実質的にシンボルとして動作させたいので、真っ先にこれを評価しておかないとまずいことになりそうです。


    // parser.go
    var precedences = map[token.TokenType]int{
        token.ASSIGN:   ASSIGN, // 追加
        token.EQ:       EQUALS,
        token.NOTEQ:    EQUALS,
        token.LT:       LESSGREATER,
        token.GT:       LESSGREATER,
        token.PLUS:     SUM,
        token.MINUS:    SUM,
        token.SLASH:    PRODUCT,
        token.ASTERISK: PRODUCT,
        token.LPAREN:   CALL,
        token.LBRACKET: INDEX,
        token.DOT:      DOT, // 追加
    }


　追加した優先順位の定義をトークンに関連付けるため、優先順位テーブルにも要素を追加します。
トークンがどういった順番で結合するのかを決定するときにこのテーブルが参照されます。


### parseClassLiteral
{: .counting }

　まずはクラス定義を構文解析するための関数です。
といってもパーサは全般的に単調な作業しか行いません。
カレントトークンやピークトークン（一つ先のトークン）の種類をチェックするのと、トークンの内容から随時新しい AST ノードオブジェクトを生成するのが主な作業です。
これは、AST の種類ごとにその内容に多少の違いはあれど、基本的にはどの構文解析関数にも共通しているます。


    // parser.go
    func (p *Parser) parseClassLiteral() ast.Expression {
        lit := &ast.ClassLiteral{Token: p.curToken}

        if !p.expectPeek(token.IDENT) {
            return nil
        }

        // クラス名
        lit.Name = &ast.Identifier{
            Token: p.curToken,
            Value: p.curToken.Literal,
        }

        if !p.expectPeek(token.LBRACE) {
            return nil
        }

        // クラス本体部分
        lit.Body = p.parseBlockStatement()

        return lit
    }


### parseThis
{: .counting }


　`this` に至っては、識別子と同じ扱いです。
いったいなんのために・・・。
先にも述べましたが、このように This を扱うことについて特に意味はなかったのかなあと思うので、もしかしたら特に意味はなかったのかもしれません。


    // parser.go
    func (p *Parser) parseThis() ast.Expression {
        return p.parseIdentifier()
    }


### parseDotExpression
{: .counting }

　さくさくやっていきましょう。
説明が投げやりっぽく見えるのは、決して手を抜いているわけではありません。
元本でやっているやり方が、シンプルでお上手すぎるため、解説の余地があまりなかったりするだけです。
ほんとですよ。


    // parser.go
    func (p *Parser) parseDotExpression(left ast.Expression) ast.Expression {
        defer untrace(trace("parseDotExpression"))

        expression := &ast.DotExpression{
            Token: p.curToken,
            Left:  left,
        }

        //	precedence := p.curPrecendence()
        p.nextToken()

        expression.Right = &ast.Identifier{
            Token: p.curToken,
            Value: p.curToken.Literal,
        }

        return expression
    }


### parseAssignmentExpression
{: .counting }

　だってあれなんですよ、各種 AST 構造体の説明をした時点で、構文解析関数の説明なんてなにもするものが残っていないんですよね。
だって、トークン列をちょいちょい確かめながら、AST を生成しているだけなんですもの（けっこうまじめにこの解釈であってるはず）


    // parser.go
    func (p *Parser) parseAssignmentExpression(left ast.Expression) ast.Expression {
        defer untrace(trace("parseAssignmentExpression"))

        expression := &ast.AssignmentExpression{
            Token: p.curToken,
            Left:  left, // Identifier or Expression
        }

        precedence := p.curPrecendence()
        p.nextToken()

        expression.Right = p.parseExpression(precedence)

        return expression
    }


### パース関数を関数テーブルに登録する
{: .counting }

　各種の AST ノードに対応した構文解析関数を紹介しました。
そして、これらが実際に利用されるようになるためには、もうひと手間が必要となります。
それがなにかというと、`Parser.New()` の中で構文解析関数テーブルに関数を登録することです。
次のようにやっていきましょう。


    // parser.go
	p.prefixParseFns = make(map[token.TokenType]prefixParseFn)
	p.registerPrefix(token.THIS, p.parseThis)
	p.registerPrefix(token.CLASS, p.parseClassLiteral)

    ...

    p.infixParseFns = make(map[token.TokenType]infixParseFn)
	p.registerInfix(token.DOT, p.parseDotExpression)
	p.registerInfix(token.ASSIGN, p.parseAssignmentExpression)


　これで、追加したトークンに対応した AST が生成されるようになります。


ast.Identifier へのちょっとしたしかけ
--------

　次に進む前に、ast.Identifier にちょっとした仕掛けを施しておきましょう。
次のように ast.Identifier を修正します。


    // Identifier :
    type Identifier struct {
        Token     token.Token // token.IDENT
        Reference bool        // ここを新しく追加した
        Value     string
    }

　Reference という真偽値型の値が追加されています。
ast.Identifier の生成時にこれを設定するように、構文解析関数も修正してしまいましょう。


    func (p *Parser) parseIdentifier() ast.Expression {

        // 次のトークンが代入演算子のトークンであるかどうか
        ref := p.peekToken.Type == token.ASSIGN

        return &ast.Identifier{
            Token:     p.curToken,
            Reference: ref,
            Value:     p.curToken.Literal,
        }
    }

　追加したのは、次のトークンが代入演算子トークンであるかどうかを示すフラグです。
これを ast.Identifier に記録させておくと、あとあと役にたちます。

　さて、これで首尾よく AST が生成できるようになったと思います
引き続き評価器（Evaluator）の実装へとすすみましょう。