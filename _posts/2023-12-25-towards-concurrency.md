---
layout: post
title:  Towards Oberon+ Concurrency
author: Rochus Keller
---

(Updated 2024-01-07)

(subject to public review until further notice)

#### What is concurrency

According to [1] 3.755 concurrency is a "property of a system in which events can occur independently of each other, and hence are not ordered". This very general definition is based on independent, unordered events and actions ([1] 3.1484). How actions (i.e. activities) can be made concurrent, we learn from [1] 3.756: "either by interleaving the activities or by simultaneous execution". So we can achieve concurrency by either interleaving activities on a single processor, or by simultaneously executing activities on multiple processors.

Both methods of concurrency have been successfully implemented over the years in various programming languages. 

Today, where virtually every system includes more than one processor, support for parallel execution has become essential. A programming language can support concurrency either through libraries and external function calls, or directly through built-in language constructs. Both approaches are ubiquitous.

This paper examines how state-of-the-art parallel execution should be supported in Oberon+, while keeping the language as simple as possible.

#### Concurrency in Modula

Wirth's first programming language supporting concurrency was MODULA [3]. In [4] he writes: "I had designed and implemented the predecessor language Modula [..] as a small, custom-tailored language for experimenting with concurrent processes and primitives for their synchronization. Its features were essentially confined to this topic, such as process, signal, and monitor [5]." Modula was largely based on Pascal, but in addition to conventional block structure it introduced a modules as a set of procedures, data types, and variables, and also processes and signals as general multiprocessing facilities.

A monitor in MODULA is represented by an "interface module". Calls to procedures declared in interface modules are sequenced, i.e. if a process has called any such procedure, another process calling the same or another one of these procedures is delayed, until the first process has completed its procedure or starts waiting for a siqnal. Signals are condition variables. If a process calls wait() on the signal, it is suspended until another process calls send() on the same signal. Processes look like procedures, besides the different keyword; processes are also called like procedures, but in contrast to a procedure a call to a process initiates a new concurrent execution.

Here is a part of the example in [3] page 27 demonstrating some of the multiprocessing facilities:
```
interface module resourcereservation;
    define semaphore,P,V,init;
    type semaphore = record taken: Boolean; 
         free: siqnal end;
    
    procedure P(var s: semaphore);
    begin if s.taken then wait(s.free) end;
        s.taken := true
    end P;

    procedure V(var s: semaphore);
    begin s.taken := false; send(s.free)
    end V;

    procedure init(var s: semaphore)
    begin s.taken := false
    end init;
end resourcereservation
```
The multiprocessing facilities of MODULA were no longer present in Modula-2. Wirth writes in [4]: "There was no clear favorite way to express
and control concurrency, and hence no set of language constructs that clearly offered themselves for inclusion. One basic concept was seen in concurrent processes synchronized by signals (or conditions), and involving critical regions of mutual exclusion in the form of monitors [5]. Yet, it was decided that only the very basic notion of coroutines would be included in Modula-2, and that higher abstractions should be programmed as modules based on co-routines."

In [6], Wirth states: "Modula-2 is designed primarily for implementation on a conventional single-processor computer. For multiprogramming it offers only some basic facilities which allow the specification of quasi-concurrent processes [..]. The word process is here used with the meaning of coroutine."
In contrast to MODULA, there are no dedicated language constructs for multiprocessing in Modula-2; instead there is a standard module called "Processes" with which a new concurrent process can be started based on a procedure passed as an argument to StartProcess. [6] section 30 leaves it to the implementation whether the process is executed in genuine or quasi-concurrency, whereas the language report in the same book only foresees quasi-concurrent processes. The Processes module also includes a SIGNAL type and a SEND and WAIT procedure, with the same semantics as in MODULA. 

Here is the example in [6] page 124 demonstrating Modula-2 multiprocessing:
```
MODULE Buffer [1];
    EXPORT deposit, fetch;
    IMPORT SIGNAL, SEND, WAIT, Init, ElementType;
    CONST N = 128; (*buffer size*)
    VAR n: [0 .. N]; (*no. of deposited elements*)
        nonfull: SIGNAL; (* n < N *)
        nonempty: SIGNAL; (* n > 0 *)
        in, out: [0 .. N-1]; (*indices*)
        buf: ARRAY [0 .. N-1] OF ElementType;
        
    PROCEDURE deposit(x: ElementType);
    BEGIN
        IF n = N THEN WAIT(nonfull) END;
        (* n < N *) n := n+1; (* 0 < n <= N *)
        buf[in] := x; in := (in+1) MOD N;
        SEND(nonempty)
    END deposit;
    
    PROCEDURE fetch (VAR x: ElementType);
    BEGIN
        IF n = 0 THEN WAIT(nonempty) END;
        (* n > 0 *) n := n-1; (* 0 <= n < N *)
        x := buf[out]; out := (out+1) MOD N;
        SEND(nonfull)
    END fetch;
BEGIN n := 0; in := 0; out:= 0;
    Init(nonfull); Init(nonempty)
END Buffer.
```
Interestingly, no critical section control is provided for the body of the deposit and fetch procedures, which might not be suited for true multi-processor parallelism.

#### From Concurrent to Super Pascal

