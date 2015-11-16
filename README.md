# Scalaのマクロ
________



### きっかけ
shapelessを読もうとしていたらmacroだらけで意味不明だったのでmacroを勉強しようと思った。



### macroとは
(scalaでは)コンパイル時に構文木を受け取り、処理できる
つまりコードを置換する機構



### メリットとデメリット
* メリット  boilerplateの軽減やruntimeでは取得しづらい情報も取得できる。 コンパイル時間が増えて余暇が捗る
* 使いすぎると利用者が混乱する事に。。  ideの支援が受け辛い 




###利用
macro本体の実装は利用するコードより先にコンパイルされていなければいけないのでsbtで使う時は
プロジェクトを分けて先にマクロ側が先にコンパイルされるようにする。

```scala
lazy val user = (project in file("user"))
    .dependsOn(macroImpl)

lazy val macroImpl = (project in file("macro-impl"))
    .settings(
      libraryDependencies ++= Seq(
        "org.scala-lang" % "scala-reflect" % scalaVersion.value
      )
    )
```



## macroの種類
* def macro :   
普通の関数と同じように呼び出され値を返す。  
新しい型を定義したりはできない。  
利用者からみれば普通の関数  
* macro annotation :    
アノテーションとして利用する  
新しい型を定義できる。


## def macroの定義
```scala
object MacroImpl{
    def apply[A, B](a: A): B = macro impl
    
    def impl[A :c.WeakType](c: Context)(a: c.Tree): c.Tree = {
        import c.universe._
        
        val result = a match {
            case 抽象構文木のパターン => 返す値の抽象構文木
            case _ => c.abort(c.enclosingPosition, "マッチしなかったらコンパイルエラー")
        }
    }
}
```


c.enclosingPositionはマクロが展開されている場所で
c.abortでコンパイルエラーにできる。
c.infoで警告も出せる。

 
例
def macroに渡された値がリテラルだったらそのまま返しリテラルではなかったらコンパイルエラーにするmacro
```scala
object isLiteral{
  def apply[A](a: A): A = macro impl[A]

  def impl[A](c: Context)(a: c.Tree): c.Tree = {
    import c.universe._

    a match {
      case Literal(l) => Literal(l)
      case _ => c.abort(c.enclosingPosition, s"$a is not literal")
    }
  }
}
```


単純な例では構文木を手で組み立てれば良いが少し複雑になると途端に手につけられなくなる
```scala
//二つの整数を足してprintする
Apply(Ident(TermName("println")), List(Apply(Select(a, TermName("$plus")), List(b))))
```


????


準クォートを使えばかなり見やすくなる



##準クォートとは ?
ASTの生成をscalaのコードに似たdslで組み立てられる機能
上記の場合
```scala
//二つの整数を足してprintする
Apply(Ident(TermName("println")), List(Apply(Select(a, TermName("$plus")), List(b))))
//準クォートでの表記
q"println($a + $b)"
```


準クォートがどのようなASTに変換されるのかを見るには
```scala
showRaw(q"")
```
で確かめる事ができる。


```scala
showRaw(q"""println("hello world")""")
//Apply(Select(Select(This(TypeName("scala")), scala.Predef), TermName("println")), List(Literal(Constant("hello world"))))
```


準クォートで抽出/パターンマッチもできる 
```scala
val q"class $name  { ..$body }" = tree
```

##マクロの中で型を取得する
```scala
object MacroImpl{
    def apply[A](a: A): Any = macro impl
    
    def impl[A :c.WeakType](c: Context): c.Tree = {
        import c.universe._
        //型の取得
        val tpe = weakTypeOf[A] 
        //シンボルの取得
        val sym = tpe.typeSymbol
    }
}
```


typeSymbolではかなり詳細な型情報を取得できる。
isSealed, isCaseClass, isMethodなどかなり便利   

```scala
//case class  であるか判定
val sym = weakTypeOf[A].typeSymbol

val clazz = if (sym.isClass) sym.asClass else c.abort(c.enclosingPosition, s"$sym is not class")

c.Expr[Boolean](q"${clazz.isCaseClass}")
```


以上を踏まえて渡された値がcase class またはobjectのみで定義されたsealed trait/classか判定するマクロ
(つまり代数的データ型かどうか判定する)    
```scala
def impl[A: c.WeakTypeTag](c: Context): c.Expr[Boolean] = {
    import c.universe._
    val tpe = weakTypeOf[A]
    val sym = tpe.typeSymbol
    
    def abort(msg: String) = c.abort(c.enclosingPosition, msg)

    def classSym(tpe: Type): ClassSymbol = {
      val sym = tpe.typeSymbol
      if (!sym.isClass) abort(s"$sym is not a class or trait")
      sym.asClass
    }
    
    def isObjectLike(sym: ClassSymbol) = sym.isModuleClass
    
    def isSealedHierarchyCaseClassOrCaseObject(symbol: ClassSymbol): Boolean = {
      def helper(classSym: ClassSymbol): Boolean = {
        //同じファイルから継承されているサブクラスを取得
        classSym.knownDirectSubclasses.toList forall { child0 =>
          println(child0.asClass.isCaseClass)
          val child = child0.asClass
          //再起的に検索
          isObjectLike(child) || child.isCaseClass || (child.isSealed && helper(child))
        }
      }
      symbol.isSealed && helper(symbol)
    }

    val symbol = classSym(tpe)

    c.Expr[Boolean](q"${isSealedHierarchyCaseClassOrCaseObject(symbol)}")
  }
```


実演
________



#まとめ
* 普通にscalaを書く分には触らないであろうマクロだが使いどころを見極めれば業務でも使える？
* 準クォートとweakTypeOfで取得したtype sympolさえ理解できていればそこまで難しくはないかも
参考   
[Martin Odersky教授の準クォートの論文](http://infoscience.epfl.ch/record/185242/files/QuasiquotesForScala.pdf)
[scala Documentation](http://docs.scala-lang.org/ja/overviews/macros/overview.html)


#以下関係ない事
reveal.jsではまった事


デフォルトでsyntax highlightが効かない！
```
<code class="scala">
</code>
```
をindex.htmlに含めないとsyntax highlightが効かない。。

