---
layout: post
title:  Comparing Oberon+ with Oberon-2 and Oberon-07
author: Rochus Keller 
---
(Updated 2023-12-27)

The Oberon+ language definition started as a union of the Oberon-07 and Oberon-2 language definitions. Since both Oberon-2 and Oberon-07 are an extension of the first Oberon version (1987 and 1990), the Oberon+ language definition therefore also includes the original Oberon language. This means that each program written in Oberon up to Oberon-2 and including Oberon-07 is compatible with the Oberon+ compiler. 

[Oberon-2](http://www.ssw.uni-linz.ac.at/Research/Papers/Oberon2.pdf) added the following elements to the original Oberon:

- Type–bound procedures
- Read–only export
- Open arrays
- Extended WITH statements (with variants)
- FOR statements

[Oberon-07](http://people.inf.ethz.ch/wirth/Oberon/Oberon07.Report.pdf) (revision 3.5.2016) also describes itself as an extension of the original Oberon (thus ignoring most of the Oberon-2 extensions); here is a list of the major changes in Oberon-07:

- Removed the LOOP and EXIT statements
- Added the FOR statement
- Added the optional ELSIF part to the WHILE statement
- Removed the WITH statement
- Extended the CASE statement so that the case variable may also be a record or pointer to record type ("type CASE", as a replacement of WITH)
- RETURN is no longer a statement but part of the procedure body (which supports RETURN without BEGIN)
- Support direct array and record assignments, making the COPY procedure obsolete 
- Variables imported from other modules are now read-only
- Removed the concept of type inclusion, as well as the SHORTINT and LONGINT types
- TRUE and FALSE are keywords, no longer global constants
- Removed access to local variables and parameters of outer procedures from nested procedures ("non-local access")

Since Oberon+ includes a union of Oberon-2 and Oberon-07, all removals done by Oberon-07 are ignored. The resulting redundancy with WITH and type CASE is tolerated in favor of compatibility. To accomodate the effect of the Oberon-07 RETURN, procedure bodies can either consist of BEGIN plus statement sequence, or a RETURN statement. Oberon-07 programs with read-only access to imported module variables still run on Oberon+ without modification (they simply do without write access); if the author of an Oberon-07 module wants to enforce read-only exports of the module variables, visibility mark must be changed to "-". 

In addition to the above, Oberon+ adds the following:

- Lowercase keywords and base type identifiers
- Simplifications and shortened keywords (^ instead of POINTER TO, [] instead of ARRAY OF, etc.)
- Underscores in identifiers
- All semicolons optional, most commas optional
- UTF-8 source files with Unicode string literals and comments
- WCHAR type
- String concatenation using the + operator
- Line comments
- Import paths, see [this post]({% post_url 2021-07-18-considering-imports %})
- Flexible declaration sequences, more than one CONST, TYPE, VAR or IMPORT section in arbitrary order
- DEFINITION modules
- Enumerations
- Procedure local records with type-bound procedures
- Delegate procedure types (procedure types for bound procedures)
- Constant VAR parameters using the IN keyword
- ANYREC, implicit base type of each record
- Generic modules, see [this post]({% post_url 2021-07-17-considering-generics %}), with support for literals and procedure references as generic arguments
- FFI language for cross-platform C shared library integration, see [this post]({% post_url 2021-08-03-towards-a-foreign-function-interface %})
- Variable length arrays (VLA)
- Conditional compilation (as recommended by the Oakwood guidelines)
- Exception handling, see [this post]({% post_url 2022-05-15-towards-exception-handling %})


Features in evaluation:

- Array and record literals
- COMPLEX type, as defined in the Oakwood guidelines, including standard libraries
- Covariance of overloaded method return types and fields (like Component Pascal)
- Concurrency, see [this post]({% post_url 2023-12-25-towards-concurrency %})



