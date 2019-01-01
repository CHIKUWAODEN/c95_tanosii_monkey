評価
========
{: .counting }

　新しく追加した AST を評価するために、評価器（Evaluator）の方も対応していきましょう。
評価機周りの修正は、Object インターフェイスに対応する Object の追加と、評価関数と呼ばれる Eval() の修正から成り立っています。
まずはオブジェクトから見ていきましょう。


新しいオブジェクトタイプの追加
--------
{: .counting }

　まず、これから私達が表現したい新しいオブジェクトの種類を追加します。
追加するのは次の四つです。
もしかしたら、四という数字に疑問を持たれたでしょうか？
ここにはクラス、（クラスの）インスタンス、そして this があります。
もう一つ・・・これが余計に見えるかもしれないのですが、`REFERENCE` というものが見えます。
じつはこの `REFERENCE` というのが、後に約に立つことになります。


    // object.go
    const (
        ...
        CLASS_OBJ        = "CLASS"
        INSTANCE_OBJ     = "INSTANCE"
        THIS_OBJ         = "THIS"
        REFERENCE_OBJ    = "REFERENCE"
    )


オブジェクト構造体を追加する
--------
{: .counting }

　それでは、新しく追加する各オブジェクトがどのようなものか紹介します。
言うまでもありませんが、これらは Object インターフェイスに準拠しているため、全て評価関数 evaluator.Eval() の戻り値とすることができます。


### Class
{: .counting }

　クラスを表現するためのオブジェクトです。
Eval に ast.ClassLiteral が渡された場合に、この構造体のインスタンスが生成されます。
クラス名を表す Name と、Body はクラスの内容を定義する BlockStatement となっています。


    // object.go
    type Class struct {
        Name *ast.Identifier
        Body *ast.BlockStatement
    }

    func (c *Class) Type() ObjectType { return CLASS_OBJ }
    func (c *Class) Inspect() string {
        var out bytes.Buffer

        out.WriteString("class")
        out.WriteString(c.Name.String())
        out.WriteString("{")
        out.WriteString(c.Body.String())
        out.WriteString("}")

        return out.String()
    }


### Instance
{: .counting }

　object.Instance はインスタンスを表現するものになります。
それぞれのインスタンスは、独立した環境（Environment）を持つことになるため、これを This として持たせています。


    // object.go
    type Instance struct {
        Class *Class
        This  *Environment
    }

    func (i *Instance) Type() ObjectType { return INSTANCE_OBJ }
    func (i *Instance) Inspect() string {
        var out bytes.Buffer

        out.WriteString("instanfe of ")
        out.WriteString(i.Class.Name.String())

        return out.String()
    }


### Class
{: .counting }

　This はその実態としては object.Instance へのエイリアスです。
これまでなんどか述べているとおり、`this` というキーワードを識別子とし object.Instance を参照できるように実装できれば、
無用の長物かもしれません。


    // object.go
    type This struct {
        Instance *Instance
        Name     string
    }

    func (t *This) Type() ObjectType { return THIS_OBJ }
    func (t *This) Inspect() string {
        return "this is " + t.Instance.Inspect()
    }


### Reference
{: .counting }


　object.Reference は少し特殊なオブジェクトです。
まずは実装をみてみましょう。
ただの Object インターフェイスにに準拠した他の構造体とはことなり、Assing(), Value() といった関数が設定されているのがわかるでしょうか。


    // object.go
    type Reference struct {
        Env  *Environment
        Name string
    }

    func (r *Reference) Type() ObjectType { return REFERENCE_OBJ }
    func (r *Reference) Inspect() string {
        obj, ok := r.Env.Get(r.Name)
        if ok {
            return obj.Inspect()
        }
        return "<missing reference>"
    }

    func (r *Reference) Assign(obj Object) Object {
        return r.Env.Set(r.Name, obj)
    }

    func (r *Reference) Value() Object {
        obj, ok := r.Env.Get(r.Name)
        if ok {
            return obj
        }
        return &Null{}
    }


