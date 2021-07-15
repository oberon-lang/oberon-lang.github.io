## Code Examples

### Procedural Programming
```
module Fibonacci
  proc calc*( n : integer ): integer
    var a, b: integer // comma is optional
  begin
    if n > 1 then 
      a := calc( n - 1 )
      b := calc( n - 2 )
      return a + b
    elsif n = 0 then 
      return 0
    else 
      return 1
    end
  end calc
  var res: integer
begin
  res := calc(21)
  assert( res = 10946 )
end Fibonacci
```

### Generic Programming
```
module Collections<T>
  type Deque* = pointer to record
                      data: pointer to array of T
                      size: integer end
  proc createDeque*(): Deque 
  const initial_len = 50
  var this: Deque  // this is initialized to nil
  begin 
    new(this); new(this.data,initial_len) // semicolon is optional
    return this 
  end createDeque
  
  proc (this: Deque) append*(in element: T)
  begin 
    if this.size = len(this.data) then assert(false) end
    this.data[this.size] := element inc(this.size) 
  end append
  
  type Iterator* = record end
  proc (var this: Iterator) apply*(in element: T) end
  
  proc (this: Deque) forEach*(var iter: Iterator)
  var i: integer
  begin 
    for i := 0 to this.size-1 do iter.apply(this.data[i]) end
  end forEach
end Collections
```

### Object Oriented Programming
```
module Drawing
  import F := Fibonacci
  type Figure* = pointer to record
                   position: record x,y: integer end
                end  
  proc (this: Figure) draw*() end
    
  type
     Circle* = pointer to record (Figure) diameter: integer end
     Square* = pointer to record (Figure) width: integer end 
  proc (this: Circle) draw*() end
  proc (this: Square) draw*() end
    
  import C := Collections<Figure>
    
  var figures: C.Deque
       circle: Circle
       square: Square
    
  proc drawAll()
    type I = record(C.Iterator) count: integer end
    proc (var this: I) apply( in figure: Figure ) 
    begin figure.draw(); inc(this.count) end apply
    var i: I // count is initialized to zero
  begin
    figures.forEach(i)
    assert(i.count = 2)
  end drawAll
begin 
  figures := C.createDeque()
  new(circle)
  circle.position.x := F.calc(3); circle.position.y := F.calc(4)
  circle.diameter := 3
  figures.append(circle)
  new(square)
  square.position.x := F.calc(5); square.position.y := F.calc(6)
  square.width := 4
  figures.append(square)
  drawAll()
end Drawing  

```
### Unicode Support
```
module Unicode
  var
    str: array 32 of char
    ustr: array 32 of wchar
begin
  str := "Isto é português"
  ustr := "美丽的世界，你好!" + " " + str
  println(ustr) // prints "美丽的世界，你好! Isto é português"
end Unicode
```
### More Examples
If you are looking for representative size examples, see

- [the Oberon+ version of the "Are we fast yet" benchmark suite](https://github.com/rochus-keller/Oberon/tree/master/testcases/Are-we-fast-yet)
- [a compatible version of the Oberon System (written in Oberon-07)](https://github.com/rochus-keller/OberonSystem)
- [the Oberon+ version of a subset of the Blackbox Framework (work in progress)](https://github.com/rochus-keller/BlackboxFramework/tree/master/Minimal)

## Documentation

- [The Programming Language Oberon+ (language specification)](https://github.com/oberon-lang/specification/blob/master/The_Programming_Language_Oberon%2B.adoc)
- TBD

## Implementation

TBD: IDE, screenshot, binaries url, source code url

## Blog

TBD: Oberon History and language versions

TBD: Difference Oberon+ to Oberon-2
