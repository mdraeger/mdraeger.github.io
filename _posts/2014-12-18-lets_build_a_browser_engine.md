---
layout: post
title: "Let's Build a Browser Engine in Scala"
description: "First post of what hopefully turns into a series"
category: Functional Programming
tags: [Functional Programming, Scala, Browser Engine]
---
{% include JB/setup %}

Based on Matt Brubeck's toy browser engine in Rust, Leif Grele has started
a series of [blog posts](http://hrothen.github.io/2014/09/05/lets-build-a-browser-engine-in-haskell/),
implementing a toy browser engine in Haskell.

Here, I try to set up something similar in Scala. To be honest, I'm really 
curious how far I might get with that. 

Build Environment
-----------------

Since we are using Scala, the obvious choice for a build tool is 
[Sbt](http://www.scala-sbt.org/index.html). Therefore, we set up a `build.sbt`
with following content:

{% highlight scala %}
lazy val commonSettings = Seq(
    organization := "org.draegisoft",
    version := "0.1.0",
    scalaVersion := "2.11.4"
)

lazy val root = (project in file(".")).
    settings(commonSettings: _*).
    settings(
        name := "sonny"
    )
{% endhighlight %}

Getting started: Setting up the DOM
-----------------------------------

Similar to the original blog post, we introduce the DOM as a tree of Nodes. 
We basically use a data structure similar to the original blog post. However,
it will be revisited and refined as the requirements grow.

First, let's define the tree structure:

{% highlight scala %}
case class Dom(val node: NTree[NodeType])

class NTree[T](a: T, children: List[NTree[T]]) 
{% endhighlight %}

We already stated, that the nodes in the DOM will be of type NodeType, so 
we should tell the compiler, what that means:

{% highlight scala %}
sealed class NodeType

case class Text(text: String) extends NodeType
case class Element (data: ElementData) extends NodeType
{% endhighlight %}

Next, define what `ElementData` might be. We restrict that to a tag name and 
a `Map` of attributes. Leif uses Haskell's `Maybe` type to deal with the fact 
that a value may not be present in the attributes map. Scala offers for that
purpose the `Option` data type, which is conveniently returned by `Map`'s get
method.

{% highlight scala %}
 case class ElementData(tag: String, attributes: Map[String, String]) {
   def findID = findAttr ("id")
   def findAttr (attr: String) = attributes get (attr)
   def classes = (findAttr ("class")) match {
     case None => Set.empty
     case Some (s) => s.split(' ').toSet
   }
 }
{% endhighlight %}

Leif also defined two convenience functions `text` and `elem` to have an easier 
access to constructing the node types. I don't want to do that now, but I might
change my mind later.

For now, that's all I want to do about the DOM. The full source can be found 
[here](https://github.com/mdraeger/sonny).