　なぜ Reference といったものを利用するかというと、代入演算子を実装するためです。
代入演算子の左辺を評価したときに、この Reference を返すことで、Assign() よって右辺の値を使った代入操作が実現できます。
必要性などの詳しいことは Eval() の修正内容の紹介とともに解説したいと思います。


評価関数 Eval() の修正
--------
{: .counting }

　さて、残るは評価器の中枢 Eval() 関数の修正になります。
簡単に Eval() を説明してみましょう。
Eval() は AST ノードの種類で処理を分岐し、ノードの情報を用いて Object をレスポンスします。
また、Eval() は再帰的に Eval() を呼び出してシンタックスツリーを渡り歩きながら（Tree-Walking）全体を評価していきます。

　Eval() の修正に入る前に、我々が評価したい Monkey のソースコードをもう一度確認してみましょう。
このセクションが終わるころには、Monkey の処理系はこのコードを完璧に実行し、"Jhon doe" という値を REPL の上で表示することができるようになるはずです。

    class Foo
    {
      let myName = "bar";
      let constructor = fn(name) {
        this.myName = name;
      };
    };

    let foo = Foo("Jhon doe"); 
    foo.myName;


### クラスオブジェクトの生成
{: .counting }

　まずは ast.ClassLiteral の評価とクラスオブジェクトからみていきましょう。
ast.ClassLiteral は次のように評価され、object.Class を生成します。
ポイントなのは object.Class を生成したその場で、env にそのクラス名でもって Set() しているところです。


    // evaluator.go
    func Eval(node ast.Node, env *object.Environment) object.Object {
        switch node := node.(type) {
        ...
        case *ast.ClassLiteral:
            // Body 部分は Function と同様に
            // applyFunction のときに評価する
            classObj := &object.Class{
                Name: node.Name,
                Body: node.Body,
            }
            // object.Class を Env に登録しておく
            // これがコンストラクタとして評価される
            env.Set(node.Name.Value, classObj)
            return classObj
        }        
        return nil
    }

　たとえば `class Foo {};` という定義をしたとすると、Foo という名前で object.Class が env に登録されます。
そして、インスタンスの生成構文は、`<クラス名>()` とするようにしたのは本誌冒頭で示したとおりです。
この書き方は、関数呼び出しそのものといえます。
つまり、インスタンスの生成は、object.Class に対して関数呼び出しを行う事ととらえることができるわけです。

　しかし、object.Class はそもそも関数（object.Function）ではありません。
どうやって関数として呼び出せばよいのでしょうか。
それを実現するために object.Function を評価する関数 applyFunction にちょっとだけいたずらをしかけます。


    // evaluator.go
    func applyFunction(fn object.Object, args []object.Object) object.Object {
        switch fn := fn.(type) {
        ...
        case *object.Class:
            // コンストラクタの呼び出しとして評価し
            // object.Instance を生成する
            instance := &object.Instance{
                Class: fn,
                This:  object.NewEnvironment(),
            }

            // this を暗黙的にインスタンスの環境に束縛しておく
            instance.This.Set("this", instance)

            // 横着して Eval を使って、Body を評価する
            // ちなみに、現状だとこの BlockStatement の中に
            // return が書けてしまう
            // ここで評価された BlockStatement の内容は
            // This という環境の中で処理される
            Eval(fn.Body, instance.This)

            // もしコンストラクタを持っているならこの時点でこの
            // appylyFunction を使って関数として評価してやる
            ctor, ok := instance.This.Get("constructor")
            if ok {
                fn, ok := ctor.(*object.Function)
                if ok {
                    extendedEnv := extendedFunctionEnv(fn, args)
                    Eval(fn.Body, extendedEnv)
                }
            }
            return instance
        ...
        }
    }


　ごらんの通り applyFunction が object.Class に対してうまく振る舞うようにしてやることができるのです。
本来の Monkey 処理系においても、対象としたものがビルトインか、スクリプトを評価することで生成された object.Function なのかによって処理を分岐しています。
そこに、class.Object であった場合の処理を足すことはさほど難しいことではありません。

　さて、次の問題はその中身となります。
注目してほしい場所が二つあり、その一つが生成したばかりの object.Instance.This に 'this' という名前で自分自身をセットしている部分です。
こうすることで、メンバ関数の中で this という名前を通してインスタンスのメンバにアクセスすることができます。

