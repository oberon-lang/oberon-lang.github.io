---
layout: post
title:  Towards Oberon+ Exception Handling
author: Rochus Keller
---

#### State of error handling in Oberon and Modula-2
Original Oberon has little provisions for handling error conditions. The Oberon-07 specification [Wi16] doesn't mention errors at all; neither do the Oberon-2 [Mo91] nor the original Oberon specification [Wi88b], but both define the `HALT()` predefined procedure to terminate the program, and both specify conditions under which the program execution is automatically aborted (e.g. if the dynamic type doesn't meet a type guard). 

As we know, Oberon evolved from Modula-2 by very few additions and many more subtractions [Wi88a]. So it makes sense to check how error handling was considered in Modula-2. As with Oberon neither the standard textbook nor the language report [Wi88c] specifically address error condition handling; but the examples suggest that calling HALT is an appropriate measure to do so. However, there are other publications that specifically address this issue; e.g. [Re87] recommends user supplied error handling routines which are passed to an instance of an abstract data type via a SetErrorHandler procedure, but also states that this solution was not as elegant as the exception handling provided in Ada, especially with reusable modules. The recommended method also works with Oberon, but suffers from the drawbacks described in [Re87] as well. 

#### Discussion of existing solutions
In Ada [Ad86], which appeared at about the same time as Modula-2, for each block of statements an exception handler can be optionally declared. An exception is just a symbol of type `exception`. There are predefined exceptions (like `constraint_error` if e.g. a negative integer is assigned to a natural) and custom exceptions (e.g. `My_Exception : exception;`). An exception can be raised anywhere in the code with a statement like e.g `raise My_Exception;`. An exception handler lists the exceptions to be handled and associates each with the statements to be executed in case of the exception. If an exception is not handled by a handler it is handed up the call chain to the next handler and so on. Here is an example:

```
with ada.text_io; use ada.text_io; 
with ada.integer_text_io; use ada.integer_text_io; 

procedure read_number  is 
    n: Natural;
begin
    get(n);
    put_line(n'img);
exception
    when constraint_error => 
        put_line("You must enter an non-negative value!");
    when end_error => 
        put_line("Premature end of file!");
end read_number;
```

C++ exception handling goes beyond the capabilities of Ada 83 in that an exception is a value of any type, and the value raised as an exception is transmitted to the handler in a type-safe way [St94]. Especially interesting is the possibility to use instances of classes as exceptions, which makes it possible to handle groups of exceptions. Also in Ada 83 groups of exceptions can be handled together; however, the groups are merely (explicit) enumerations of symbols. In C++, on the other hand, a handler can be responsible for a class, which also includes all subclasses, without having to list them explicitly. According to [St94], the idea for this came from the programming language ML [Mi97]. The capability of using other types than classes as exceptions can be regarded as syntactic sugar since a value of any type can be a member of a class; not surprisingly, languages like Java or C# only support exceptions based on classes. 

For the ISO standard, some additions were made to Modula-2 compared to [Wi88c], including exception handling [Sc96]. It follows an interesting approach in that there is an `EXCEPTION` keyword, but raising and identifying a raised exception is done with procedures defined in the standard library. Here is an example with language exceptions:

```
IMPORT M2EXCEPTION;
PROCEDURE DivideByZero;
VAR x,y,z: CARDINAL;
BEGIN
  x := 10; y := 0;
  z := x DIV y;
EXCEPT
  IF M2EXCEPTION.IsM2Exception() THEN
    CASE M2EXCEPTION.M2Exception() OF
      M2EXCEPTION.wholeDivException: (* react to language exception *)
      | M2EXCEPTION.rangeException: (* react to language exception *)
    END;
  ELSE
    (* react to user-defined exceptions *)
  END;
END DivideByZero;
```

User-defined exceptions are a bit more involved. The concept assumes that each module implements the required exceptions and makes the procedures for raising them and checking their identity part of the public interface of the module (as demonstrated with the M2EXCEPTION module). Just declaring and handling exceptions on the fly as in Ada is not supported in ISO Modula-2. The exception itself is just a number, usually defined as an enumeration in the public interface. But there is also a special `EXCEPTIONS.ExceptionSource` type defined in the standard library to identify the module at runtime. It should also be noted that ISO Modula-2 is one of the few languages which support resumption semantics of exceptions (using the `RETRY` keyword). In contrast to Ada or C++, which support block statements, exception handlers in ISO Modula-2 are restricted to module or procedure bodies as a whole. 

