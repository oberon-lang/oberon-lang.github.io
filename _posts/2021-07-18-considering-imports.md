---
layout: post
title:  Considering Imports
author: Rochus Keller
---

Oberon had a module and separate compilation concept from the beginning. Modules are an explicit part of the syntax. A module can import other modules by adding the name of the modules to be imported to the import list of the importing module. 

But where are the modules actually located in the file system, and what happens, if more than one module has the same name? The Oberon language reports don't say anything about the relation of modules and files (just that a module is typically a compilation unit). The Oberon system offered a rather makeshift solution. There was actually no file system, but only one big directory. To be able to organize the files to some extent, a prefix approach was used. However, it was still possible for two modules to have the same name after the prefixes were resolved. As far as I know, the compiler then simply used the newer file silently. 

Even in the heyday of the Oberon system, it quickly became clear that this approach would reach its limits as soon as several people contributed modules. A linear namespace, which is shared by all existing modules, is no longer appropriate today. Interestingly, I don't know of any Oberon variant (before Oberon+) that would offer a better solution.

How is it done in the other languages? C++ didn't have a module concept at all for the last fourty years; at least there are namespaces since twenty years, and include directives also support file system paths, not only file names. Java supports implicit modules in that each class is associated with a file, and import directives support paths, which are mapped to file system paths relative to a global class path list known to the compiler; each class has to explicitly declare to which package it belongs. Both approaches have proven to be suitable over the years. Though personally I find it quite tedious that I'm forced to hide class files in complicated directory structures, and in C++ the path syntax is even operating system dependent. 

For Oberon+, I wanted a hierarchical namespace without forcing the developer to use a particular directory structure, and without being dependent on the operating system. It should be possible for a developer to develop several related modules without having to determine a priori where the importing application will place these modules in the hierarchical namespace.

After considering various options, a fairly simple solution manifested itself. The module names in the import list can be prefixed with an import path (a sequence of idents with separating dots, e.g. "som.Dictionary"). There is no predetermined correlation between this import path and the file system, and the imported module doesn't have to know anything about the import path used in an importing module. In the importing module, the declarations of the imported module can still be accessed by prefixing the selector with the module name, even in the presence of an import path. In case there is more than one module with the same name, each one has to be assigned to a different alias name (using the ":=" syntax); the imported modules can nevertheless be distinguished as long as their import paths differ. There is yet another rule, that imports without import path are first looked up in the import path of the importing module; this makes it possible that a bunch of related modules can be developed without caring for their import path, assuming that these modules together are always imported under the same import path. 

See the [som package for an example](https://github.com/rochus-keller/Oberon/tree/master/testcases/Are-we-fast-yet/som). E.g. the Dictionary [imports Vector without an import path](https://github.com/rochus-keller/Oberon/blob/73a08f43a2f7f5a40c6b9ab38824ef9e2f58841b/testcases/Are-we-fast-yet/som/Dictionary.obx#L26). But then e.g. DeltaBlue imports the dictionary (the IdentityDictionary in this case) under the [som import path](https://github.com/rochus-keller/Oberon/blob/73a08f43a2f7f5a40c6b9ab38824ef9e2f58841b/testcases/Are-we-fast-yet/DeltaBlue.obx#L78). 

Here is [a larger example](https://github.com/rochus-keller/BlackboxFramework/tree/master/Minimal). In this case, the directory structure of the modules corresponds to the import paths. Even if there are several modules with the same name, there is no issue because they are accessed under different import paths. 

It is up to the compiler how the mapping from files to (virtual) import paths is implemented. 