　もう一つがコンストラクタの呼び出しです。
Body を Eval() にかけることで、Instance.This には constructor という名前で object.Function がセットされます。
Eval() の直後に instance.This に `constructor` という名前で、object.Function が存在していれば、その場でさらに Eval に処理させます。
こうすることで、インスタンスの生成と同時にコンストラクタを呼び出すという処理を実現させてみました。


### ドット演算子と this
{: .counting }

　次はコンストラクタの処理の中身に注目してみましょう。
行っているのは `this.myName = name;` という処理です。
この文を少し噛み砕くと「インスタンスメンバ myName を対象として」「引数 name を代入する」という形になっています。
まずは `this.myName` の部分、ドット演算子の解釈から見ていきましょう。

    // evaluator.go
    func Eval(node ast.Node, env *object.Environment) object.Object {
        switch node := node.(type) {
        ...
        case *ast.DotExpression:
            left := Eval(node.Left, env)
            if isError(left) {
                return left
            }
            return evalDotInfixExpression(left, node.Right)
        ...
        }
        return nil
    }

　Eval() のメインの switch 文は ast.DotExpression ノードの左辺値（Left）を Eval() します。
このとき Left は this の事を意味しているはずです。
では、this はどのように評価されるのでしょうか？順番に従いここをみてみましょう。


    // evaluator.go
    func Eval(node ast.Node, env *object.Environment) object.Object {
        switch node := node.(type) {
        ...
        case *ast.This:
            return evalThis(node, env)
        ...
        }
        return nil
    }


　実際に処理を行うのは、`evalThis` というヘルパー的な関数です。
このメソッドは簡単に、もし env（ここで想定しているのは、object.Instance.This）に this が存在しなければエラーを返し、
もし存在しているのであればそれが object.Instance であると確認したうえでレスポンスします。
先に説明したとおり、object.Class に他する関数呼び出しの適用の再に、object.Instance.This に this という名前で object.Instance 自身が登録したります。
そのため、this は適当に object.Instance として評価されてくれるはずです。


    // evaluator.go
    func evalThis(node *ast.This, env *object.Environment) object.Object {

        this, ok := env.Get("this")
        if !ok {
            return newError("'this' not found")
        }

        // インスタンス生成時に予め this をインスタンスの Environment に Set してある
        instance, ok := this.(*object.Instance)
        if !ok {
            return newError("'this' is not instance")
        }

        return instance
    }


　this の評価を首尾よく終えられれば、続いて evalDotInfixExpression という関数が呼び出されます。
これはドット演算子を評価するためのヘルパーの役割です（evalThis と似ていますね）。


    func evalDotInfixExpression(
        left object.Object,
        right *ast.Identifier,
    ) object.Object {

        inst, ok := left.(*object.Instance)
        if !ok {
            return newError("unexpecte left value type")
        }

        _, ok = inst.This.Get(right.Value)
        if !ok {
            return newError("undefined member : %s", right.Value)
        }
        return &object.Reference{
            Env:  inst.This,
            Name: right.Value,
        }
    }


　この関数は、引数 left が object.Instance であった場合にのみ動作します。
そしてここで返すのが object.Reference ということに注目してください。
なぜ素直に `inst.This.Get()` の戻り値である myName の値、つまり object.String を返さないのでしょうか？
それは、次に説明する代入操作を実現するためには、名前に束縛されている値オブジェクトを操作するだけではだめで、きっちりと env に対して新しい値を Set() してやらなければならないからです。


### 代入
{: .counting }

　残るは代入処理のみです。
これまでの処理で、Reference が得られています。
代入処理関数はとても簡単で、左辺から得られた object.Reference に対して、右辺を評価して得られた Object を Assign() という関数を通して設定するだけです。

    // evaluator.go
    func Eval(node ast.Node, env *object.Environment) object.Object {
        switch node := node.(type) {
        ...
        case *ast.AssignmentExpression:
            // Expression
            left := Eval(node.Left, env)
            if isError(left) {
                return left
            }

            // Expression
            right := Eval(node.Right, env)
            if isError(right) {
                return right
            }

            ref, ok := left.(*object.Reference)
            if !ok {
                newError("...")
            }
            ref.Assign(right)
        }
        return nil
    }