How exception handling can be completely done without any additional syntax is nicely demonstrated in Lua [Ie06]. In Lua there is a predeclared function called `pcall()` which takes a function F as the first argument and then the arguments required to call F, one after the other. So if F is called by `pcall()` and not directly, any error occuring in F or another function called by F, does not terminate the program, but just makes `pcall()` return instead with a result representing the error which has occured. F is said to be called in "protected mode", thus the name. Lua has yet another predeclared function called `error()` which takes any value as argument (usually a string describing the error). If `error()` is called, it never returns, but instead terminates the program, or ends the closest `pcall()` call on the call stack; in the latter case the result of `pcall()` includes the argument passed to `error()`. So `pcall()` and `error()` represent a formidable means for dynamic non-local exits, which is an official part of the language without making the syntax more complex.

But Lua was not the first language to solve the problem of exception handling in this way. Already Lisp 1.5 in 1961 had a function called `ERRORSET` which did pretty much the same thing as `pcall()`, see [Mc61]. MacLisp evolved from Lisp 1.5 around 1966 and included the ERRORSET function, renamed to `ERRSET`; in addition it had a function called `ERR` which did essentially the same as the Lua `error()` function; by 1972 the functions `CATCH` and `THROW` were added to MacLisp, which worked similarly to ERRSET/ERR, but were especially conceived for dynamic non-local exits without interfering with error handling [St96]; not hard to guess where the keywords `catch` and `throw` come from, which can be found in many current languages.

#### Derivation of an optimal solution for Oberon+ and discussion
Wirth formidably demonstrated how to only integrate one essential feature - namely type extension - into the language to enable the object-oriented programming style, and at the same time to keep the language as simple as possible [Wi88b]. So the question here is: which is the one essential feature to enable exception handling. 

The above considerations suggest that this feature must be the dynamic non-local exit with a record representing the exception type. Oberon already has type extension, so handling groups of exceptions can be supported like in C++ or Java. Lua and MacLisp demonstrated, that actually only two functions are required to realize dynamic non-local exit with value transmission: a throw and a catch function. 

In the spirit of Oberon, to make the language as simple as possible, an approach based on two predeclared procedures, without changing the syntax, seems very attractive. Let's call these procedures `PCALL()` (as in Lua because of the similarity) and `RAISE()` (as in Ada, ISO Modula-2 and most other Pascal descendants). ISO Modula-2 - in contrast to C++, Ada, C# or Java - supports resumption semantics, which is likely the reason why a special syntax for handlers was necessary; I concur with [St94] that termination semantics is simpler, cleaner and powerful enough. 

PCALL() expects a procedure-typed argument, for the procedure to be called, and each argument or this procedure (if any). We already have a predeclared procedure with a parameter of varying type and a variable number of additional parameters, namely `NEW()`; when used to instantiate records, NEW() receives a pointer to the corresponding record type; when used to instantiate arrays, NEW() receives a pointer to the array type and a length for each (open) dimension. So this concept is not new to Oberon. The compiler is able to check the number and type compatibility of the arguments in a similar way it is done for NEW(). 

RAISE() expects a POINTER TO ANYREC which shall not be NIL (otherwise the program execution is aborted). ANYREC is already defined in Oberon+ as predeclared record type of which each RECORD is an implicit extension. If RAISE() is called without a preceding call of PCALL() on the call chain the program execution is aborted; a compiler is able to recognize and report such an error. Also a variant of RAISE() without parameter would make sense, if just a non-local exit was intended, not throwing an exception; in that case RAISE() could use an internal instance of ANYREC to avoid NIL.

