---
layout: post
title: "Cats Functor"
description: "cats functor"
categories: [scala-Cats]
tags: [scala,스칼라,Cats,cats, 켓츠, Functor, 함자]
redirect_from:
  - /2018/07/25/
---

> Cats Functor .
>

* Kramdown table of contents
{:toc .toc}

# Functor
functor에 관하여는 
[Functor](https://sslee05.github.io/blog/2017/09/10/scala-functor/)를 참조  

# Future 와 참조 투명성
Future는 부작용 계산에 대한 처리를 Future에서 할경우 예상치 못한 결과 즉 side-effect를 가질 수 있다.  
{% highlight scala %}
import scala.util.Random
import scala.concurrent.{Future,Await}
import scala.concurrent.ExecutionContext
import scala.concurrent.duration._

object FutureExplorer extends App {
  
  implicit val ec = ExecutionContext.global
  
  val future01 = {
    
    val r = new Random(0L)
    
    //부작용을 가지는 계산에 대한 처리 
    val x = Future(r.nextInt())
    
    for {
      a <- x
      b <- x
    }yield(a,b)
    
  }
  
  val future02 = {
    val r = new Random(0L)
    for {
      //참조 투명을 검증하기 위해 x에 Future을 대입
      a <- Future(r.nextInt()) 
      b <- Future(r.nextInt())
    }yield(a,b)
  }
  
  println(Await.result(future01, 1 seconds))
  println(Await.result(future02, 1 seconds))
  
}

//(-1155484576,-1155484576)
//(-1155484576,-723955400)
{% endhighlight %}

따라서 Future와 같은 성질의 다른 안전한 참조 투명성을 지닌 다른 유형인 찾아 보면 다음과 같이 Funtion 의 합성을 통하여 연산의 지연으로 Future 와 같은 미래의 연산을 나타 낼 수 있다.  

# Function
fn01: X => A, fn02: A => B 일때 fn02 compose fn01 이면  X => B 의 함수를 얻을 수 있다. 이를 통해 연산을 chain형태로 나타 낼 수 있으며 이때 이는 연산의 표현이며 표현이 곧 실행을 의미 하지는 않는다.  
{% highlight scala %}
val fn01: Int => Double = (i: Int) => i.toDouble
val fn02: Double => Double = (d: Double) => d * 2
val fn03: Double => String = (d: Double) => d.toString
  
//함수의 합성으로 연산의 chain 실제 실행 되지 않는다. 
val fn04: Int => String = fn03 compose fn02 compose fn01
  
//Cats 의 map 으로 표현 
import cats.instances.function._
import cats.syntax.functor._
  
val fn05:Int => String = fn01 map fn02 map fn03
  
// 함수의 변수 자리에 익명함수로 치환하여 참조투명성을 check 
val fn06:Int => String = 
 ((d:Double) => d.toString)  compose ((d:Double) => d * 2 ) compose ((i:Int) => i.toDouble)
val fn07:Int => String = 
  ((i:Int) => i.toDouble) map ((d:Double) => d * 2) map ((d:Double) => d.toString)
  
//실제 실행
println(fn04(5))
println(fn05(5))
println(fn06(5))
println(fn07(5))
{% endhighlight %}
위의 예제를 보면 어떤 단일 함수를 map으로 sequence하게 연결함으로써 어떤 연산의 chain을 만들어 낼 수 있으며 이는 단지 연산의 선언적 표현으로 실제 실행은 되지 않고 언제가 호출시 실행된다는 점에서 Future와 비슷하다고 할 수 있겠다.  

# Higher Kinded type 
Functor, Monad, Applicative Functor 등을 이야기 할때 F\[_\] 이런 형태를 이야기 하지 않을 수 없다.  
우리가 java에서 List\<Integer\>, List\<String\> 등 Integer, String 등의 어떠한 정해지지 않은 유형을 담는 List를 표현할때  List\<A\> 라고 표현한다. Map은 Map\<K,V\> 이렇게.  
그럼 저런 List\<A\>, Map\<K,V\> 처럼 List나 Map 어떠한 유형을 내포하는 유형인데 List인지, Map 인지 정해지지 않은 유형을 표현할때는 어떻게 표현할까?  
이게 고계타입(Higher Kinded type)이다.  
일전에 Functor 게시글에서 이를 Higher Kinded type(고계타입 이라고)라고 하는데 이는 어떠한 type를 유형을 가지는 유형를 표현한 것이다.  


# cats Functor 
## type class
{% highlight scala %}
trait Functor[F[_]] {
  def map[A,B](ma: F[A])(f: A => B):F[B]
}

{% endhighlight %}

api에 보면  companion object의  apply method 은 아래와 과 같다.  
{% highlight scala %}
  def apply[F[_]](implicit functor: Functor[F]): Functor[F]
{% endhighlight %}
위의 코드에서 apply method가 적용될때는 F라는 고계타입에 해당하는 Functor가 암시적으로 받드시 있어야 한다. 따라서 import cats.instance.list._ 처럼 cats instance를 import해야 한다.  
{% highlight scala %}
import cats.Functor
import cats.instances.list._

//object Functor
//def apply[F[_]](implicit instance: Functor[F]): Functor[F]
val functor = Functor[List]
{% endhighlight %}

## Functor type class & instance
기존에 해왔듯이 Functor의 type class 도 같다. Functor type class와 companion object의 apply method가  있고 실제 기본 유형에 대한 instance들은 cats.instances에 있다.  
{% highlight scala %}
import cats.Functor
import cats.instances.list._
  
val xs = List(1,2,3,4,5)
val ys:List[Int] = Functor[List].map(xs)(n => n * 2)
println(ys)
//List(2, 4, 6, 8, 10)

import cats.instances.option._
  
val op01 = Option(123)
val op02:Option[Int] = Functor[Option].map(op01)(n => n * 2)
println(op02)
//Some(246)

{% endhighlight %}

## Functor syntax
위의 Function 의 예제 코드에서  function01 map function02 map function03 처럼 사용했다. 하지만 Function에는 map method가 없다.  
따라서 이것도 Function syntax가 관련될을 것이라 짐작 할 수 있다.  
{% highlight scala %}
import cats.Functor
implicit class FunctorOps[F[_], A](src: F[A]) {
  def map[B](func: A => B)(implicit functor: Functor[F]): F[B] =
    functor.map(src)(func)
}
{% endhighlight %}

List 나 Option등은 기존에 map method 가 있다. 이런경우 List나 Option등의 map method가 적용된다.  
구지 cats.Function 의 map을 사용해하고 싶은 경우 좀 꽁수를 써야 한다.
{% highlight scala %}
def ConvertForth[F[_]](src: F[Int])(implicit F: Functor[F]): F[Int] = 
  src.map(i => i) // cats.syntax.functor._ 가 있어야 된다.
  
val xs = List(1,2,3,4,5)
xs.map(_ * 2) //여기서 map method는 List의 map method 
ConvertForth(xs).map(_ * 2)//여기서의 map method 는 cats.Functor의 map method
//List의 map method가 Functor의 map method이기 때문에 결과 적으로는 같다.
{% endhighlight %}
src가 map method를 사용하려면 cats.syntax.functor._ 가 있어야 된다.  
cats.syntax.functor._ 에는 아마도 위에처럼  FunctorOps 암시자가 있겠지  

# Functor law
Functor law는  다음과 같다.  
Identity: 즉 Functor에 항등함수를 적용하면 원래의 Functor와 같다.  
Composition: 두 함수 f와 g로 합성후 map를 적용한 것은 함수 f를 map적용한 후 다음 함수 g를 map 적용한 것과 동일하다.  
위의 Functor type class, instance, syntax등을 이용하여 functor law예제를 보자  
{% highlight scala %}
val xs = List(1,2,3,4,5)
  
//1. identity law : ma map idFunction = ma
def identityFn[A]: A => A = a => a
  
import cats.Functor
import cats.instances.list._
  
//object Functor
//def apply[F[_]](implicit instance: Functor[F]): Functor[F]
val functor = Functor[List]
  
import cats.syntax.eq._
import cats.instances.int._
  
println(functor.map(xs)(identityFn) === xs)
  
//2. Composition law: 
// ma map f map g == ma map ( f compose g)
  
//Composition Law
val fn01 = (i: Int) => i * 2
val fn02 = (i: Int) => s"[$i]"
  
import cats.instances.string._
import cats.instances.list._
  
import cats.syntax.functor._
//List 의 경우 map method가 있어 List의 map method가 적용된다.
//따라서 다음과 같이 꽁수를 적용 하면 List의 map method적용이 아닌 
//cats.Functor map method가 적용된다.
def ConvertForth[F[_]](src: F[Int])(implicit F: Functor[F]): F[Int] = 
  src.map(i => i) // cats.syntax.functor._ 가 있어야 된다.
  
//functor.map(functor.map(xs)(fn01))(fn02) === 
// ConvertForth(xs).map(fn02 compose fn01)
println((xs map fn01 map fn02) === ConvertForth(xs).map(fn02 compose fn01))
{% endhighlight %}

# Custom Functor 만들기
Functor의 apply method는 암시자로 고계타입의 type parameter를 받는 Functor instance를 받는다.
{% highlight scala %}
def apply[F[_]](implicit F: Functor[F]): Functor[F]
{% endhighlight %}
따라서 custom Functor를 만들경우 implicit F에 해당하는 Functor instance를 암시자로 제공하면 된다.  
{% highlight scala %}
import cats.Functor
implicit val functorOption: Functor[Option] = new Functor[Option] {
  def map[A,B](ma: Option[A])(f: A => B): Option[B] = ma map f
}

import cats.syntax.option._
val optionFct =  Functor[Option].map(1.some)(_ + 2)
{% endhighlight %}
Future에 대한 Functor를 만들경우 좀 고려해야 할 것이 ExecutionContext가 암시자로 필요하다는 것이다. Future에서 암시자를 받기 때문이다.  
따라서 아래 코드와 같이 Context안에  ExecutionContext 암시자가 있어야 한다.
{% highlight scala %}
import cats.Functor

implicit val ec = ExecutionContext.global
implicit def functorFuture(implicit ec: ExecutionContext): Functor[Future] =
  new Functor[Future] {
    def map[A,B](ma: Future[A])(f: A => B): Future[B] = ma map f 
  }

//def apply[F[_]](implicit F: Functor[F]): Functor[F]
//val fctFuture = Functor[Future](functorFuture)
val futureFct = Functor[Future]
{% endhighlight %}

# 연습문제 01
Leaf과 Branch를 가지는 Tree구조의  Functor를 만들어라.  
{% highlight scala %}
sealed trait Tree[+A]
case class Branch[+A](left: Tree[A], right: Tree[A]) extends Tree[A]
case class Leaf[A](value: A) extends Tree[A]

import cats.Functor
import cats.syntax.functor._

implicit val treeFunctor: Functor[Tree] = new Functor[Tree] {
  def map[A,B](ma: Tree[A])(f: A => B): Tree[B] = ma match {
    case Branch(l,r) => Branch(map(l)(f),map(r)(f))
    case Leaf(v) => Leaf(f(v))
  }
}

val left01 = Branch(Leaf(1),Leaf(2))
val right01 = Branch(Leaf(3),Leaf(4))
val br = Branch(left01,right01)//br의 type은 Branch[Int]

//br.map은 not member of Branch[Int]라고 compile error발생 
//이유는 Functor의 type parameter가 invariant이기 때문이다.
val result = br.map(i => s"node value $i") //compile error

//따라서 class명과 같은 소문자로 시작하는 smart constructor를 명시하여 
//return type을 Tree로 명시하는 하나의 방법이 있다.
def branch[A](left: Tree[A], right:Tree[A]): Tree[A] = Branch(left,right)
def leaf[A](a: A): Tree[A] = Leaf(a)

val left01 = branch(leaf(1),leaf(2))
val right01 = branch(leaf(3),leaf(4))
val br = branch(left01,right01)// br의 type은 Tree[Int]
val result = br.map(i => s"node value $i")
{% endhighlight %}


[^1]: This is a footnote.

[kramdown]: https://kramdown.gettalong.org/
[Simple Texture]: https://github.com/yizeng/jekyll-theme-simple-texture
