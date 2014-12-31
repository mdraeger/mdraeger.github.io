---
layout: post
title: "Let's Build a Browser Engine in Scala Part 3"
description: "Third post, refine data structure, implement html parsing, test"
category: Functional Programming
tags: [Functional Programming, Scala, Browser Engine]
---
{% include JB/setup %}

Part 3
------

Welcome to the third part of the browser engine in Scala experiment.

In this part, I will implement the html parsing using parser combinators. 
In order to do that, we need to add another dependency to `build.sbt`:
{% highlight scala %}
"org.scala-lang.modules" %% "scala-parser-combinators" % "1.0.2"
{% endhighlight %}

This is actually new with Scala version 2.11; until 2.10 the parser combinators
where part of the base package.

Adapting the DOM classes
------------------------

During implementation of the parser, I realized, that the `Dom` class is 
obsolete. However, I wanted to have the two convenience methods which Leif
implemented in his Haskell version, wrapped in a `Dom` object:
{% highlight scala %}
object Dom {

  type AttributeMap = Map[String, String]
  type Node = NTree[NodeType]

  def elem (name: String, atts: AttributeMap): (List[Node] => Node) = 
    new NTree(Element(ElementData(name, atts)), _)

  def text(t: String): Node = new NTree(Text(t), Nil)
}

{% endhighlight %}

`elem` and `text` just save a lot of type work. The two type declarations 
also save some typing and make compiler messages a lot more readable.

I also realized that in order to test the Nodes for equality, it might help
to have an `equals` method in place. A `toString` method helps to determine
what went wrong, if `equals` failed.

{% highlight scala %}
class NTree[T](val a: T, val children: List[NTree[T]]) {
  override def equals (other: Any) = other match {
    case o: NTree[T] => a == o.a && children == o.children
    case _ => false
  }

  private def str(indent: Int): String = 
    " " * indent + a + '\n' + (children map (_.str (indent + 2)) mkString("\n")) 
  override def toString = str(0)
}

case class Text(text: String) extends NodeType {
  override def toString = "Text: " + text
}

case class Element (data: ElementData) extends NodeType {
  override def toString = "Element: " + (data toString)
}