　ここで仮に、object.Reference が object.String や object.Integer だったらどうでしょうか？
いや、左辺値数字や文字列を置くわけがないじゃないか、と思われるかもしれませんが、Eval() は左辺に識別子が置かれると、識別子に結びつけられた値オブジェクトを
Environment から探索し返そうとしますし、その中身は実に object.String や obejct.Integer となってしまいます。
これでは Env を書き換えることができません。

　では ast.AssignmentExpression の Left の型を Identifier としたら？
これは実装にもよるのでしょうが、少なくとも今回のアプローチではこれが無理筋でした。
なぜなら、代入演算子の評価を行うより先に、左辺のドット演算子を評価しなければならなかったからです。
左辺は識別子ではなく、式として評価されることを求められるゆえ、左辺を Expression とせざるをえなかったのです。

　ここでようやく登場するのが Reference です。
これは、まちがいなく object.Obejct の定めるインターフェイスを満たしているのですが、実際には値を持っていません。
Environment と参照するための名前となる string を持ち、値のフリをすることができるオブジェクトなのです。
Reference のポイントは、フリをする対象となる値の属する Environment と名前をセットで保持しているので Environment が抱える値をセットする、つまり上書きすることができるという点です。
これでうまく束縛するしかできなかった値を書き換えるという動作を実現させることができました。


### インスタンスメンバ以外への代入
{: .counting }


　これまで `this.myName` や `foo.myName` といった、ドット演算子式を左辺においた代入演算子の処理ばかりみてきました。
当然 `let a = 0; a = 1;` といったこともできるようになっているはずと思われるかもしれませんが、実はそうではありません。
なぜかというと、evalIdentifier という識別子ノードを評価する関数が Reference を返さないからです。

　しかし、私たちはこれに対する対策をすでに用意しています。
それが、構文解析器の修正のついでに ast.Identifier に追加しておいた Reference という値です。
これをどう使うのか、早速みてみましょう。

    // evaluator.go
    func evalIdentifier(node *ast.Identifier, env *object.Environment)
    object.Object {
        val, ok := env.Get(node.Value)
        if ok {
            if node.Reference {
                return &object.Reference{
                    Env:  env,
                    Name: node.Value,
                }
            }
            return val
        }
        ...
    }

　必要な部分だけを抜粋しました。
ポイントは、ast.Identifier.Referece が真のときだけ、戻り値を object.Reference で返し、
そうでなければ Environment.Get() から返されたものをそのまま返しています。

　どうしてこのようなことをするかというと、識別子に求められる性質が文脈によって変化するからです。
代入式の場合は左辺は Reference としなければならないように評価器を改造してきましたが、単体で識別子が評価された場合は別です。
なぜなら、Monkey の実装は Eval() で返される値はオブジェクトであるというルールのもと、テストや各種のコードが実装されています。
そこでいきなり evalIdentifier の戻り値が object.Reference になってしまうと、もろもろの実装が壊れてしまいます。

　そこで、今回は最小限の手間でこれを回避するため、先のトークンによって挙動を変えるという'ずる'をしたわけです。
このやり方は割とうまく行って、テストなども破壊することなく済みましたが、あまり筋は良くないとは正直に思います。


### ドット演算子のもう一つの振る舞い
{: .counting }

　Monkey のサンプルコードの最後にある `foo.myName` の部分を一応最後にみてみましょう。
といってもこれは簡単で、foo が Identifier であるため、グローバルな Environment から foo に束縛された値、つまりコンストラクタが返す object.Instance として解決されます。
object.Instance であればあとは this と同じ要領で処理され、Reference が返されるわけです。
REPL では最終的に Eval() が返したオブジェクトに対して Inspect() を実行し、実際の値を表示しようとしますが、これは Reference が指し示す値の Inspect() が実行されることとなるため、こちらでもちゃんと代入後の値である "Jhon doe" が表示されてくれるという寸法です。


　以上！おつかれさまでした！