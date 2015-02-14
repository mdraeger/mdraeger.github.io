---
layout: post
title: "Let's Build a Browser Engine in Scala Part 6"
description: "Sixth post, implement boxes"
category: Functional Programming
tags: [Functional Programming, Scala, Browser Engine]
---
{% include JB/setup %}

Implementing the box layout
-----------

Welcome to part 6 of my little toy browser engine implementation series. 
This time, I will implement the box layout. As usual, just a small subset
of the actual html/css layout will be supported. 

To get it all started, let's define a few helpful classes and objects to 
hold the information that we want to pass around.

{% highlight scala %}
/* 
Dimensions class that defines the box dimensions.
*/
case class Dimensions( val x:       Float
                     , val y:       Float
                     , val width:   Float
                     , val height:  Float
                     , val padding: EdgeSize
                     , val border:  EdgeSize
                     , val margin:  EdgeSize)

case class EdgeSize( val left:   Float
                   , val right:  Float
                   , val top:    Float
                   , val bottom: Float)

sealed abstract class BoxType

case class BlockNode( val styledElement: Layout.StyledElement) extends BoxType
case class InlineNode( val styledElement: Layout.StyledElement) extends BoxType
case object AnonymousBlock extends BoxType

// Default edge size
object EmptyEdge extends EdgeSize(0,0,0,0)

// default dimensions
object DefaultDim extends Dimensions(0,0,0,0, EmptyEdge, EmptyEdge, EmptyEdge)
{% endhighlight %}

The layout object contains all the functions that build the layout tree. 
In order to stay close to the original Haskell implementation, I avoid the 
object oriented facilities of Scala. At some time in the future I might 
make up my mind about that. 

{% highlight scala %}
object Layout {
  type LayoutBox = NTree[(Dimensions, BoxType)]
  type StyledElement = (NodeType, PropertyMap)

  def buildLayoutTree(root: StyledNode): Either[String, LayoutBox] = {
    def addDim(b: BoxType) = (DefaultDim, b)

    def blt(node: StyledNode): NTree[BoxType] = {
      // utility function, mimicking Haskell's Data.List.groupBy
      def hGroupBy[A](f: A=> (A => Boolean), list: List[A]): List[List[A]] =
        list match {
          case Nil => Nil
          case x :: xs =>
            val t = xs span f(x)
            (x :: t._1) :: hGroupBy(f, t._2)
        }

      def anonify(boxNodes: List[NTree[BoxType]]): List[NTree[BoxType]] = {
        val groupCriterion: NTree[BoxType] => NTree[BoxType] => Boolean = 
          s1 => s2 => isInline(s1) && isInline(s2)
        hGroupBy(groupCriterion, boxNodes) flatMap(mergeInlines)
      }

      def mergeInlines(nodes: List[NTree[BoxType]]): List[NTree[BoxType]] = {
        if (isInline(nodes.head)) List(NTree(AnonymousBlock, nodes)) else nodes
      }

      def isInline(node: NTree[BoxType]): Boolean = node match {
        case NTree(InlineNode(_), _) => true
        case _                       => false
      }

      val (nd, cs) = node match {
        case NTree(nd, cs) => (nd, cs)
      }

      val ns_ = cs.filter(c => display(c) != DisplayNone).map(blt)

      val (n, ns) = display(node) match {
        case Block  => (BlockNode(nd), anonify(ns_))
        case Inline => (InlineNode(nd), ns_)
        case _      => ??? // can never happen
      }

      NTree(n, ns)
    }

    display(root) match {
      case Block       => Right(blt(root).map(addDim))
      case Inline      => Right(blt(root).map(addDim))
      case DisplayNone => Left("error: root node has display:none")
    }
  }

  def layout(box: LayoutBox, dim: Dimensions): Either[String, LayoutBox] = {
    val NTree((_, boxType), _) = box
    boxType match {
      case BlockNode(_) => layoutBlock(dim, box)
      case InlineNode(_) => ???
      case AnonymousBlock => ???
    }
  }

  def layoutBlock(dim: Dimensions, root: LayoutBox): Either[String, LayoutBox] = {
    def errorString: String => Either[String, LayoutBox] = s => Left(s)
    
    calcWidth(dim, root).fold(
      errorString, box => calcPosition(dim, box)).fold(
        errorString, layoutChildren _).fold(
          errorString, calcHeight _)
  }

  // utility function to retrieve the style node
  def getStyledElem(node: LayoutBox) = {
    val box = node match { case NTree((_, box), _) => box }
    box match {
      case BlockNode(s) => Right(NTree(s, Nil))
      case InlineNode(s) => Right(NTree(s, Nil))
      case AnonymousBlock => Left("Error: attempted to access the nonexistant StyleNode of an AnonymousBlock")
    }
  }

  // get the total height of an element
  def marginBoxHeight(dim: Dimensions) = {
    val Dimensions(_, _, _, height, pad, bor, mar) = dim
    List(height, pad.top, pad.bottom, bor.top, bor.bottom, mar.top, mar.bottom).sum
  }
}
{% endhighlight %}

The method `layoutBlock` shows how you can get around the fact that Scala 
lacks a Monad implementation. However, that only works with the `Either` 
type and still looks a little awkward. Also, the left option's type has to be 
the same for all functions to allow the error function to be reused.

The function `calcWidth` looks rather painful. 

