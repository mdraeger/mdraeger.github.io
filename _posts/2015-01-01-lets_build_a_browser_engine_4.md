---
layout: post
title: "Let's Build a Browser Engine in Scala Part 4"
description: "Fourth post, introduce Css"
category: Functional Programming
tags: [Functional Programming, Scala, Browser Engine]
---
{% include JB/setup %}

Css parsing
-----------

Welcome. As it turned out, the first day of the new year was not as busy as 
expected. This gave me the opportunity to implement the next step on the way
to Sonny, my toy browser engine.

Today, I implement a simple subset of CSS, namely I want to parse a stylesheet
like this on:

{% highlight css %}
h1, h2, h3 {margin: auto; color: #cc0000; }
div.note { margin-bottom: 20px; padding: 10px; }
#answer { display: none; }
{% endhighlight %}

Like before, I start with writing a stub for the parser and the data structure.As you can see, I changed the inheritance hierarchy for the parser a bit. Since
a couple of the parsers will be the same for CSS and HTML parsing, I decided to
have a common ancestor for both parsers. That made the parser classes a little
smaller and helped to focus on the essential parts.

{% highlight scala %}
class CssParser extends SonnyParser{
   type Stylesheet = List[Rule]

   def parseCss: Parser[Stylesheet] = ???
}

/** A rule is a list of selectors that share a list of declarations */
case class Rule(val selector: List[Selector], 
                val declaration: List[Declaration])

/** Selectors are ordered by their specificity. */
abstract class Selector extends Ordering[Selector]{
  type Specificity = (Byte, Byte, Byte)

  def spec: Specificity

  /** 
    * define an ordering on selectors by lexigographically ordering
    * their specificity.
    */
  def compare (a: Selector, b: Selector) = 
    implicitly[Ordering[Tuple3[Byte,Byte,Byte]]].compare(a.spec, b.spec)
}

/** For now we stick with simple selectors only. */
case class Simple (val tag: Option[String],
                   val id:  Option[String],
                   val cls: List[String]) extends Selector {

  /** calculate the specificity of this selector */
  override def spec = (id map (_.length.toByte) getOrElse (0),
                       cls.size.toByte,
                       tag map  (_.length.toByte) getOrElse (0))
  
  override def toString = "Tag: %s, Id: %s, Classes: [%s]".
                           format(tag, id, cls mkString (", "))
}

/** an empty selector */
object NilS extends Simple(None, None, Nil){
  def apply() = new Simple(None, None, Nil)
}

case class Declaration(name: String, value: Value)

abstract class Value

case class Keyword(val keyword: String) extends Value {
  override def toString = "Keyword: " + keyword
}

case class Length(val value: Float, val unit: Unit) extends Value {
  override def toString = "Length: %f %s".format (value, unit)
}

case class ColorValue(val value: Color) extends Value{
  override def toString = "Color: " + value
}

abstract class Unit

/** Only one unit for now */
case class Px() extends Unit {
  override def toString = "Px"
}

/** 
* There is no unsigned byte type in Scala. Therefore, we use short
* and impose range restrictions.
*/
case class Color(val r: Short, val g: Short, val b: Short, val a: Short){
  require (r >= 0 && r < 256 && 
           g >= 0 && g < 256 && 
           b >= 0 && b < 256 && 
           a >= 0 && a < 256)  

  override def toString = "#%2h%2h%2h%2h".format(r, g, b, a)
}
{% endhighlight %}

I also added a bunch of `toString` methods to allow for easier debugging. 
That is something the Haskell just does a lot better by deriving a readable
string representation automatically.

Having done that part, I'm ready to set up the test case. I took Leif's test
without any changes for that.

{% highlight scala %}

class CssParserSpec extends CssParser with FlatSpecLike with Matchers{

  val testCss = """
h1, h2, h3 {margin: auto; color: #cc0000; }
div.note { margin-bottom: 20px; padding: 10px; }
#answer { display: none; }
"""

  val testSheet = List(Rule( List( Simple (Some("h1"), None, Nil)
                                 , Simple (Some("h2"), None, Nil)
                                 , Simple (Some("h3"), None, Nil))
                           , List( Declaration("margin", Keyword("auto"))
                                 , Declaration("color", ColorValue(
                                               Color(204,0,0,255)))))
                      ,Rule( List( Simple (Some("div"), None, "note"::Nil))
                           , List( Declaration("margin-bottom", Length(20, Px()))
                                 , Declaration("padding", Length(10, Px()))))
                      ,Rule( List( Simple (None, Some("answer"), Nil))
                           , List( Declaration("display", Keyword("none")))))
  
  "The parser" should "parse valid css correctly" in {
    implicit val parserToTest = parseCss
    parsing(testCss) should equal (testSheet)
  }
}
{% endhighlight %}

Now, we can start to implement the actual parser. This doesn't provide any more
insight than the [HTML parser](/functional%20programming/2014/12/31/lets_build_a_browser_engine_3/), therefore I will not comment on it very much.

{% highlight scala %}
package org.draegisoft.sonny

import scala.util.parsing.combinator._

class CssParser extends SonnyParser{
   type Stylesheet = List[Rule]

   def parseCss: Parser[Stylesheet] = (eol?) ~> 
                                      repsep(rule, (space?) ~ (eol?)) <~
                                      (eol?)

   private def rule: Parser[Rule] = 
      (repsep(selector, ',' ~ (space?)) <~ (space?)) ~ 
       declarations ^^ 
         { case selectors ~ declarations => Rule(selectors, declarations) }

   private def selector: Parser[Selector] = ???
            
   // supply string tags to distinguish the parser results.
   private def class_ = "." ~> ident ^^ { c => ("class", c) }
   private def id = '#' ~> ident ^^ { i => ("id", i) }
   private def univ = '*' ^^ { _ => "univ" }
   private def tag = ident ^^ { t => ("tag", t) }

   private def ident = """[*@_]?-?[a-zA-Z_][a-zA-Z0-9-_]*""".r

{% endhighlight %}

I think the `selector` parser requires some explanation, though. If someone
happens to know a better way, I'm happy to learn it. The problem that I 
encountered was that I didn't know, which parser succeeded in the parser 
combination. Therefore, I added a string tag to the four component parsers
and added a pattern match to the `selector` parser. From here, I use a 
`foldLeft` to generate the final selector. Leif used the `StateT` monad for 
that, in the original Rust implementation a mutable object is used. Since I wanted to avoid mutable state, this was the best I could come up with.

{% highlight scala %}
   private def selector: Parser[Selector] = 
      rep(class_ | id | univ | tag) ^^ { 
         case l => l.foldLeft(NilS()) {
           case (simple, "univ") => simple
           case (Simple(n, _, cs), ("id", i: String)) => Simple(n, Some(i), cs)
           case (Simple(_, i, cs), ("tag", t: String)) =>
                   Simple(Some(t), i, cs)
           case (Simple(n, i, cs), ("class", c: String)) => 
                   Simple(n, i, cs ++ List(c))
           case _ => NilS()
         }
      }
{% endhighlight %}

That's it for the selectors. Next, the declarations. These are actually pretty
straight forward, since the parts are easilly composable.

{% highlight scala %}
   private def declarations: Parser[List[Declaration]] = 
     '{' ~> repsep(declaration, (space?)) <~ (space?) <~ '}'

   private def declaration: Parser[Declaration] = 
     ((space?) ~> ident <~ (space?) <~ ':' <~ (space?)) ~
     (value <~ (space?) <~ ';') ^^ { case n ~ v => Declaration(n, v) }

   private def value: Parser[Value] = len | color | keyword

   private def len: Parser[Value] = float ~ unit ^^ { case f~u => Length(f,u) }

   // use the Java float parser to read the value
   private def float: Parser[Float] = """[\d\.]+""".r ^^ 
                                      { java.lang.Float.parseFloat(_) }

   private def unit: Parser[Unit] = "(p|P)(x|X)".r ^^ { case _ => Px() }
   
   private def color: Parser[Value] = '#' ~> """[0-9a-fA-F]{6}""".r ^^
     {case s =>
       val r = java.lang.Short.parseShort(s take (2), 16)
       val g = java.lang.Short.parseShort((s drop (2)) take (2), 16)
       val b = java.lang.Short.parseShort((s drop (4)) take (2), 16)
       ColorValue(Color(r, g, b, 255)) 
     }

   private def keyword: Parser[Value] = ident ^^ { case s => Keyword(s) }
}
{% endhighlight %}


Ok, we wrote a CSS parser in 55 lines. As usual, you can see the full source [here](http://github.com/mdraeger/sonny) and the original posts by Leif are 
[here](https://hrothen.github.io).