Concurrent Pascal appeared about a year before MODULA. The MODULA report [3] references the Concurrent Pascal Report from 1975 [7]. Not surprisingly both languages share many features, albeit with a different syntax.

Here is an example from [8]:
```
type diskbuffer = monitor(consoleaccess, diskaccess: 
                          resource; base, limit: integer);
        var disk: virtualdisk; sender, receiver: queue;
            head, tail, length: integer;
        
        procedure entry send(block: page);
        begin
            if length = limit then delay(sender);
            disk.write(base + tail, block);
            tail := (tail + 1) mod limit;
            length := length + 1;
            continue(receiver);
        end;
    
        procedure entry receive(var block: page);
        begin
            if length = 0 then delay(receiver);
            disk.read(base + head, block);
            head := (head + 1) mod limit;
            length := length − 1;
            continue(sender);
        end;
    begin “initial statement”
        init disk(consoleaccess, diskaccess);
        head := 0; tail := 0; length := 0;
    end
```
A monitor here is a regular type (which supports multiple instances), not a module. There is also a class type with similar properties as a monitor, but guaranteed at compile time. It is alsoworth mentioning that in Concurrent Pascal the declaration of monitor and class types enclose the procedure declarations - a concept that is later also found in Object Pascal or e.g. Active Oberon. Further, MODULA directly supports the condition variables described in [5], wheras Concurrent Pascal supports a process queue type, which is essentially equivalent to a condition variable.

Until 1978, shared memory was the most common communication mechanism, and semaphores, critical sections, and monitors were among the synchronization mechanisms. In his 1978 paper [11], Hoare demonstrated how both - communication and synchronization - can be realized with just one language element: synchronous communication. Hoare's seminal paper proposes named processes communicating with each other strictly through synchronous message-passing. 

The rendezvous concept of Ada83 has been particularly influenced by this concept [12]; here is a simple, partial example:

```
task timeserver is
  entry read_time(now : out time); 
  entry set_time(new_time : in time); end timeserver;
end timeserver;

task body timeserver is 
begin
    ...
  accept read_time (now : out time) do 
   -- code required to
   -- provide the service
  end;
    ...
end time_server;

-- rendezvous happens here:
timeserver.read_time(time_now) ; 
```
The calling thread blocks until the implementation of read_time is finished and a value is returned to time_now. There are much more complicated options than shown here (including selective waiting) and there are also well-studied issues [12].

In 1984 Hoare and his colleagues refined the theory of CSP into its modern, process algebraic form [13]. In this version, anonymous processes communicate by sending or receiving values from named unbuffered channels; since the channels are unbuffered, the send operation blocks until the value has been transferred to a receiver, thus providing a mechanism for synchronization. This work inspired a number of programming languages such as Occam [15], Squeak [16] and Joyce [14].

Joyce is based on CSP and Pascal and permits unbounded (recursive) activation of agents [14]. An "agent" is a procedure which runs concurrently with other agents. Agents communicate synchronously by means of symbols transmitted through channels. Channels are created dynamically (in contrast to CSP).

Here is the Fibonacci example from [14] page 18:
```
type func = [val(integer)];
agent fibonacci(f: func; x: integer);
    var g, h: func; y, z: integer;
    begin
        if x <= 1 then 
            f!val(x)
        else
            begin
                +g; fibonacci(g, x − 1);
                +h; fibonacci(h, x − 2);
                g?val(y); h?val(z); f!val(y + z)
            end
end;
```
The recursive calls of the "fibonacci" procedure start agents which run concurrently and communicate by means of the dynamically created "g" and "h" channels.

Another example from [14] is this monitor that implements a non-terminating ring buffer accessed via channels:
```
agent buffer(inp, out: stream);
    const n = 10;
    type contents = array [1..n] of integer;
         var head, tail, length: integer;
         ring: contents;
begin
    head := 1; tail := 1; length := 0;
    while true do
        poll
            inp?int(ring[tail]) & length < n −>
                tail := tail mod n + 1;
                length := length + 1|
            out!int(ring[head]) & length > 0 −>
                head := head mod n + 1;
                length := length − 1
        end
end;
```
An empty buffer may input a message only. A full buffer may output only. When the buffer contains at least one and at most n-1 values, it is ready either to input or to output a message.

Here are the relevant syntax definitions:
```
Program = [ ConstantDefinitionPart ] [ TypeDefinitionPart ] 
          AgentProcedure
AgentProcedure = "agent" AgentName ProcedureBlock ";"
AgentStatement = AgentName [ "(" ActualParameterList ")" ]

PortType = "[" Alphabet "]"
Alphabet = SymbolClass { "," SymbolClass }
SymbolClass = SymbolName [ "(" TypeName ")" ]
PortStatement = "+" PortAccess
OutputCommand = PortAccess "!" OutputSymbol
InputCommand = PortAccess "?" InputSymbol
InputSymbol = SymbolName [ "(" VariableAccess ")" ]

PollingStatement = "poll" GuardedStatementList "end"
GuardedStatementList = GuardedStatement 
                      { "|" GuardedStatement }
GuardedStatement = Guard "->" StatementList
Guard = InputOutputCommand [ "&" Expression ]
```

