---
layout: post
title:  Towards an Oberon+ Foreign Function Interface (FFI)
author: Rochus Keller
---

C is ubiquitous, and there are thousands of battle-tested C libraries worth reusing. Unfortunately, however, the C programming language has its pitfalls, so it makes perfect sense not to implement the actual application in C directly.

These are all good reasons to include interfacing to C libraries as an official feature in Oberon+.

The original Oberon was a world of its own, even its own operating system running on dedicated hardware. Integration with components from other programming languages was not foreseen. 

However, over time compiler and language variants appeared where external library integration was supported. The approaches ranged from implicit integration via modules, which offer a platform-independent API and have been implemented specifically for each platform (e.g. with inline assembler), to explicit association of modules with dynamic link libraries (DLL) or even the possibility to use Component Object Model (COM) interfaces as record base types. The language was only minimally extended by attributes which could be attached to modules, records, pointers and procedures. 

The original approach in Oberon is to have a safe, platform-independent language, and to handle the unsafe, platform-dependent parts using features that reside in the SYSTEM module. The language report recommends to restrict the use of the SYSTEM module to specific low-level modules, which are inherently non-portable, but easily recognized due to the identifier SYSTEM appearing in their import lists.

Having studied various Oberon variants in depth, I consider the following requirements to be important.

### Integration with C libraries must be easy, but not seamless

One should not blur the difference between data structures in Oberon and C. The conceptual model of Oberon differs considerably from that of C; e.g. unlike C, Oberon does not support pointer arithmetic; it is not even (regularly) possible to obtain a pointer from a record or an array. Pointers to structured types are explicitly declarable and initialized with nil by the compiler; they point to objects only if one is either explicitly associated with NEW() for the pointer, or if the pointer is assigned the value of another compatible pointer. There is no way to delete an object pointed to by a pointer (if no pointer points to the object anymore, it is removed by the garbage collector); thus, a pointer always points to a valid object or has the value nil.

In contrast, a pointer in C can point to an arbitrary memory location even if there is no valid object at it; one can obtain the address from arbitrary objects (even temporary ones on the stack) and change the pointer's value arbitrarily; the compiler's type checking can be easily bypassed. For low-level applications and in the hands of experienced, disciplined developers, C is undoubtedly a very powerful tool. 

For all other applications and developers, C is a risk. Thus, seamlessly masking C data types as Oberon data types obscures the risk that the Oberon developer is exposing himself to. Although the Oberon types representing C types are marked with attributes in the aforementioned language variants, they are syntactically and semantically still the same Oberon types as described in the Language Report.

From my point of view, C types must be represented in Oberon+ as independent types, clearly different from Oberon types with explicitly different rules for declaration, assignment, etc.; anyway, C has types that cannot be represented in Oberon at all (e.g. unions), which is again an argument in favor of an explicit type system. So let's postulate CSTRUCT, CUNION, CARRAY and CPOINTER. A CPOINTER is not the same as a POINTER, and you cannot assign them to each other or mix them up otherwise; the same is generally true for the other types as well; specific differences will be discussed later.

And of course using a C library should be easy; from my point of view easy means just adding a DEFINITION module representing the C library to the project and the compiler/linker takes care of everything else. The DEFINITION module corresponds to the C header files and it should be possible to automatically generate it from the original headers with no or minimal modifications requried (keeping in mind that e.g. a pointer type in C can also represent an array or an out parameter which is undecidable for the generator - or even for the user when there is no documentation otherwise - in general).


### The language must support C library integration as a regular feature

If this is not specified as a regular feature in the language definition, there are too many exceptions and special cases that are not documented or only documented in some implementation notes. The number of special cases, and thus the risk of unspecified properties, is significantly higher if the regular Oberon types are also (mis)used to represent C types than if there are two clearly separated type systems, each with specific properties and clear interfaces.

What should be considered? As already mentioned there should be a corresponding Oberon+ type for each structured and pointer C type. Basic types instead can be re-used for both Oberon+ and the integrated C data objects, provided that a solution can be found for the C types with unspecified byte size. 

