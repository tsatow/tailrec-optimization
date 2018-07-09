# 末尾再帰最適化であそぼう

---

## 自己紹介

* 名前: 佐藤 貴比呂(SIer => atWare)
* Twitter: Satoooooooooooo(o×12)
* ブログ: [少年易酔学難成](http://satoooooooooooo.hatenablog.com/)
* Scala歴1年

---

## きっかけ

* 昨年大晦日にS99を解いていた
* P20くらいまでは簡単で面白くない

=>とりあえず`@annotation.tailrec`付けて遊ぼう！

---

## きっかけ

* その後、客先でFPinScala読書会がはじまる
* これも6章までは一度解いているので面白くない

=>とりあえず`@annotation.tailrec`付けて遊ぼう！

---

## foldRightの末尾再帰化(1)

末尾再帰最適化の初歩として、foldRightの末尾再帰にしてみる。

```scala
// 修正前
def foldRight[A, B](l: List[A], z: B)(f: (A, B) => B): B = l match {
  case Nil => z
  case Cons(h, t) => f(h, foldRight(t, z)(f))
}

// 修正後
def foldRight[A, B](l: List[A], z: B)(f: (A, B) => B): B = {
  foldLeft(reverse(l), z)(flip(f))
}
```

---

## `foldRight`の末尾再帰化(2)

foldLeftが末尾再帰なおかげでfoldRightも簡単に最適化できる。

```scala
def flip[A, B](f: (A, B) => B): (B, A) => B = (b, a) => f(a, b)

def reverse[A](as: List[A]): List[A] = {
  foldLeft(as, Nil: List[A])((b, a) => Cons(a, b))
}

@annotation.tailrec
def foldLeft[A, B](l: List[A], z: B)(f: (B, A) => B): B = l match {
  case Nil => z
  case Cons(h, t) => foldLeft(t, f(z, h))(f)
}
```

---

## `flatten`の末尾再帰化

* S99のP07より。
* 普通のflattenと違いリストのネスト具合が一意ではない。

```
P07 (**) Flatten a nested list structure.
Example:
scala> flatten(List(List(1, 1), 2, List(3, List(5, 8))))
res0: List[Any] = List(1, 1, 2, 3, 5, 8)
```

---

## `flatten`の末尾再帰化(1)

リスト要素とネスト方向の二方向に再帰が発生するので末尾再帰化できない><

```scala
def flatten(list: List[Any]): List[Any] = {
  // 末尾再帰にできぬ...できぬのだ!!
  def go(acc: List[Any], maybeList: Any): List[Any] = maybeList match {
    // Listの場合
    case e :: rest => go(go(acc, e), rest)
    case Nil       => acc
    // それ以外
    case e         => e :: acc
  }

  P05.reverse(go(Nil, list)) // P05では末尾再帰のreverseを定義している
}
```

---

## flattenの末尾再帰化(2)

じゃあリスト方向とネスト方向で再帰関数をわければいいじゃない

---

## flattenの末尾再帰化(3)

```scala
@annotation.tailrec
def flatten(list: List[Any]): List[Any] = {
  // もちろん、addとかaddAllは自前で末尾再帰化したコードを用意している
  @annotation.tailrec
  def go(acc: List[Any], list: List[Any]): List[Any] = list match {
    case (l: List[Any]) :: rest => go(addAll(acc, l), rest)
    case e :: rest              => go(add(acc, e), rest)
    case Nil                    => acc
  }

  if (forall(list, (e => !e.isInstanceOf[List[Any]]): Any => Boolean)) {
    list
  } else {
    flatten(go(Nil, list))
  }
}
```

---

## 二分木の畳み込み(1)

* FPinScalaのExercise3.29の問題
* 末尾再帰化処理はこんな感じ

```scala
sealed trait Tree[+A]
case class Leaf[A](value: A) extends Tree[A]
case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]

def fold[A, B](a: Tree[A])(f: A => B)(g: (B, B) => B): B = a match {
  case Branch(l, r) => g(fold(l)(f)(g), fold(r)(f)(g))
  case Leaf(v)      => f(v)
}
```

---

## 二分木の畳み込み(2)

これを末尾再帰化するにあたっての問題

* 次の再帰に持っていきたいデータが二つある
  - 未捜査のツリー
  - 部分木の計算結果
* これをどのように保持するか...

---

## 二分木の畳み込み(3)

```scala
def fold[A, B](tree: Tree[A])(f: A => B)(g: (B, B) => B): B = {
  @annotation.tailrec
  def eval(stack: List[Either[B, Tree[A]]]): List[Either[B, Tree[A]]] = stack match {
    case Cons(Left(e1), Cons(Left(e2), rest))   => eval(Cons(Left(g(e2, e1)), rest))
    case Cons(Left(e1), Cons(r@Right(_), rest)) => Cons(r, Cons(Left(e1), rest)) // 呼出元で木を取り出しやすいよう入れ替え
    case _                                      => stack
  }
  @annotation.tailrec
  def go(tree: Tree[A], stack: List[Either[B, Tree[A]]]): B = tree match {
    case Leaf(v) =>
      // 計算済みの値がstackに連続して溜まると未計算の木が取り出せないので出来るだけ計算を進める
      eval(Cons(Left(f(v)), stack)) match {
        case Cons(Right(next), rest) => go(next, rest)
        case Cons(Left(e1), _)       => e1 // 最後まで評価が終わっているはず
        case Nil                     => f(v)
      }
    case Branch(l, r) => go(l, Cons(Right(r), stack))
  }
  go(tree, Nil)
}
```
