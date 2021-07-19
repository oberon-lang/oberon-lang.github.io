---
layout: post
title:  Comparing Oberon+ with Oberon-2 and Oberon-07
author: Rochus Keller
---

The Oberon+ language definition started as a union of the Oberon-07 and Oberon-2 language definitions. Since both Oberon-2 and Oberon-07 are an extension of the first Oberon version (1987 and 1990), the Oberon+ language definition therefore also includes the original Oberon language. This means, that in principle each program written in Oberon up to Oberon-2 and including Oberon-07 is compatible with the Oberon+ compiler. There is one exception at which we will look below.

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

Since Oberon+ includes a union of Oberon-2 and Oberon-07, all removals done by Oberon-07 besides the last one ("non-local access") are ignored. The resulting redundancy with WITH and type CASE is tolerated in favor of compatibility. To accomodate the effect of the Oberon-07 RETURN, procedure bodies can either consist of BEGIN plus declaration sequence, or a RETURN statement. The read-only access to imported module variables is no issue, since programs meeting this constraint are still compatible with original Oberon (they simply do without write access). 

The missing non-local access of Oberon+ can cause incompatibilities with programs written in original Oberon or Oberon-2, although only a fraction of the existing code seems to use this feature. Whether it is feasible and worthwhile to add it to Oberon+ is subject to further investigations. In any case a simple work-around is to pass the required objects as VAR or IN parameters to the nested procedures.

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
- Constant VAR parameters using the IN keyword
- Generic modules, see [this post]({% post_url 2021-07-17-considering-generics %})

Pending features of Oberon+:

- Delegate procedure types (procedure types for bound procedures)
- C FFI (based on DEFINITION modules with special C compatible types)
- Common base type of all records, ANYREC (like Component Pascal)

Features in evaluation:

- Procedure body interleaved with declaration sequences
- Non-local variable and parameter access
- Covariance of overloaded method return types and fields (like Component Pascal)