{% highlight scala %}
def calcWidth(dim: Dimensions, box: LayoutBox) = {
   val NTree((boxDim, boxType), children) = box
   getStyledElem(box) match {
     case Left(s:String) => Left(s)
     case Right(style) =>
       val auto = Keyword("auto")
       val zero = Length(0, Px)
       val w = value(style, "width") getOrElse auto
       val vals = List( List("margin-left",        "margin")
                      , List("margin-right",       "margin")
                      , List("border-left-width",  "border-width")
                      , List("border-right-width", "border-width")
                      , List("padding-left",       "padding")
                      , List("padding-right",      "padding")).map(
                    keys => lookup(style, keys, zero))
       val total = (w :: vals).map(toPx).sum
       val underflow = dim.width - total

       def checkUnderflow(w: Value, mlf_mrt: (Value, Value)) = {
         val (mlf, mrt) = mlf_mrt
         (w == auto, mlf == auto, mrt == auto) match {
           case (false, false, false) => (w, mlf, Length(toPx(mrt) + underflow, Px))
           case (false, false, true) => (w, mlf, Length(underflow, Px))
           case (false, true, false) => (w, Length(underflow, Px), mrt)
           case (false, true, true) => (w, Length(underflow/2, Px), Length(underflow/2, Px))
           case (true, a, b) => 
             val l = if (a) zero else mlf
             val r = if (b) zero else mrt
             if (underflow >=0 ) (Length(underflow, Px), l, r)
             else (zero, l, Length(toPx(r) + underflow, Px))
         }
       }

       def checkAutoMargins(x: Value, y: Value) = {
         def check(a: Value) = if (a == auto) zero else a

         if (w != auto && total > dim.width) (check(x), check(y))
         else (x,y)
       }

       val (ml__ :: mr__ :: Nil, vals_) = vals splitAt(2)
       val (w_, ml_, mr_) = checkUnderflow(w, checkAutoMargins(ml__, mr__))

       val List(w__, ml, mr, blw, brw, plf, prt) = 
         (List(w_, ml_, mr_) ++ vals_) map(toPx)

       def updateDim(d: Dimensions) = {
         val pad = d.padding
         val mar = d.margin
         val bor = d.border
         val newPadding = EdgeSize(plf, prt, pad.top, pad.bottom)
         val newMargin = EdgeSize(ml, mr, mar.top, mar.bottom)
         val newBorder = EdgeSize(blw, brw, bor.top, bor.bottom)
         Dimensions(d.x, d.y, w__, d.height, newPadding, newBorder, newMargin)
       }

       Right(NTree((updateDim(boxDim), boxType), children))
    }
}
{% endhighlight %}

The rest doesn't look that bad. The function `layoutChildren` is an example
how bad it can look, if there is no proper Monad implementation. The `scalaz` 
package provides such an implementation but I haven't looked into that yet.
Furthermore, I wanted to implement this toy project with as few additional
libraries as possible and so far, I can take the pain.

{% highlight scala %}
def calcPosition(dim: Dimensions, box: LayoutBox) = {
 val NTree((boxDim, boxType), children) = box
 getStyledElem(box) match {
   case Left(s:String) => Left(s)
   case Right(style) => 
     val zero = Length(0, Px)
     val vals = List( List("margin-top",          "margin")
                    , List("margin-bottom",       "margin")
                    , List("border-top-width",    "border-width")
                    , List("border-bottom-width", "border-width")
                    , List("padding-top",         "padding")
                    , List("padding-bottom",      "padding")).map(
                  keys => lookup(style, keys, zero)).map(toPx)

     def updateDim(d: Dimensions, values: List[Float]) = {
       val List(mt, mb, bt, bb, pt, pb) = values
       val pad = d.padding
       val mar = d.margin
       val bor = d.border
       val x_ = dim.x + mar.left + bor.left + pad.left
       val y_ = dim.y + dim.height + pt + bt + mt
       val newPadding = EdgeSize(pad.left, pad.right, pt, pb)
       val newBorder = EdgeSize(bor.left, bor.right, bt, bb)
       val newMargin = EdgeSize(mar.left, mar.right, mt, mb)
       Dimensions(x_, y_, d.width, d.height, newPadding, newBorder, newMargin)
     }

     Right(NTree((updateDim(boxDim, vals), boxType), children))
 }
}

def layoutChildren(box: LayoutBox) = {
 def accumulate(current: (Dimensions, List[LayoutBox]),
                childNodes: List[LayoutBox]): 
                  Either[String, (Dimensions, List[LayoutBox])] = {
   childNodes match {
     case Nil => Right(current)
     case head::rest =>
       val (d, acc) = current
       val NTree((cDim, _), _) = head
       layout(head, d) match { 
         case Left(s) => Left(s)
         case Right(c_) =>
           accumulate((Dimensions(d.x, d.y, d.width, d.height + marginBoxHeight(cDim),
                       d.padding, d.border, d.margin), acc ++ List(c_)), rest)
       }
   }
 }

 val NTree((dim, x), cs) = box
 accumulate((dim, List.empty[LayoutBox]), cs) match {
   case Left(s) => Left(s)
   case Right((dim_, cs_)) => Right(NTree((dim_, x), cs_))
 }
}

def calcHeight(box: LayoutBox) = {
 val NTree((boxDim, boxType), children) = box
 getStyledElem(box) match {
   case Left(s:String) => Left(s)
   case Right(style) => 
     val d = value(style, "height") match {
       case Some(Length(h, Px)) => 
         Dimensions(boxDim.x, boxDim.y, boxDim.width, h,
                    boxDim.padding, boxDim.border, boxDim.margin)
       case _ => boxDim // can be only None
     }
     Right(NTree((d, boxType), children))
 }
}
{% endhighlight %}

This concludes the sixth part of the sonny implementation.
Next task will be the painting of our boxes. 
As usual, you can see the full source 
[here](http://github.com/mdraeger/sonny) and the original posts by Leif are 
[here](https://hrothen.github.io).