PCALL() returns the POINTER TO ANYREC, which was passed to RAISE(), or NIL in case RAISE() was not called in the course of PCALL. There are two points to consider here: should PCALL() be a function or proper procedure, and should the procedure type passed to PCALL() allow for return types. If PCALL() was a function it could be called from any expression, which doesn't seem to be desirable for different reasons; one reason is that a runtime system as e.g. specified in ECMA-335 might require that the evaluation stack is empty when entering a protected block or handler; thus when calling PCALL() in an assigment or for an actual argument special management of temporary values would be required (since the evaluation stack is not available); another reason might be, that in all discussed languages so far protected blocks are associated with module or procedure bodies, or statements, but never with expressions. I decide for declaring PCALL() as a proper procedure, because it is cleaner and simpler to implement. As a consequence the result has to be handed over to the caller by VAR parameter, as it is done e.g. in NEW(). I also decided that only proper procedure types can be passed to PCALL(), because the call of PCALL() is effectively a substitute to the call of the procedure type, and as such has rather a statement than an expression flair, and because it's always possible to pass back results via VAR parameters. 

The approach with PCALL/RAISE is more flexible than the ISO Modula-2 approach, because exceptions can be handled in context, and not only at the end of a module or procedure body. It is also more flexible than Ada 83 and ISO Modula-2, in that any record with any information can be used as an exception (not only symbols or numbers with an optional string attached) and grouping of exceptions is possible without explicitly listing all possible cases. The approach is as flexible as C# and Java, in that protected calls can be nested and there can be more than one "protected block" in a procedure body. Only the C++ approach is more powerful because objects can also be passed by value, not only as pointers to heap allocated objects; but as demonstrated in C# and Java this does not seem to be a big disadvantage. Also the restriction to proper procedures as arguments to PCALL() is no big disadvantage, since a function procedure can be encapsulated in a local proper procedure with an additional VAR parameter representing the return type.

Here is an example (as implemented in the Oberon+ IDE 0.9.71):
```
module Exception
  type Exception = record  end
  proc Print(in str: array of char)
  var e: pointer to Exception 
  begin
    println(str)
    new(e)
    raise(e)
    println("this is not printed")
  end Print
  var res: pointer to anyrec
begin
  pcall(res, Print, "Hello World")
  case res of
  | Exception: println("got Exception")
  | anyrec: println("got anyrec")
  | nil: println("no exception")
  else
    println("unknown exception")
  end
end Exception
```

#### References

- [Ad86] United States Government (1986). Rationale for the Design of the Ada Programming Language (Ada'83 Rationale). http://archive.adaic.com/standards/83rat/html/ratl-00.html (accessed 2022-05-14).
- [Ie06] Ierusalimschy, R., et al. (2006). Lua 5.1 reference manual. https://www.lua.org/manual/5.1/ (accessed 2022-05-14).
- [Mc61] McCarthy, J., et al. (1961). LISP 1.5 programmer's manual. 1nd Edition, Jul 14 1961. Computation Center & Research Lab of Electronics at MIT.
- [Mi97] Milner, R., et al. (1997). The definition of standard ML: revised. MIT press.
- [Mo91] Mössenböck, H., & Wirth, N. (1991). The programming language Oberon-2. Structured Programming, 12(4), 179-196.
- [Re87] Rehmer, K. (1987). Error handling using Modula-2. ACM SIGPLAN Notices, 22(3), 40-48.
- [Sc96] Schoenhacker, M., & Pronk, C. (1996). ISO/IEC 10514–1, the standard for Modula-2: changes, clarifications and additions. ACM SIGPLAN Notices, 31(8), 84-95.
- [St94] Stroustrup, B. (1994). The design and evolution of C++. Addison-Wesley.
- [St96] Steele, G. L., & Gabriel, R. P. (1996). The evolution of Lisp. In History of programming languages---II (pp. 233-330).
- [Wi16] Wirth, N. (2016). The Programming Language Oberon. https://people.inf.ethz.ch/wirth/Oberon/Oberon07.Report.pdf (accessed 2020-11-16).
- [Wi88a] Wirth, N. (1988). From modula to oberon. Softw., Pract. Exper., 18: 661-670. 
- [Wi88b] Wirth, N. (1988). The programming language Oberon. Softw., Pract. Exper., 18(7), 671-690.
- [Wi88c] Wirth, N. (1988). Programming in Modula-2. 4th Edition. Springer-Verlag.


Pending:
- If we had record literals, we could pass a record by value to RAISE(), without declaring a pointer to record and calling NEW().
- What about non-local access and PCALL? Should it be supported?
- If a proc called via PCALL which has VAR params throws an exception or calls a proc which raises an exception the state of the object handed in by VAR is random (CLR keeps all changes up to the RAISE, C goes back before the PCALL).

