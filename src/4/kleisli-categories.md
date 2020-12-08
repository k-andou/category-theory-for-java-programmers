# クライスリ圏
　これまで型と純粋関数を圏としてモデル化する方法を見てきた。圏論で副作用、また非純粋関数をモデル化する方法があることも触れた。次のような例：実行時のログまたはトレースをする関数を見てみよう。javaでは次のようにグローバルに状態を変更して実装するだろう。
```java
String logger;

Boolean negate(Boolean b) {
  logger += "Not so! ";
  return !b;
}
```
参照透過性（同じ入力に対して同じ作用と同じ出力とを持つこと）が成り立たないので、この関数は純粋関数ではない。
　現代では、グローバルな状態を変更するのをできるだけ避けようとする(並列の複雑さのため)。ライブラリではこのようなコードは決して使われない。
　幸いにも、このような関数を純粋関数にすることができる。引数と戻り値で明示的にログを通過させればよい。文字列の引数を追加して、更新されたログを含む文字列とペアにして戻せばよい。
```java
class Pair<A, B> {
  private final A left;
  private final B right;

  Pair(A left, B right) {
    this.left = left;
    this.right = right;
  }
}

Pair<Boolean, String> negate(Boolean b, String logger){
  return new Pair<>(!b, logger + "Not so! ");
}
```
この関数は純粋で、副作用を持たず、同じ引数で呼び出されれば毎回同じペアを返し、必要なら記憶することもできる。けれども、ログは累積していくものであることを考慮すると、ある呼び出しを引き起こすことができるようなすべての可能な履歴を記憶しなければならない。次のようにそれぞれ別々のエントリになる。
```java
negate(true, "It was the best of times. ");
negate(true, "It was the worst of times. ");
```
　また、ライブラリでのインターフェースとしても余り良くない。呼び出される関数は戻り値の型のその文字列を無視したいが、引数としての文字列を通過させなければならず、便利ではない。
　ログの追加するのをより軽減して同じことをする方法はないか？関心事を分離すること方法はないか？この単純な例では、関数の主な目的はあるブール値を別のブール値に変換することである。ログを残すのは副次的なことである。その関数で特定されるメッセージは決まっているが、ある連続するログの中にそのメッセージを追加する処理と別の関心事である。その関数が文字列を生成したいが、ログを生成することの負担は避けたい。妥協すると以下のようになる。
```java
Pair<Boolean, String> negate(Boolean b) {
  return new Pair<>(!b, "Not so! ");
}
```
ログは関数の呼び出しの間で集約されるだろうと言う発想である。
　もう少し現実的な例を見てみよう。小文字を大文字に変換する関数と文字列を空白の区切り文字で文字列の配列に分割する関数で取り上げる。
```java
String toUpper(String s) {
  return s.toUpperCase();
}
String[] toWords(String s) {
  return s.split(" ");
}
```
関数toUpperとtoWordsを変更することで、正規な戻り値の最上位にメッセージ文字列を上に載せるようにしたい。これらの関数の戻り値を「装飾する」。次のように総称型でWriterを定義すれば、最初の任意の型Aの値と二つ目の文字列型の値のペアをカプセル化する。
```java
class Writer<A> extends Pair<A, String> {
  Writer(A v, String s) {
    super(v, s);
  }
}
```
これを使って装飾する。
```java
Writer<String> toUpper(String s) {
  String result = s.toUpperCase();
  return new Writer(result, "toUpper ");
}
Writer<String[]> toWords(String s) {
  String[] result = s.split(" ");
  return new Writer(result, "toWords ");
}
```
これらの二つの関数を合成して、別の装飾された関数、を大文字にして単語に分解する、そしてこれらの操作のログを生成する関数にしたい。それは次のようにするかもしれない。
```java
Writer<String[]> process(Stirng s) {
  var p1 = toUpper(s);
  var p2 = toWords(p1.left);
  return new Writer(p2.left, p1.right + p2.right);
}
```
我々が達成したいゴールは次のようなものである。ログの集約は個々の関数の関心事ではない。これらはそれぞれのメッセージを生成し、その後このメッセージはより大きなログに外部で連結される。
　このようなスタイルのプログラム全体を想像して欲しい。繰り返しと間違いをおこしやすいコードの悪夢である。しかし我々はプログラマーである。繰り返しのコードを扱う方法を知っている。それを抽象化すればよい。けれども、これはありふれた抽象化ではない。関数の合成を抽象化するのである。合成は圏論の核心である。より多くのコードを書く前に、圏論の視点からその問題を解析してみよう。

## Writer圏
 追加機能を上に載せるために一連の関数の戻り値を装飾する発想はとても良い結果をうむことがわかる。そのような例をこれから提示していくつもりである。出発点は型と関数の正則圏である。対象としての型の話題を置いておき、装飾された関数で射を再定義する。
 例えば、`Integer`から`Boolean`を得る`isEven`と言う関数を装飾してみよう。この関数は装飾された関数で表される射に変化する。重要な点は、この射が`Integer`と`Boolean`の対象の間での矢印として依然として見なせることである。この装飾された関数が次のようにペアを返すとしてもである。
```java
Pair<Boolean, String> isEven(Integer n){
  return new Pair<>(n % 2 == 0, "isEven ");
}
```
圏の法則により、この射は`Boolean`の対象から任意の対象を得る別の射と合成出来るはずである。特に、次のような先に述べた`negate`で合成できるはずである。
```java
Pair<Boolean, String> negate(Boolean b){
  return new Pair<>(!b, "Not so! ");
}
```
明らかにこれらの射は通常の関数と同じように合成することは出来ない。入力と出力が一致しないからである。これらの合成は次のようなものになるべきである。
```java
Pair<Boolean, String> isOdd(Integer n) {
  Pair<Boolean, String> p1 = isEven(n);
  Pair<Boolean, String> p2 = negate(p1.left);
  return new Pair<>(p2.left, p1.right + p2.right);
}
```
ここで行った圏論で２つの射の合成の手順は次のようなもので構成されている。
 1. 最初の射に対応する装飾された関数を実行する。
 1. 結果のペアの最初の要素を取り出して、２番目の射に対応する装飾された関数に渡す。
 1. 最初の結果の２番目の要素（文字列）と２番目の結果の２番目の要素（文字列）を連結する。
 1. 最後の結果の最初の要素と連結した文字列を一緒にして新たなペアを作って戻す。