Joyce is an interesting language and even allows indirect naming of ports (and thus passing ports as arguments to agents). A message type in Joyce can be any known type, but explicitly not including port types (although for no obvious reasons; so Joyce doesn't implement the "channels as first-class values" concept yet, later known from Newsqueak [18] and its successors). 

The language is also interesting because it represents the transition of its author from the monitor concept of the seventies to the ideas of CSP. Neither Modula nor Oberon ever made this transition. According to his biography, Per Brink Hansen was supposed to design a special-purpose operating system with parallel processes for a real-time application, but soon reached a point where he found himself unable to write a clear description of what he intended, which was the occasion where he started to design a new parallel programming language, and left his former ones behind. Besides a similar Pascal subset, Joyce and Concurrent Pascal have little in common.

And as if Brink Hansen wanted to make the paradigm shift even clearer, he went on to design yet another CSP-based programming language called SuperPascal [19]. In contrast to Concurrent Pascal and Joyce, which were used to implement real systems, SuperPascal is conceived a "publication language" for the specification of parallel scientific algorithms. Nevertheless, it contains interesting concepts that are worth studying for their reusability in Oberon+.

In contrast to Joyce, where agent procedures were declared and dynamically started, SuperPascal has a "parallel" and "forall" statement to either denote a fixed number of statements for parallel execution, or a parallel execution of the same statement by a dynamic number of processes. As in Joyce, also SuperPascal supports recursive parallel execution. Whereas in Joyce ports were declared with implicit channels, channel types are explicit in SuperPascal. Another interesting difference is the use of built-in open(), send() and receive() procedures instead of the "+", "!" and "?" operators in Joyce.

Here are the relevant syntax definitions:
```
channel-type = "*" "(" message-type-list ")" 
message-type-list = type-identifier { "," type-identifier }

open-statement = “open” “(” open-parameters “)”
send-statement = “send” “(” send-parameters “)”
receive-statement = “receive” “(” receive-parameters “)”

parallel-statement = “parallel” process-statement-list “end”
forall-statement = “forall” index-variable-declaration “do” 
                   element-statement
```
Both in Joyce and SuperPascal, ports or channels are reference types and thus support indirect naming. In both languages a channel can transport different message types, but in Joyce the message type is detected by a token and in SuperPascal by the order of arrival. SuperPascal has no longer a poll statement. The author writes in [20]: "I have not found it necessary to use a statement that enables a parallel process to poll several channels until a communication takes place on one of them. Nondeterministic communication is necessary at the hardware level in a routing network, but appears to be of minor importance in parallel programs for computational science."

#### Ada versus Java

It was already stated that the rendezvous concept of Ada83 has been particularly influenced by CSP. The concurrency features of Ada83 have been criticized in many respects, among other things because of the rendezvous performance overhead. Therefore Ada95 added Protected Types, each instance of which essentially corresponds to a monitor as described by Brinch Hansen and Hoare [2, 5]. 

A protected type in Ada 95 encapsulates some data items, which can only be accessed through the protected type’s operations. It is declared as shown in the following example [33]:
```
protected type Shared_Int is
  procedure Set (Val : in Integer);
  function Get return Integer;
  entry Wait_Until_Zero;
private
  Current : Integer := 0;
end Shared_Int;

protected body Shared_Int is
  procedure Set (Val : in Integer) is
  begin
    Current := Value;
  end Set;
  function Get return Integer is
  begin
    return Current;
  end Get;
  entry Wait_Until_Zero when Current = 0 is
  begin
    null;
  end Wait_Until_Zero;
end Shared_Int;
```
Although protected types are fully integrated into the type system of Ada 95 which supports object-oriented programming, protected types don't support inheritance nor polymorphism. [33] includes a consolidated proposal on how to extend Ada to support object-oriented protected types. Interestingly, this proposal has not yet been implemented with the Ada 2023 release. Though, Ada 2005 introduced Java-like interfaces, which are also applicable to protected types and thus make them object-oriented to a certain degree, but not to the extent proposed in [33].

An important issue when combining object-oriented with concurrency concepts is the so called "inheritance anomaly", which is described in [33] as "the synchronization between operations of a class is not local but may depend on the whole set of operations present for the class. When a subclass adds new operations, it may therefore become necessary to change the synchronization defined in the parent class to account for these new operations." A major cause is the partition of the object state, as it happens with sub and super classes having their own data items. This issue pretty much undermines the benefit of object-orientation and is likely the reason why Ada after nearly thirty years still doesn't support portected type extensions.

In Java, each class becomes a monitor, if a method or block is attributed with the "synchronized" keyword [34]. The attributed method or block automatically becomes a critical section. Each object (including the "class object" associated with a class's static members) has a lock. If a
thread invokes a method marked as synchronized, then it will be blocked unless either the method’s containing object is unlocked or the thread already holds the lock on this object. A Java class may contain both synchronized and unsynchronized methods. Thus invoking a synchronized method does not guarantee that the access will be safe. Ada, in contrast, protects all externally invokable operations on a protected type.

Deciding whether to specify a method as synchronized in Java is not always easy. Unnecessarily making a method synchronized degrades performance and may lead to deadlock in the presence of mutually dependent methods. Failing to specify synchronized when it is needed can cause unpredictable effects. If a synchronized method is inherited, it is also synchronized in the subclass. One of the effects of Java’s thread model being based on OOP is thus a simple
approach to inheriting mutual exclusion protection. However, Java still suffers from the problem, that mutually dependent methods may lead to deadlocks. It is almost inevitable that a Java subclass that requires additional synchronization logic will need to completely re-implement superclass methods containing wait/notification calls, thus breaking encapsulation.

According to Brinch Hansen [35], "the most important security measure [for a parallel programming language]is to check that processes access disjoint sets of variables only and do not interfere with each other in time-dependent ways." He concludes that Java does not support a monitor concept, unless all methods are declared as synchronized and all variables are declared as private, and that parallel threads in Java can access shared variables otherwise, either directly or indirectly, without any synchronization. He didn't see how a compiler could detect such errors and concludes: "It is astounding to me that Java’s insecure parallelism is taken seriously by the programming community, a quarter of a century after the invention of monitors and Concurrent Pascal. It has no merit."

#### The lineage of Go

The Go programming language [27] started in 2007, reached version 1.0 in 2012, and entered the [Stack Overflow survey of most popular technologies](https://insights.stackoverflow.com/survey/) in 2017 for the first time with a popularity grade of 4.2% (vs. Java with 39.3% and C++ with 22.1%); it has grown to 13.24% (vs. Java 30.55% and C++ with 22.42%) in 2023. Go is particularly interesting here because it implements the CSP channel concept in a very popular programming language.

The evolution of Go started in 1985 with a small language called Squeak [16] which already used processes communicating over synchronous channels. The distinguishing "channels as first-class values" concept that is still present and important in Go today, appeared 1989 for the first time in the successor language called Newsqueak [18]. Newsqueak was an interpreted language used by its author to implement a window system. The language has Pascal-like type declarations and assignments, and C-like control structures.

Newsqueak was followed by Alef in 1995 and by Limbo in 1996. Alef, which was used in the Plan 9 operating system from Bell Labs, can be seen as a compiled version of Newsqueak. It is worth noting that the Oberon system has influenced the parts of Plan 9, which were implemented by the author of Newsqueak [26]. Limbo is very similar to Newsqueak as well; it was used to implement parts of the Inferno operating system from Bell Labs, which is a descendant of Plan 9 and still in use today.

Go mostly adopted the concurrency concept from Newsqueak with some additional features from other languages, such as buffered channels.

But even though Go is designed entierly on the "channels as first-class values" principle, with dedicated syntax for channel communication, it still includes the whole array of synchronization primitives in a dedicated package of the standard library. Even if it is possible to to emulate low-level concurrency primitives using higher-level concepts like channels [28], it's in many cases just not practical. An article that has met with a broad response [31], expresses it as follows: "Unfortunately, as it stands, there are simply a surprising amount of problems that are solved better with traditional synchronization primitives than with Go’s version of CSP. [..] The summary here is that you will be using traditional synchronization primitives in addition to channels if you want to do anything real." [32] found, that "it is as easy to make concurrency bugs with message passing as with shared memory, sometimes even more"; other important finding are that mutexes are used twice as often on average as channels, and that message passing is the main cause of blocking bugs.

#### Concurrency in Oberon

While MODULA supports concurrency directly and Modula-2 via a module (i.e. library), there is no support for concurrency at all in Oberon. In [9] Wirth writes: "The system Oberon does not require any language facilities for expressing concurrent processes. The pertinent rudimentary features of Modula, in particular the coroutine, were therefore not retained. This exclusion is merely a reflection of our actual needs within the concrete project, but not on the general relevance of concurrency in programming."

The same applies to Oberon-2 and Oberon-07, but there was an independent effort to include coroutines in the standard library. The Oakwood guidelines [10] include an optional Coroutines module with "non-preemptive threads each with its own stack but all sharing a common address space" with the following interface:

```
DEFINITION Coroutines;
    TYPE Coroutine = RECORD END; Body = PROCEDURE;
    PROCEDURE Init (body: Body; stackSize: LONGINT; 
                    VAR cor: Coroutine);
    PROCEDURE Transfer (VAR from, to: Coroutine);
END Coroutines.
```

The Init procedure creates a new, suspended coroutine, whereas the Transfer procedure transfers control from the currently executing coroutine to the one passed to the Transfer procedure. This interface is very similar to the low-level process support in Modula-2 [6] based on which the features in the "Processes" module are implemented.

For the sake of completeness, the work described in [17] shall be mentioned where the Oberon system was extended by threads and synchronization primitives, but without changing the Oberon language. The work demonstrates, that also in Oberon, which has a garbage collector, the implementation of (low-level) multithreading features can be orthogonal to the language, just as a set of library data types and procedures.

So far the most consequent concurrency implementation of the Oberon language family can be found in Active Oberon ([22] and [21] section 4). Active Oberon regards objects as processes and provides an "object-centered access protection" and "activity control". An object type declaration can have a statement block which runs in a new thread on object instantiation. Type-bound procedures of the object type can be marked EXCLUSIVE, in which case their body becomes a critical section. In addition there is a built-in AWAIT procedure which suspends the calling thread until the boolean expression argument becomes TRUE. The authors of Active Oberon call such an object type an "active object".

Here is the (slightly modified) example of [21] page 157:
```
MODULE Eratosthenes; 
IMPORT Out, Buffers;
CONST N = 2000; Terminate = -1;

TYPE Sieve = OBJECT (Buffers.Buffer)
    VAR prime, n: INTEGER; next: Sieve;

    PROCEDURE & Init;
    BEGIN
        Initˆ;
        prime := 0; next := NIL
    END Init;

    BEGIN {ACTIVE}
        LOOP
            Get(n);
            IF n = Terminate THEN
                IF next # NIL THEN next.Put (n) END;
                EXIT
            ELSIF prime = 0 THEN
                Out.Int(n, 0); Out.String(" is prime"); 
                Out.Ln;
                prime := n;
                NEW (next)
            ELSIF (n MOD prime) # 0 THEN
                next.Put (n)
            END
        END
END Sieve;

PROCEDURE Start*;
VAR s: Sieve; i: INTEGER;
BEGIN
    NEW(s);
    FOR i := 2 TO N-1 DO s.Put (i) END;
    s.Put(Terminate)
END Start;

END Eratosthenes.
```

But what actually is an "active object"? An active object essentially corresponds to an actor as defined in the Actor Model [24]. This conception of the term was already established in the well-known book "Pattern languages of program design" in 1996 [23]. An "Active Object [..] decouples method execution from method invocation to enhance concurrency and simplify synchronized access to an object that resides in its own thread of control. [..] This decoupling is designed so the client thread appears to invoke an ordinary method. This method is automatically converted into a method request object and passed to another thread of control, where it is converted back into a method and executed on the object implementation" [23]. An example where this kind of decoupling is e.g. realized is Erlang [25], one of the most successful actor languages, where named processes communicate via an asynchronous message passing system (in contrast to synchronous message passing in CSP), and use the receive primitive to retrieve messages that match desired patterns.

From this perspective, object types in Active Oberon are nothing but plain old monitors [5], as already implemented in Concurrent Pascal [7] or MODULA [3] in the seventies (see above). The fact that a thread is associated with the instance of the object type merely means that no extra data type needs to be introduced for a thread; but the thread has no special role or exclusivity in relation to the object instance otherwise. The AWAIT concept is a bit more comfortable than the extra signal datatype and send() procedure as e.g. in MODULA, but also less flexible; and beyond that, the concept was already proposed by Brinch Hansen in 1972 [2]. What is even more surprising is the fact that the monitor concept appears to have been adopted completely uncritically. Neither in [22] nor in any other publication of the group there is any indication that the state of the art has been evaluated, or why a concept from the early seventies should be more suitable than all the more recent ones. [22] refrains entirely from referencing the relevant original publications; [21] at least references Hoare's monitor paper [5], and even the original CSP paper [11] (but strangely only on the subject of "language interoperability", without acknowledging its importance for to the design of a concurrent programming language). The authors did neither seem to have been concerned about the "inheritance anomaly" discussed above [33], nor the fact that Active Oberon essentially shares all the disadvantages with Java that Brinch Hansen criticized [35].

All in all, it has to be concluded that Oberon either omits the important topic of concurrency altogether, or only considers it in an outdated form that ignored almost thirty years of research when published.

#### Influencing factors

The following elements are relevant for the design decisions on how to add concurrency to Oberon+.

##### Types of implementation

Concurrency can be implemented in a programming language by

- a library (either implemented in the present or a foreign language)
- built-in types and procedures (without special syntax)
- dedicated syntax

As we saw, Modula-2 does well with a library-only implementation of concurrency without dedicated syntax or built-in types or procedures. In contrast, Concurrent Pascal, Ada83, Joyce and Newsqueak decided for a dedicated syntax. MODULA and Super Pascal, on the other hand, use a mixture of dedicated syntax and built-in procedures. Yet another approach is used in Active Oberon, which added a generic attribute syntax, where attributes are just identifiers with arbitrary dedication of meaning.

When do we need dedicated syntax? Apparently no dedicated syntax is needed for low-level concurrency primitives like mutexes, condition variables (including wait and signal procedures) and threads, which can be well implemented even with a library. In Modula-2, the Processes module can even be implemented in Modula-2 itself to a certain degree, using a few low-level primitives represented with built-in types and procedures. On the other hand, structural, more high-level features like monitors or critical sections are unhandy or even infeasible without some degree of dedicated syntax. Message passing and channels are a borderline case. If e.g. type-safe channels are required, the language either needs type casting features to bypass the type checking of the compiler, and even some kind of address operator if type casting of structured value types is required, or a dedicated syntax with typed channels; but in case of channels also built-in types and procedures would do the job in principle. So there is a trade-off and it is therefore important to carefully balance the options.

Another point to consider is the support for static analysis, which is much harder or even infeasible if concurrency is just a library feature.

##### Philosophy of the language

Every programming language follows a philosophy, i.e. a set of guiding principles. 

For example, languages such as C or C++ want to give the programmer maximum flexibility so that compiler checks can be easily bypassed, but at the price of a high susceptibility to errors and great responsibility on the part of the programmer. In such a language, the programmer should probably be trusted to work with low-level concurrency primitives as well.

On the other hand, in languages that are committed to a certain type of concurrency, this should also be manifested in the syntax of the language, as it is the case e.g. in Concurrent Pascal, Joyce, Newsqueak and Go.

But there are also languages that want to support the programmer on as many levels as possible; these include e.g. C++, where you can build high-level constructs yourself with low-level primitives and even extend the language itself, e.g. by means of operator overloading; or languages such as Ada, where almost every conceivable use case is mapped with a dedicated syntax; or languages such as Lisp and Smalltalk, which are intended to create any abstractions with as little and generic syntax as possible.

Oberon's official philosophy is "as simple as possible"; however, if you look at the implementation of the Oberon system, you can also see that the language is optimized for polymorphic message passing and handling, and that this is not only a guiding principle of the system, but also of the programming language used to implement this system. Instead of polymorphic method dispatch - as in other object-oriented languages - there are message handlers, which are plain procedures or procedure variables, with a parameter of record type passed by reference; this record type can be regarded as a "message type", and the same handler is also compatible with all subtypes of this message type. Not surprisingly, Oberon has control structures which discriminate by the dynamic type of an expression. Concurrency based on message passing therefore looks like an obvious idea for Oberon.

##### Language complexity

If concurrency can be fully covered by a library, this obviously doesn't increase the complexity of the language - though it very likely increases the complexity of the code using the library. On the other hand, if we add low-level concurrency constructs to the syntax, both the language and the code written in the language become more complex. It is reasonable to assume that if the right high-level constructs are added to the language, a little extra complexity in the language can contribute to a significant complexity reduction in the code that is written in the language. Conversely, it can also be observed that a language with overly minimalist properties simply shifts the complexity into the code. The challenge is therefore again to find the right balance.

As already mentioned, Hoare demonstrated with CSP, how data exchange among threads and the synchronisation of this exchange can be combined, thus reducing complexity. 

We could ask why an extra syntax is needed to send to or receive from a channel, as e.g. in Joyce, Newsqueak or Go. Super Pascal has demonstrated that this can be well done with a send() and receive() built-in procedure. We could even get rid of the poll (Joyce) or select (Newsqueak, Go) statement, used to wait on multiple channels, by extending send() and receive(), thus saving several language constructs at once. In Super Pascal, `receive(c, v1, ... , vn)` is an abbreviation for `receive(c,v1), ..., receive(c,vn)`. We could instead define a receive and send function of the form `receive(c1,v1,c2,v2,...,cn,vn)`, which returns the index of the channel which was actually selected, with the same effect as a select statement in Go. 

But how does this reduce complexity? First of all, the concept of built-in procedures is already well established in Wirth languages, also spelling out identifiers; and we can reuse the CASE statement together with the send or receive function to dispatch to the statement list associated with the selected channel. This neither makes the language more complex, nor is it more effort to use than a dedicated syntax.

Adding the monitor concept to Oberon causes more complexity, because a monitor is similar but not identical to both a record and a module, and we have to take care which (bound) procedures should guarantee exclusive access, and in case of a record inheritance has to be considered as well; and there are even more issues, e.g. if and how to handle recursive locks. Also associating threads with record (i.e. class) instances adds extra complexity because of the unrelated life cycle of the instance and the thread; its simpler to just provide a means to run an existing procedure in a separate thread (as e.g. done in Go with the "go" keyword).

##### Duality of concepts

It is well known which low-level concurrency primitives exist and how these can be combined to realize higher-level concepts like monitors or channels [2] (the "mutex semaphore" as the "most basic quantity [..] taking care of the mutual exclusion of the critical sections" was published by Dijkstra already in 1965; the "condition variable" together with the "wait" and "signal" operations were co-invented by Brinch Hansen and Hoare in the context of monitors in 1973). 

As already mentioned it is possible to implement low-level concurrency primitives in a library without dedicated syntax. Thus a common approach is to only offer low-level concurrency primitives (in the standard library, or as built-in types and procedures, or as dedicated syntax) and leave it to the user of the programming language to construct higher-level concurrency concepts from these low-level primitives. 

But it is also possible to emulate low-level concurrency primitives based on the higher-level concepts provided by a programming language. For example, [28] demonstrates how to simulate a semaphore using a monitor, and concludes: "This is important theoretically since it shows that we have not lost any expressive power in the transition from semaphores to monitors". 

How a semaphore can be implemented using Go channels is demonstrated in [29]; here is the relevant code:
```
type semaphore struct {    
    semC chan struct{}
}

func New(maxConcurrency int) Semaphore {    
    return &semaphore{        
        semC: make(chan struct{}, maxConcurrency),    
    }
}

func (s *semaphore) Acquire() {    
    s.semC <- struct{}{}
}

func (s *semaphore) Release() {    
    <-s.semC
}
```
It seems therefore feasible to stick to a high-level concurrency concept in a programming language. But as e.g. demonstrated in [31], a single high-level concept is likely insufficient to decently represent all real-world problems. Of course, one can always offer low-level primitives via a library, as e.g. done in Go [27]. But this might lead to yet to other issues caused by the interaction of high- and low-level concepts, as e.g. demonstrated in [32].


#### Elaboration of Oberon+ concurrency

From the previous sections we can conclude, that message passing based on channels is a good fit for Oberon (in terms of simplicity and congruence with polymorphic message handling), and that this concept was well studied and established as an improvement over low-level concurrency primitives. But we also saw that also monitors should be considered for good reasons, because even in Go, low-level synchronization primitives are still more frequentely used than channels, and there are problems where channels are too complicated, even considering their duality with monitors. Where Wirth in 1978 concluded, that there was "no clear favorite way to express and control concurrency, and hence no set of language constructs that clearly offered themselves for inclusion" [4], this situation has changed today, more than fourty years later. 

Also the findings in [33] and [35], and the feedback from public discussions about previous versions of this paper [30] have been taken into consideration.

The concept is introduced with as few syntax extensions as possible and reasonable, and the already established concept of built-in procedures is preferred over a library implementation. 

The Oberon+ syntax shall be extended as follows:

```
type = NamedType | enumeration | ArrayType 
       | RecordType | PointerType | ProcedureType 
       | ChannelType | MonitorType

ChannelType = CHANNEL [ length ] OF type

MonitorType = MONITOR FieldList { [';'] FieldList} END
```
Note that only upper-case versions of reserved words and names are shown here, but Oberon+ also supports lower-case versions.

A channel declaration is similar to an array declaration. The new reserved word `CHANNEL` declares a channel of a specific base type. In contrast to Joyce and similarly to Go, a channel has exactly one type, and this type is not restricted (i.e. it can also be pointer type). If length is missing or 0, an unbuffered channel is declared; if length is a natural number, a buffered channel is declared.

A channel is a value type with the special rule that it cannot be copied (i.e. assigned or passed as a value parameter). If a channel is a field of a record type, instances of this record type or its subtypes can no longer be copied. Neither a channel nor a record with a channel field can be the type of a local variable. A channel can be passed by reference or be the base type of a pointer. Channel types are compatible if they have equal base types. If a channel is the base type of a pointer, it has to be allocated by the NEW() built-in procedure; if the length in the channel declaration is missing, NEW() can have an optional length parameter.

A monitor declaration is similar to a record declaration. It uses the new reserved word `MONITOR` and has a field list like a record, but it doesn't have a base type (i.e. it cannot be extended) and fields cannot have export marks and can only be accessed from the procedures bound to the monitor type. Accordingly, procedures may be associated (i.e. bound) with a monitor type declared in the same scope, in which case their body becomes a critical section. A procedure bound to a monitor cannot call the RAISE built-in procedure. If a procedure bound to a monitor calls a procedure which directly or indirectly calls RAISE, this call must be protected by PCALL.

A monitor, like a channel, is a value type with the special rule that it cannot be copied (i.e. assigned or passed as a value parameter). If a monitor is a field of a record type, instances of this record type or its subtypes can no longer be copied. Neither a monitor nor a record with a monitor field can be the type of a local variable. A monitor can be passed by reference or be the base type of a pointer. The same compatibility rules as for records without base types apply. If a monitor is the base type of a pointer, it has to be allocated by the NEW() built-in procedure.

In addition the following procedures are predeclared (pseudo-syntax):

```
PROCEDURE SEND(VAR c: CHANNEL OF T; v: T );

PROCEDURE RECEIVE(VAR c: CHANNEL OF T; VAR v: T );

PROCEDURE AWAIT();

PROCEDURE SIGNAL();
```

The SEND procedure transmits the value of the actual parameter v to the channel c. If the channel c is unbuffered or the buffer is full, the call blocks until another thread calls RECEIVE on the channel.

The RECEIVE procedure reads a value from the channel c and stores it in the variable v. If the channel c is unbuffered or the buffer is empty, the call blocks until another thread calls SEND on the channel.

SEND and RECEIVE modify the channel state, i.e. don't work with IN parameters or receivers.

In Go - in contrast to its predecessors - channels can be closed; Oberon+ doesn't support this feature because it is not strictly necessary and has a complicated semantics which contradicts the Oberon philosophy of simplicity.

The AWAIT and SIGNAL procedures can only be called from a procedure bound to a monitor.

The AWAIT procedure suspends the calling thread until SIGNAL is called. More than one thread can be waiting on the same MONITOR instance. Calling SIGNAL wakes up the next waiting thread.

An extended version of the WITH statement can be used to choose which of a set of possible SEND or RECEIVE operations will proceed:
```
WITH 
  SEND(c_i, v_i) DO Si
  | RECEIVE(c_j, v_j) DO Sj
  ELSE S
END;
```
For this purpose the definition of Guard syntax is slightly extended:
```
Guard = qualident ':' qualident 
        | 'SEND' ActualParameters 
        | 'RECEIVE' ActualParameters
```

The extended WITH statement selects the first of the channels ready to communicate and executes the listed SEND or RECEIVE guard. If several channels are ready to communicate at once, one of them is randomly selected. If no ELSE clause is present, the statement blocks until at least one channel is ready to communicate. If ELSE is present, the statement doesn't block but executes the statements of the ELSE clause.

```
PROCEDURE FORK(p: ProcedureType 
              [ ';' a1: T1 { ';' aN: Tn } ]);
```
Calling FORK executes the procedure p with actual arguments a1 to aN in a new thread. The procedure p cannot have variable parameters, and must not be a predeclared, nor a type-bound procedure, nor may it access local variables or parameters declared in outer (type-bound) procedures or call procedure which access local variables or parameters declared in outer (type-bound) procedures. The actual arguments must be compatible with the formal parameters of p. If a new thread cannot be started, the program halts. The specification does not prescribe what type of thread this should be (i.e. the program cannot assume that threads are e.g. light-weight).

 
#### References

- [1] ISO/IEC/IEEE 24765:2017 Systems and software engineering — Vocabulary
- [2] Hansen, P.B. (2002): The Origin of Concurrent Programming. Springer Science+Business Media, New York
- [3] Wirth, N. (1976): MODULA - a language for modular multiprogramming. Berichte des Instituts für Informatik 18, ETH Zürich
- [4] Wirth, N. (2007): Modula-2 and Oberon. In Proceedings of the third ACM SIGPLAN conference on History of programming languages (HOPL III), New York
- [5] Hoare, C.A.R. (1974): Monitors: An Operating System Structuring Concept. Comm. ACM 17, 10, pp. 549 – 557
- [6] Wirth, N. (1988): Programming in Modula-2, 4th Edition. Springer, Berlin
- [7] Hansen, P.B. (1975): Concurrent Pascal Report. California Institute of Technology
- [8] Hansen, P.B. (1975): The programming language Concurrent Pascal. IEEE Transactions on Software Engineering
- [9] Wirth, Niklaus (1988): From modula to Oberon. Softw. Pract. Exper. 18, 7, 661–670
- [10] Cadach, A. et al. (1993): The Oakwood Guidelines for Oberon-2 Compiler Developers. http://www.math.bas.bg/bantchev/place/oberon/oakwood-guidelines.pdf
- [11] Hoare, C.A.R. (1978): Communicating sequential processes. Commun. ACM 21, 8, 666–677
- [12] Burns, Alan; Lister, Andrew M.; Wellings Andrew J (1987): A review of Ada tasking. Springer-Verlag, Berlin
- [13] Brookes, S. D.; Hoare, C. A. R.; Roscoe, A. W. (1984): A Theory of Communicating Sequential Processes. J. ACM 31, 3
- [14] Hansen, P.B. (1987): Joyce—A programming language for distributed systems. Softw: Pract. Exper., 17: 29-50
- [15] Inmos Corp. (1984): Occam Programming Manual. Prentice Hall Trade
- [16] Cardelli, L.; Pike, R. (1985): Squeak: a language for communicating with mice. SIGGRAPH Comput. Graph. 19, 3 (Jul. 1985), 199–204
- [17] Lalis, S.; Sanders, B.A. (1994): Adding concurrency to the Oberon system. In Proceedings of the international conference on Programming languages and system architectures. Springer-Verlag, Berlin
- [18] Pike, R. (1989): A Concurrent Window System. Comput. Syst. 2(2): 133-153
- [19] Hansen, P.B. (1994): The programming language SuperPascal. Softw. Pract. Exper. 24, 5
- [20] Hansen, P.B. (1994): SuperPascal—a publication language for parallel scientific computing. Concurrency: Pract. Exper., 6: 461-483
- [21] Reali, P.R.C. (2003): Using Oberon’s active objects for language interoperability and compilation [Doctoral Thesis, ETH]. https://doi.org/10.3929/ethz-a-004554781 
- [22] Gutknecht, J. (1997): Do the Fish Really Need Remote Control? A Proposal for Self-Active Objects in Oberon. In Proceedings of the Joint Modular Languages Conference on Modular Programming Languages . Springer-Verlag, Berlin  
- [23] Lavender, R.G.; Schmidt, D.C. (1996): Active object: an object behavioral pattern for concurrent programming. Pattern languages of program design 2. Addison-Wesley Longman Publishing Co., Inc., USA, 483–499
- [24] Agha, G. (1986): Actors: a model of concurrent computation in distributed systems. MIT Press, Cambridge, MA, USA
- [25] Armstrong, J. (2007): A history of Erlang. In Proceedings of the third ACM SIGPLAN conference on History of programming languages (HOPL III). Association for Computing Machinery, New York, NY, USA
- [26] Pike, R. (1994): Acme: a user interface for programmers. In Proceedings of the USENIX Technical Conference on USENIX. USENIX Association, USA
- [27] Donovan, A.A.; Kernighan, B.W. (2015): The Go Programming Language (1st. ed.). Addison-Wesley Professional
- [28] Ben-Ari, M. (1982): Principles of Concurrent Programming. Prentice Hall Professional Technical Reference
- [29] https://levelup.gitconnected.com/go-concurrency-pattern-semaphore-9587d45f058d
- [30] https://news.ycombinator.com/item?id=38764412
- [31] https://www.jtolio.com/2016/03/go-channels-are-bad-and-you-should-feel-bad/
- [32] Tu, T.; Liu, X.; Song, L.; and Zhang, Y. (2019): Understanding Real-World Concurrency Bugs in Go. In Proceedings of the Twenty-Fourth International Conference on Architectural Support for Programming Languages and Operating Systems. ACM, New York
- [33] Wellings, A.J.; Johnson, B.; Sanden B.; Kienzle, J.; Wolf, T.; and S. Michell (2000): Integrating object-oriented programming and protected objects in Ada 95. ACM Trans. Program. Lang. Syst. 22, 3, 506–539
- [34] Brosgol, B.M. (1998): A comparison of the concurrency features of Ada 95 and Java. In Proceedings of the 1998 annual ACM SIGAda international conference on Ada (SIGAda '98). Association for Computing Machinery, New York, USA, 175–192
- [35] Hansen, P.B. (1999): Java's insecure parallelism. SIGPLAN Not. 34, 4, 38–45



  




  

  

