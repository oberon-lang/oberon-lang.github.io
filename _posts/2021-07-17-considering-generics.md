---
layout: post
title:  Considering Generics
author: Rochus Keller
---

Adding generics to Oberon was an interesting task whose solution required several attempts.

Usually, generic programming is realized by means of parametric polymorphism, where procedures and data structures are declared with variable types. In the nineties I had worked with Ada, which has generic units. Since modules are an explicit part of the language in Oberon, my first thought was to make the modules generic. 

The Ada approach has several advantages over e.g. C++; one of them is shared generics which avoids the possible "code bloat" of C++ templates; another is that generic modules can be validated by the compiler before they are instantiated. And instantiation of generic modules corresponds well with the separate compilation of modules. But the complexity of the Ada constraint syntax and the prospect that the instantiation of generic modules via import list (which seemed like the most natural and simple way to do it) would occur before the declaration of the actual type parameters looked discouraging.

I have therefore pursued generic types and extended the validator and code generator accordingly. There was already a proposal by Roe and Szyperski in the nineties [Ro97]. However, I considered it too restrictive. As in C++, any type should be able to be used as an actual type parameter, and - in contrast to e.g. Java - no boxing of base types should be required. Unfortunately, there was no satisfactory solution for how to incorporate procedures into the concept; every approach was kind of ugly. 

One of the problems was that the type params of type bound procedure had to somehow correspond with the type actuals of the receiver type. I finally decided for implicit type parameters, i.e. the receiver type is referenced without type actuals and there is no type param on the type bound procedure, but instead these are implicitly taken from the referenced receiver type declaration. Unfortunately this caused a bunch of rather complex rules which were in contradiction with the goal of simplicity (actually each approach caused a set of complex rules). And there was the issue with the instantiation of plain procedures which contradicted with the Oberon way that each object should have a unique name in the scope where it is defined. The code generator was no less complex, and unfortunately I encountered various "chicken or egg" problems to which I could not find a satisfactory solution. 

My conclusion was that parametric polymorphism just doesn't fit Oberon, and you experience the full force of complexity when you try to fight it. Another insight, not surprising in itself, was that you don't actually recognize the problems in this complex constellation until you are right in front of them. Thinking through and planning everything beforehand was not possible, at least not for me.

So a more suitable approach was needed. Since I wanted to make the declaration sequences flexible (in order and number) anyway, as in Component Pascal, I could benefit from this in several respects. The fact that a multi pass compiler is needed to implement flexible orders is from my point of view not worth thinking about in this day and age (the single pass requirement of the original Oberon had anyway consequences, which increased the complixity of the language as an undesired side effect). 

This brought me back to generic modules, where it was now no longer a problem, where in the module the actual type used for the module instantiation is declared. With that relaxation also all of the aforementioned problems vanished or solved themselves. Not surprisingly, the new validator and code generator was completed in a fraction of the time required by the previous approach. So generic modules are obviously the perfect match for Oberon.

It also came naturally that one can validate generic modules already, before they are instantiated (in contrast e.g. to C++ templates). An as a side effect this works with any actual types. The only thing the compiler needs to know about the type is that there is a default value. Everything else can be delegated e.g. to procedure types which are declared and used in the generic module, which again fits well with Oberon (e.g. procedure types for comparison operations or a hash function over the generic type, which are e.g. required by a dictionary, see [here for an example](https://github.com/rochus-keller/Oberon/blob/73a08f43a2f7f5a40c6b9ab38824ef9e2f58841b/testcases/Are-we-fast-yet/som/Dictionary.obx#L56)). This avoids the complex constraint language of Ada generics. The specification of the generic modules in the language report required less than one page.

#### References
[Ro97] Roe, P.; Szyperski, C. (1997). Lightweight parametric polymorphism for Oberon. In: Mössenböck H. (eds). Modular Programming Languages. JMLC 1997. Lecture Notes in Computer Science, vol 1204. Springer, Berlin, Heidelberg. 