case class ElementData(tag: String, attributes: Dom.AttributeMap) {
  override def toString = 
    "%s [%s]".format (tag, 
                      attributes map { 
                        case (k, v) => k + '=' + v
                      } mkString(", ")
                     )
{% endhighlight %}

Setting up test cases
---------------------

I'm a fan of writing my tests first. Since I only had to copy Leif's test 
cases, that was even convenient and I could start with reasonable error 
messages right away. In a later version, I will probably put all the test 
values in a separate object, such that I can access them from different 
spec classes.

{% highlight scala %}
  def text = Dom.text (_)
  val testText = "candygram"
  val testP    = "<p ham=\"doctor\">sup</p>"
  val testHtml = """
   <html>
      <head>
         <title>Test</title>
      </head>
      <p class="inner">Hello, <span id="name">world!</span></p>
      <p class="inner">Goodbye!</p>
   </html>"""
  
  "The parser" should "parse the text \"candygram\" correctly" in {
    implicit val parserToTest = parseText
    parsing(testText) should equal (text(testText))
  }

  "The parser" should "parse the element <p ham=\"doctor\">sup</p> correctly" in {
    implicit val parserToTest = parseElement
    val testElem = Dom.elem("p", Map(("ham" -> "doctor")))(List(text("sup")))
    parsing(testP) should equal (testElem)
  }

  "The parser" should """parse the html document 
    <html>
       <head>
          <title>Test</title>
       </head>
       <p class=\"inner\">Hello, <span id="name">world!</span></p>
       <p class=\"inner\">Goodbye!</p>
    </html> correctly""" in {
    implicit val parserToTest = parseHtml

    val dom   = {
      val span  = Dom.elem("span",  Map(("id" -> "name")))    (List(text("world!")))
      val hello = Dom.text("Hello, ")
      val p1    = Dom.elem("p",     Map(("class" -> "inner")))(List(hello, span))
      val p2    = Dom.elem("p",     Map(("class" -> "inner")))(List(text("Goodbye!")))
      val title = Dom.elem("title", Map.empty)                (List(text("Test")))
      val head  = Dom.elem("head",  Map.empty)                (List(title))
    Dom.elem("html",  Map.empty) (List(head, p1, p2))
    }

    parsing(testHtml) should equal (dom)
  }
{% endhighlight %}

Parsing
-------

Scala's parser combinators are pretty comparable to Haskell's Parsec library.
Yet, I needed some help to find an elegant way of putting together all the
pieces.

One piece I got from [this paper](http://www.cs.kuleuven.be/publicaties/rapporten/cw/CW491.pdf), 
another one from the Australian Scala User Group [here](http://www.berniepope.id.au/docs/scala_parser_combinators.pdf).

First, I set up the overarching parser for the document, an element, and a text:

{% highlight scala %}
import scala.util.parsing.combinator._

import org.draegisoft.sonny.{Dom, Element, Text}

class HtmlParser extends RegexParsers {
    import Dom.{AttributeMap, Node}

    override def skipWhitespace = false
    
    def parseHtml: Parser[Node] = (eol *) ~> (space?) ~> parseElement

    def parseElement: Parser[Node] = (space?) ~>
                                     openTag ~ 
                                     (parseNode *) ~ 
                                     endTag <~
                                     (space?) <~
                                     (eol *) >> mkElement
    
    def parseText: Parser[Node] = charData ^^ Dom.text 
{% endhighlight %}
The Scala parser combinator DSL looks pretty straight forward. `parseHtml` 
parses and returns the first html node that it can find. `parseElement` parses 
an opening tag (including the attribute list), a list of child nodes, and a 
matching end tag. The `*` behind a parser is the postFix notation for `rep(_)`, 
the `?` denotes the postfix for `opt(_)`.

A node can either be a text or a more complex element. The `parseNode` parser 
will try the element parser first and only if that fails, it will try the 
`parseText` parser.

{% highlight scala %}
    private def parseNode: Parser[Node] = parseElement | parseText

    private def attributes: Parser[AttributeMap] = 
      (space ~> attribute *)  ^^ (_.toMap)

    private def attribute: Parser[(String, String)] = 
        (name <~ equals) ~ string ^^ {case (k ~ v) => (k -> v)}

    private def openTag = (eol *) ~> "<" ~> name ~ attributes <~ (space?) <~ ">" <~ (eol *)

    private def endTag = (eol *) ~> "</" ~> name <~ (space?) <~ ">" <~ (eol *)

    private def equals       = (space?) ~ "=" ~ (space?)
    private def string       = doubleString | singleString
    private def charData     = "[^<]+".r
    private def space        = ("""\s+""".r *) ^^ {_.mkString}
    private def name         = """(:|\w)(\-|\.|\d|:|\w)*""".r 
    private def doubleString = "\"" ~> "[^\"]*".r <~ "\""
    private def singleString = "'" ~> "[^']*".r <~ "'"
    private def separator    = eoi | eol
    private def eol          = sys.props("line.separator").r
    private def eoi          = """\z""".r
{% endhighlight %}
Regular expressions are a most convenient way of getting the small pieces which
 we want to put together finally. I think, the definitions above are pretty
straight forward. 

`~` combines two parsers and returns both results. `<~` combines two parsers 
and only returns the result of the left parser. `~>` does the same with changed
roles.

{% highlight scala %}
    private def mkElement: (String~AttributeMap~List[Node]~String => Parser[Node]) = {
      case startName ~ atts ~ children ~ endName =>
        if (startName == endName)
          success (Dom.elem (startName, atts)(children))
        else 
          err("tag mismatch")
      }
}
{% endhighlight %}

The method `mkElement` finally creates a node object from a parsed element. Here, we also check that the opening and closing tag are the same. 

Ok, we wrote an html parser in 53 lines. As usual, you can see the full source [here](http://github.com/mdraeger/sonny) and the original posts by Leif are 
[here](https://hrothen.github.io).