Structured types should not be re-used due to the different memory management strategies (automatic in Oberon+, manual in C); it should be possible partially re-use arrays though to avoid unneccessary copies; since in Oberon+ as well as C array elements are stored at consecutive memory addresses, this seems feasible; but a C function neither knows nor is forced to respect the length of the ARRAY which is a risk.

POINTER should not be re-used for C since they should be specific on what they can point to; I even think that pointers to basic types should not be supported (even if C does so), but instead replaced by pointers to one-element arrays (which is possible because a pointer in C can point to a single or an array of data objects). Taking the address (pointer) of a data object is an operation not present in (regular) Oberon+ yet; C uses the '&' operator; Oberon has the ADR() predefined procedure in the SYSTEM module, which could be re-defined and re-used.

Last but not least both languages have procedures, but C neither has type-bound procedures nor VAR parameters; C uses pointers also for the purpose of VAR parameters (among a couple of other purposes); and there is the issue of calling conventions, at the latest when an Oberon+ procedure type should be used as a callback in a C library.


### Unsafe operations must be explicit and easy to find

What is an unsafe operation? There are a few clear indications, but much is also a matter of judgment. 

Many people, including myself, would consider Oberon a safe language because there is automatic memory management, the addresses of data elements cannot be obtained (except via SYSTEM module, which is not present in Oberon+), no pointer arithmetic is possible, and pointers always point to a valid object or have the value nil. 

It would therefore be unsafe to manually deallocate memory (or access data objects whose memory has been freed elsewhere), to obtain the memory addresses of data objects (which possibly becomes invalid, e.g. when leafing the scope), to manipulate pointers, or to call a function in an external library which does any of these (this list is not exhaustive).

If a C library is exclusively represented by a module, then one can assume that the call of each procedure in that module is potentially unsafe; this is easy to identify; it would not be that easy (i.e. each procedure would have to be considered individually) if a module could contain both, Oberon and C procedure declarations. 

Using a predefined procedure like ADR() to obtain an address fits well with Oberon+; if the IDE offers a cross-reference list which shows the use of each symbol, the use of ADR() is automatically included; if ADR() is restricted to the aforementioned C types, the import of the module where C types are declared is yet another indication. 

Special consideration requires the question whether it should be possible to use the C types for declarations in regular modules; in this case it is essential whether it is possible to also dynamically allocate (and free) objects of these types; if this is not possible, only taking the address of such an object is a (potentially) unsafe operation (when the module is not calling C functions otherwise). But it's easy for the parser to find out where ADR() or C types are used and e.g. to mark those modules as unsafe.

Component Pascal supports an implicit address-of operation when assigning a structured value type to a pointer variable or parameter; this reduces the amount of typing, but hides the risk; so one has to find a suitable balance here; requiring ADR() when assigning or calling an ordinary Oberon+ procedure (but not when calling a C function) looks like a good compromise (the latter can be detected by the import list).


### C data object must be directly accessible, without marshalling

If I have to build an adapter to access C data objects, it gives more work on the one hand, and on the other hand the additional indirection reduces performance. Therefore I would like to be able to diretly access C structured data objects from whithin Oberon+ code. If this data is allocated or modified by a C function this bears the risk that Oberon+ is accessing invalid objects.

When the data objects used and provided by C functions have their own types, the memory management of the Oberon and C types can be kept apart. It should not be possible for a CPOINTER to reference a RECORD (in contrast to a CSTRUCT or CUNION); with ARRAYS we have likely to make compromises to avoid copying (e.g. by restricting ADR() to ARRAYs of basic types). 

The question is whether we can afford to not to allow RECORDs and ARRAYs to have C types as field or element types, or whether this would reduce the usefulness of the FFI too much; I currently tend to only allow RECORDs and ARRAYs to have Oberon or CPOINTER field and element types to avoid that memory referenced by a CPOINTER is part of a structured Oberon data object (therefore the result of ADR() cannot point e.g. to a RECORD field or ARRAY elemement); or the compiler would silently dynamically allocate the structured C types and just store their address in the RECORD field or ARRAY element (for the programmer, the illusion of the structured C value type would be maintained). 

