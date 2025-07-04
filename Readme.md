# CakeML

A MLIR recipe for ocaml cake. Taste of ML language from MLIR perspective.

**This is a ongoing project. It's a proof of concept for using MLIR for functional language syntax and will not be suitable for production in any time. following features may be done in future**

language syntax:
- [ ] `mut` keyword allow variable mutable
- [ ] variant, tuple, list type

compiler opts:
- [ ] Alpha transformation to solve name conflict
- [ ] Variable capture as formal parameter
- [ ] Inline let/lambda to some extent

I would like to achieve these above features based on optimization passes or extending LetAlg dialect we already have. I would also like to finish a compact runtime data structure design

- [ ] efficient stack frame

## ML in MLIR dialect

- ML's let sytle
```ocaml
let x = 1 in let y = 2 in x + y
```

in LetAlg dialect
```llvm
module {
  func.func @test_function() {
    %c1_i32 = arith.constant 1 : i32
    %c2_i32 = arith.constant 2 : i32
    %0 = letalg.let %c1_i32, %c2_i32 : i32, i32 (%arg0: i32,%arg1: i32){
      %2 = arith.addi %arg0, %arg1 : i32
      %3 = "letalg.yield"(%2) : (i32) -> i32
    } -> i32
    %1 = "letalg.yield"(%0) : (i32) -> i32
  }
}
```
Adjacent `let` variable bindings will be defined in same block of LetAlg dialect. This expression, `%0 = letalg.let %c1_i32, %c2_i32 : i32, i32 (%arg0: i32,%arg1: i32)`, expresses actual parameter `%c1_i32` and `%c2_i32` will be bound to formal parameter `%arg0` and `%arg1`.

- lambda
```ocaml
let f x = x + 10 in f 2
```

in LetAlg dialect
```llvm
module {
  func.func @test_function() {
    %0 = letalg.lambda "f" (%arg0: i32){
      %c10_i32 = arith.constant 10 : i32
      %3 = arith.addi %arg0, %c10_i32 : i32
      %4 = "letalg.yield"(%3) : (i32) -> i32
    } -> (i32) -> i32
    %1 = letalg.let %0 : (i32) -> i32 (%arg0: (i32) -> i32){
      %c2_i32 = arith.constant 2 : i32
      %3 = "letalg.apply"(%arg0, %c2_i32) : ((i32) -> i32, i32) -> i32
      %4 = "letalg.yield"(%3) : (i32) -> i32
    } -> i32
    %2 = "letalg.yield"(%1) : (i32) -> i32
  }
}
```

`%1 = letalg.let %0 : (i32) -> i32 (%arg0: (i32) -> i32){` expression `let` op take a function type argument `(i32) -> i32`: `%arg0` as formal parameter and `%0` as actual parameter.

- currying

```ocaml
let f x y = x + y + 10 in f 2
```

in LetAlg dialect
```llvm
module {
  func.func @test_function() {
    %0 = letalg.lambda "f" (%arg0: i32,%arg1: i32){
      %3 = arith.addi %arg0, %arg1 : i32
      %c10_i32 = arith.constant 10 : i32
      %4 = arith.addi %3, %c10_i32 : i32
      %5 = "letalg.yield"(%4) : (i32) -> i32
    } -> (i32, i32) -> i32
    %1 = letalg.let %0 : (i32, i32) -> i32 (%arg0: (i32, i32) -> i32){
      %c2_i32 = arith.constant 2 : i32
      %3 = "letalg.apply"(%arg0, %c2_i32) : ((i32, i32) -> i32, i32) -> ((i32) -> i32)
      %4 = "letalg.yield"(%3) : ((i32) -> i32) -> ((i32) -> i32)
    } -> (i32) -> i32
    %2 = "letalg.yield"(%1) : ((i32) -> i32) -> ((i32) -> i32)
  }
}
```

`%1 = letalg.let` return type is `(i32) -> i32`. This `let` op take function type `(i32, i32) -> i32` and only provide the first parameter and return the curried function.

## Passes

There only a few rewriting/optimization passes right now. It's in very primitive stage. An example of rewriting before and after

input is following. lambda `f` has a capture variable from outer closure.
```
let a = 1 in let f x = x + a + 10 in f 2
```

before. Following is initial form of letalg representation, which is nested. This nested representation is good expressive for input in natural because ml's syntax is deeply nested.
```
func.func @test_function() {
  %0 = letalg.let (){
    %c1_i32 = arith.constant 1 : i32
    %2 = letalg.lambda "f" (%arg0: i32){
      %5 = arith.addi %arg0, %c1_i32 : i32
      %c10_i32 = arith.constant 10 : i32
      %6 = arith.addi %5, %c10_i32 : i32
      %7 = "letalg.yield"(%6) : (i32) -> i32
    } -> (i32) -> i32
    %c2_i32 = arith.constant 2 : i32
    %3 = "letalg.apply"(%2, %c2_i32) : ((i32) -> i32, i32) -> i32
    %4 = "letalg.yield"(%3) : (i32) -> i32
  } -> i32
  %1 = "letalg.yield"(%0) : (i32) -> i32
}
```

after. Passes like Closure conversion and declaration lift denest the structure. It will made easy to lower to next step low level dialect.
```
func.func @test_function() {
  %c1_i32 = arith.constant 1 : i32
  %0 = letalg.lambda "f" (%arg0: i32,%arg1: i32){
    %3 = arith.addi %arg1, %arg0 : i32
    %c10_i32 = arith.constant 10 : i32
    %4 = arith.addi %3, %c10_i32 : i32
    %5 = "letalg.yield"(%4) : (i32) -> i32
  } -> (i32) -> i32
  %1 = letalg.let %c1_i32, %0 : i32, (i32) -> i32 (%arg0: i32,%arg1: (i32) -> i32){
    %c2_i32 = arith.constant 2 : i32
    %3 = "letalg.apply"(%arg1, %arg0, %c2_i32) : ((i32) -> i32, i32, i32) -> i32
    %4 = "letalg.yield"(%3) : (i32) -> i32
  } -> i32 attributes {declCnt = 2 : i32}
  %2 = "letalg.yield"(%1) : (i32) -> i32
}
```