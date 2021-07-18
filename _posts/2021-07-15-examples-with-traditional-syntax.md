---
layout: post
title:  Code Examples using traditional Oberon syntax
author: Rochus Keller
---

The following examples correspond to the ones on the front page and have the same functionality, but use the more traditional Oberon syntax also supported in Oberon+. The compiler recognizes the syntax version by the MODULE keyword; if it is written in small caps, the new syntax is assumed. 

### Procedural Programming
```
MODULE Fibonacci;
  VAR res: INTEGER;
  PROCEDURE calc*( n : INTEGER ): INTEGER;
    VAR a, b: INTEGER;
  BEGIN IF n > 1 THEN 
      a := calc(n - 1);
      b := calc(n - 2);
      RETURN a + b
    ELSIF n = 0 THEN
      RETURN 0
    ELSE 
      RETURN 1
    END
  END calc;
BEGIN
  res := calc(21);
  ASSERT( res = 10946 )
END Fibonacci.
```

### Generic Programming
```
MODULE Collections<T>;
  TYPE Deque* = POINTER TO RECORD
                      data: POINTER TO ARRAY OF T;
                      size: INTEGER END;
       Iterator* = RECORD END;
       
  PROCEDURE createDeque*(): Deque;
    CONST initial_len = 50;
    VAR this: Deque;  (* this is initialized to nil *)
  BEGIN 
    NEW(this); NEW(this.data,initial_len); 
    RETURN this 
  END createDeque;
  
  PROCEDURE (this: Deque) append*(IN element: T);
  BEGIN 
    IF this.size = LEN(this.data) THEN ASSERT(FALSE) END;
    this.data[this.size] := element; INC(this.size)
  END append;
  
  PROCEDURE (VAR this: Iterator) apply*(IN element: T) END;
  
  PROCEDURE (this: Deque) forEach*(VAR iter: Iterator);
    VAR i: INTEGER;
  BEGIN FOR i := 0 TO this.size-1 DO 
    iter.apply(this.data[i]) END;
  END forEach
END Collections.
```

### Object Oriented Programming
```
MODULE Drawing;
  IMPORT F := Fibonacci;
         C := Collections<Figure>;
  	
  TYPE Figure* = POINTER TO RECORD
                            position: RECORD 
                                      x,y: INTEGER 
                            END END;  
                     
       Circle* = POINTER TO RECORD (Figure) 
                            diameter: INTEGER END;
       Square* = POINTER TO RECORD (Figure) 
                            width: INTEGER END; 
                     
  PROCEDURE (this: Figure) draw*() END;
  PROCEDURE (this: Circle) draw*() END;
  PROCEDURE (this: Square) draw*() END;
    
  VAR figures: C.Deque; circle: Circle;
       square: Square;
    
  PROCEDURE drawAll*();
    TYPE I = RECORD(C.Iterator) count: INTEGER END;
    VAR i: I; (* count is initialized to zero *)
    PROCEDURE (VAR this: I) apply( IN figure: Figure );
    BEGIN figure.draw(); INC(this.count) 
    END apply;
  BEGIN
    figures.forEach(i);
    ASSERT(i.count = 2);
  END drawAll;
BEGIN figures := C.createDeque();
  NEW(circle); circle.position.x := F.calc(3); 
  circle.position.y := F.calc(4); circle.diameter := 3;
  figures.append(circle);
  NEW(square); square.position.x := F.calc(5); 
  square.position.y := F.calc(6); square.width := 4;
  figures.append(square);
  drawAll()
END Drawing. 
```
### Unicode Support
```
MODULE Unicode;
  VAR
    str: ARRAY 32 OF CHAR;
    ustr: ARRAY 32 OF WCHAR;
BEGIN
  str := "Isto é português";
  ustr := "美丽的世界，你好!" + " " + str;
  PRINTLN(ustr) 
  (* prints "美丽的世界，你好! Isto é português" *)
END Unicode.
```
