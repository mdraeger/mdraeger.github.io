---
layout: post
title: "Let's Build a Browser Engine in Scala Part 5"
description: "Fifth post, implement styling"
category: Functional Programming
tags: [Functional Programming, Scala, Browser Engine]
---
{% include JB/setup %}

Adding style information
-----------

Welcome to part 5 of my little toy browser engine implementation series. 
Today, I add the style information to the `NTree` resulting in tree of
`StyledNode`.

Before I start, I will add a few declarations and also do some cleanup. First,
I moved all the shared test values from the `Spec` classes into a single object
`TestValues`. This allows for smaller test classes that look a lot cleaner.

Second, it turned out that it is not enough to inherit the `Ordering` trait in
order to offer an implicit ordering on `Selector`. Therefore, I changed the
heredity and created a new object `SelectorOrdering`:

{% highlight scala %}
object SelectorOrdering extends Ordering[Selector]{
  /** 
    * define an ordering on selectors by lexigographically ordering
    * their specificity.
   **/
  def compare (a: Selector, b: Selector) = 
    implicitly[Ordering[Tuple3[Byte,Byte,Byte]]].compare(a.spec, b.spec)
}
{% endhighlight %}

Then I implemented the stub for the `Style` object:

{% highlight scala %}
object Style {
  type PropertyMap = Map[String, Value]
  type StyledNode = NTree[(NodeType, PropertyMap)]
  type MatchedRule = (Selector, Rule)

  def styleTree(root: Node, sheet: Stylesheet): StyledNode = ???
}
{% endhighlight %}

This is enough to implement the test case in `StyleSpec.scala`.

{% highlight scala %}
package org.draegisoft.sonny.test

import org.draegisoft.sonny._
import Style._

import org.scalatest._

import TestValues._

class StyleSpec extends FlatSpec with Matchers{

  "StyleSpec" should "create the styletree out of DOM and CSS" in {
     styletree equals (styleTree(dom, css2))
  }
}
{% endhighlight %}

With the test values: 

{% highlight scala %}
// css declarations for styled dom
val css2 = List( Rule ( List ( Simple ( Some("head"), None, Nil))
                      , List ( Declaration ("margin", Keyword("auto"))
                             , Declaration ("color", (ColorValue(
                                                        Color(0,0,0,255))))))
               , Rule ( List ( Simple ( Some("p"), None, List("inner")))
                      , List ( Declaration ("padding", Length(20, Px())))))

// Styled dom 
val styletree: StyledNode = {
  val rule1 = Map(("margin", Keyword("auto"))
                 , ("color", ColorValue(Color(0,0,0,255))))
  val rule2 = Map(("padding", Length(17, Px())))
  val propEmpty = Map.empty[String, Value]
  val attrEmpty = Map.empty[String, String]
  val test_ = new NTree((Text("Test"), propEmpty), Nil)
  val title = new NTree((Element(ElementData("title", attrEmpty))
                        , propEmpty), List(test_))
  val head = new NTree((Element(ElementData("head", attrEmpty))
                       , rule1), List(title))
  val hello = new NTree((Text("Hello, "), propEmpty), Nil)
  val world = new NTree((Text("world!"), propEmpty), Nil)
  val span = new NTree((Element(ElementData("span", Map(("id", "name"))))
                       , propEmpty), List(world))
  val goodbye = new NTree((Text("Goodbye!"), propEmpty), Nil)
  val p1 = new NTree((Element(ElementData("p", Map(("class", "inner"))))
                     , rule2), List(hello, span))
  val p2 = new NTree((Element(ElementData("p", Map(("class", "inner"))))
                     , rule2), List(goodbye))
  new NTree((Element(ElementData("html", attrEmpty))
            , propEmpty), List(head, p1, p2))
}
{% endhighlight %}

In order to create a tree of `StyledNode` from `Node`, the nodes in the DOM
need to be mapped to the new type. In order to do that, I needed a map method:

{% highlight scala %}
def map[A] (f: T => A): NTree[A] = 
  new NTree(f(a), children map { t => t map (f) })
{% endhighlight %}

Of course, the type `T` of `NTree` cannot be invariant anymore.

Now, we can implement the `styleTree` method in terms of `map`:

{% highlight scala %}
def styleTree(root: Node, sheet: Stylesheet): StyledNode = {
  def style (n: NodeType): (NodeType, PropertyMap) = n match {
    case Text(_) => (n, Map.empty)
    case Element(data) => (n, specifiedValues(data, sheet))
  }
  // 'style' the dom using the supplied stylesheet 
  root map (style)
}

{% endhighlight %}

The method `specifiedValues` orders the rules that match an element by their
specificity and returns a property map for easy access.

{% highlight scala %}
private def specifiedValues( data: ElementData
                           , sheet: Stylesheet): PropertyMap = {
  val sortedRules = matchingRules(data, sheet).sortBy(_._1)(SelectorOrdering) 
  def expand(matchedRule: MatchedRule): List[(String, Value)] = matchedRule._2 match {
    case Rule(_, declarations) => declarations map { 
                                    case Declaration(name, value) => (name -> value) }}
  (sortedRules map (expand) flatten) toMap
}
{% endhighlight %}

The rest is easy, though not necessarily pretty. For now, only simple selectors
are implemented.

{% highlight scala %}
private def matchingRules(data: ElementData, 
                          sheet: Stylesheet): List[MatchedRule] = 
  sheet.foldLeft (List.empty[MatchedRule]) { 
    (list: List[MatchedRule], rule: Rule) => matchRule (data, rule) match {
          case Some(matchedRule) => matchedRule :: list
          case None => list
        }
  }

private def matchRule(data: ElementData, rule: Rule): Option[MatchedRule] = {
  val matchingSelector = (rule selector) find (s => matches(data, s))
  matchingSelector map (s => (s, rule))
} 
{% endhighlight %}

Now it is only left to decide whether a selector matches an element or not: 

{% highlight scala %}
private def matches(data: ElementData, selector: Selector): Boolean = 
  selector match {
    case Simple(_, _, _) => matchSimple(data, selector)
  }

private def matchSimple(data: ElementData, simple: Selector): Boolean =
  simple match {
    case Simple (None, None, c) => matchClasses(data, c)
    case Simple (Some(n), None, c) => matchNames(data, n) && 
                                      matchClasses(data, c)
    case Simple (None, Some(i), c) => matchId(data, i) && 
                                      matchClasses(data, c)
    case Simple (Some(n), Some(i), c) => matchNames(data, n) &&
                                         matchId(data, i) &&
                                         matchClasses(data, c)
  }

private def matchNames(data: ElementData, name: String): Boolean = 
  name == data.tag

private def matchId(data: ElementData, id: String): Boolean = 
  id == data.findID

private def matchClasses( data: ElementData
                        , classes: List[String]): Boolean = {
  val cls: Set[String] = data.classes
  classes match {
    case Nil => true
    case (c::cs) => (data.classes contains (c)) && matchClasses(data, cs)
  }
}
{% endhighlight %}

This concludes the fifth part of the sonny implementation. Next time, I shall 
work on boxes. As usual, you can see the full source 
[here](http://github.com/mdraeger/sonny) and the original posts by Leif are 
[here](https://hrothen.github.io).