It would still be possible to have local or module variables of a structured C type; passing the address of these to a C function could potentially damage the stack; to avoid this the compiler could again silently allocate those dynamically (e.g. in a dedicated memory area) and just store their address on the stack (this is likely disproportionate though, as long as the C function runs on the same stack as the Oberon+ procedures).


### The role of DEFINITION modules

Oberon+ uses DEFINITION modules as first-class citicens, not just to summarize the public module elements using a pseudo language. In general a DEFINITION is a module of which the implementation is only available in a binary (non human readable) format; the same applies to external libraries. If an Oberon+ module is compiled and exported for re-use, the result is a binary object with a DEFINITION module.

It should be possible to automatically generate DEFINITION modules from the header files of the required C library (see above). A DEFINITION representing a C library (instead of a precompiled Oberon+ module) should be easily recognized, e.g. by the presence of module attributes like [ lib "SDL2", prefix "SDL_" ]; no such attributes are required for a precompiled Oberon+ module. Another idea would be to explicitly add the UNSAFE keyword to DEFINITIONs representing C libraries, or even to all MODULES where C types are used.

The parser takes care that a DEFINITION representing a C library only uses the Oberon+ dedicated C types in declarations (and not regular Oberon+ structured types). If at all, such modules should only import other DEFINITIONs representing C libraries or header files. 

### Update 2021-11-07

Over the last months I implemented different versions of the FFI. 

The first one was implemented in the LuaJIT version of Oberon+. It mostly corresponded to what I've written above. But apparently I reached the limits of LuaJIT. The OberonSystem version based on SDL randomly crashed and I wasn't able to find the reason nor a work-around; a reason could be that SDL internally uses additional threads which is not compatible with LuaJIT, but that's just an assumption. 

I therefore evaluated the Common Language Infrastructure (CLI, ECMA-335) and specifically different Mono versions which turned out to be a suitable alternative technology to LuaJIT. Implementing a code generator required some time and a lot of research and trial and error. It became apparent that a few changes would be necessary for the FFI due to the strict separation of managed and unmanaged memory in CLI. 

The most important change is giving up the ADR() function. ADR() didn't fit because it could be used independently of calls to external procedures. This is not a problem for the C structured types (CSTRUCT etc.) which don't require special memory management consideration. But it is an issue in the case an ARRAY should be directly passed to an external procedure (to avoid copies). CIL supports taking the address of blitable structured types as long as the memory is "pinned" (i.e. excluded from relocation by the garbage collector); this feature allows us to pass the address to an external procedure without the GC turning the address invalid during the call; but we also have to "unpin" the memory as soon as possible. And also if we make a copy of the array we have to release the copy after use and possibly write back the data modified by the external procedure call to the original array. With ADR() we know when to "pin" or copy the array, but it is very difficult to find out when it is safe to unpin or when we have to write back the data (remember the address could be passed out of scope or even stored in a module variable). A possible solution could be a RELEASE() function by which we tell the runtime that an address received by ADR() is no longer used; but this is not only additional effort but also a perfect source of programming errors.

I therefore decided to follow the (apparently undocumented) Component Pascal approach, where procedure calls can include an implicit address-of operation (the formal parameter is a pointer to a base type compatible with the provided actual parameter). But this is only supported for C structured types; this should somewhat outweigh the disadvantage of being an implicit operation. One could also have made this operation explicit e.g. by making the ADR() call part of the function call, or by a new keyword like "ref" in C#, but I considered this to be less elegant and unnecessarily complicated given the fact that it is only supported for unsafe types anyway. 

Component Pascal also supports implicit address-of operation when assigning a structured type to a pointer of this structured type. To avoid the aforementioned unpin/write-back issue Oberon+ only supports this for C structured types. Since this statement can occur anywhere, this would be a sensible application of ADR(); but I weighted the extra function just for this purpose heavier than the implicit operation, which may be a mistake - we'll see. Anyway the parser can easily identify all implicit address-of operations even without explicit ADR() and the IDE could list or mark these if need be.

Here is [the specification](https://github.com/oberon-lang/specification/blob/master/The_Programming_Language_Oberon%2B.adoc#foreign-function-interface).

Here is [an FFI example](https://github.com/rochus-keller/OberonSystem/blob/FFI/ObSdl.obx).



