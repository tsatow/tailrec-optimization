# 末尾再帰最適化とのたたかい

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

## 末尾再帰最適化を考えはじめたきっかけ

* その後、客先でFPinScala読書会がはじまる
* これも6章までは一度解いているので面白くない

=>とりあえず`@annotation.tailrec`付けて遊ぼう！

---

この発表はどんな感じで遊んできたか、の記録です。

たぶん、まだ知見とか共有できるレベルにない。

---

## foldRightの末尾再帰化

末尾再帰最適化の初歩として、foldRightの末尾再帰にしてみる。

* 修正前

```scala
def foldRight[A, B](l: List[A], z: B)(f: (A, B) => B): B = l match {
  case Nil => z
  case Cons(h, t) => f(h, foldRight(t, z)(f))
}
```

* 修正後

```scala
def foldRight[A, B](l: List[A], z: B)(f: (A, B) => B): B = {
  foldLeft(reverse(l), z)(flip(f))
}
```

---

## `foldRight`の末尾再帰化

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

## `flatten`の末尾再帰化

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

じゃあリスト方向とネスト方向で再帰関数をわければいいじゃない

---

