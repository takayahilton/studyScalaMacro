# Scalaのマクロ
________



### きっかけ
shapelessを読もうとしていたらmacroだらけで意味不明だったのでmacroを勉強しようと思った。



### macroとは
(scalaでは)コンパイル時に構文木を受け取り、処理できる機能



##利用
macro本体の実装は利用するコードより先にコンパイルされていなければいけないのでsbtで使う時は
プロジェクトを分けて先にマクロ側が先にコンパイルされるようにする。


```scala
lazy val user = (project in file("user"))
  .dependsOn(macroImpl)
```



##macroの種類
* def macro - 普通の関数と同じように呼び出され値を返す。 新しい型を定義したりはできない。
* macro annotation - アノテーションとして利用できるマクロ 新しい型を定義できる。


##def macroの定義
```scala
object MacroImpl{
    def method[A, B](a: A): B = macro impl
    
    def impl[A :c.WeakType](c: Context)(a: c.Expr[A]): c.Expr[B] = {
        c.universe._
        
        val result: B = ???
        
        c.Expr[B](q"$result")
    }
}
//利用者側
    object Main extends App{
        //利用者側からは普通の関数として使える
        MacroImpl.method(a)
    }
```


#まとめ
* macro むずい

