---
layout: post
title: "Let's Build a Browser Engine in Scala Part 2"
description: "Second post, introduce testing"
category: Functional Programming
tags: [Functional Programming, Scala, Browser Engine]
---
{% include JB/setup %}


Test Setup
----------

Welcome to the second part of the browser engine in Scala experiment.
In this part I will already deviate from Leif's original by setting up the 
test environment first. Another deviation is that I don't use unit tests but
specs to perform the test. I just like the spec DSL more.

With Scala and sbt, this is actually pretty simple; we just need to issue 
`test` in the sbt environment. If we use `~ test`, the command will be executed 
every time a source file is altered in the file system. Sbt will then execute
every test it finds in the `src/test` directory.

First, we need to add a dependency to the `build.sbt`:
{% highlight scala %}
"org.scalatest" % "scalatest_2.11" % "2.2.1" % "test"
{% endhighlight %}

Now, we can create a test spec `HtmlParserSpec.scala`:
{% highlight scala %}
import org.draegisoft.sonny.html.HtmlParser

import java.io._
import scala.util.parsing.input.CharSequenceReader
import org.scalatest._

class HtmlParserSpec extends HtmlParser with FlatSpecLike with Matchers{
  
  "The parser" should "parse the text \"candygram\" correctly" in {
    implicit val parserToTest = parseText
    val testText = "candygram"
    parsing(testText) should equal (testText)
  }

{% endhighlight %}

There are several ways to express unit tests, an overview is provided
[here](http://scalatest.org/user_guide/selecting_a_style#styleTraitUseCases).
For me, the `FlatSpec` feels most convenient but you should defnitely check out
other methods.


Convenience methods
-------------------

It is helpful to define a helper method similar to Leif's `parseTest` function.
This method allows to use the `parsing (string) should ...` syntax. The parser
to test is given as an implicit value that has to be defined before the parsing 
method is called. An example is given above.

If the parsing succeeds, the parser result is returned, otherwise an 
`IllegalArgumentException` is thrown. This also allows to test whether a parser
correctly fails at invalid input.

{% highlight scala %}
private def parsing[T](s: String)(implicit p: Parser[T]): T = {
  val phraseParser = phrase(p)
  val input = new CharSequenceReader(s)

  phraseParser(input) match {
    case Success(t, _) => t
    case NoSuccess(msg, _) => 
      throw new IllegalArgumentException(
        "Could not parse '%s': %s".format(s,msg)
      )
  }
}
{% endhighlight %}

The second private method for the parser test is the `assertFail` method that
conveniently allows for testing of invalid input. 

{% highlight scala %}
private def assertFail[T](input: String)(implicit p: Parser[T]) = {
  an [IllegalArgumentException] should be thrownBy { parsing(input)(p)} 
}
{% endhighlight %}

Class under Test
----------------

Finally, I should tell you, which class we are actually testing. For now, it is 
a stub of what I hope to become the Html parser class eventually:

{% highlight scala %}
import scala.util.parsing.combinator._
import org.draegisoft.sonny.{Dom, Element, Text}

class HtmlParser extends JavaTokenParsers {
    
    def parseHtml: Parser[Dom] = ???

    def parseElement: Parser[Element] = ???
    
    def parseText: Parser[Text] = ???
}
{% endhighlight %}

I hope I can show the parser still in 2014, but as usual, I won't promise
anything. The source code you can find [here](https://github.com/mdraeger/sonny).
