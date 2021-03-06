---
title: A change of undefined behavior in BuckleScript 4.0.7
---

In the latest BuckleScript release, we introduced a minor change in the codegen which broken some user libraries. Note this change only broke the code in the FFI boundary(the interop between JS).

In the early days of BuckleScript, there is no built-in uncurried calling convention support, since OCaml is a curried language, which means every function has arity one, so there is no way to express that a function has arity zero, this makes some interop challenging. In the mocha unit test library, it expects its callback to be function of arity zero.

To work around this issue, before this release, we did a small codegen optimization, for a function of type `unit -> unit`, if its argument is not used, we remove its argument in the output.

```ocaml
let f : unit -> int = fun () -> 3 
let f_used : unit -> unit = fun x -> Js.log x  
```
```reason
let f: unit => int = () => 3;
let f_used: unit => unit = x => Js.log(x);
```

Output JS prior to v4.0.7

```js
function f (){
    return 3
}
function f_used (x){
    console.log(x)
}
```

To make this hack work, in the application side, 
for a curried function application, we treat the function 
of arity 0 and arity 1 in the same way, this still works since 
curried function application could only happen on the ocaml function.

This trick is unintuitive, it makes code generated less predictable and it is not relevant any more, since 
we added native uncurried calling convention support later.

Therefore, we generate JS code in a more consistent style in this release:

```ocaml
let f : unit -> int = fun () -> 3 
```
```reason
let f: unit => int = () => 3;
```

```js
function f (param){
    return 3
}
```

So in your FFI code, if you have a callback which is expected to be of arity zero, use `unit -> unit [@bs]` or `unit -> unit [@bs.uncurry]`, it is 100% correct. Note our previous trick will only make `unit -> unit` work most time, but it can not provide any guarantee.

Since we removed the trick, the curried runtime does not treat function of arity 0 and arity 1 in the same way, so if you have code like this

```ocaml
let f : unit -> int = [%bs.raw {|function () {
    return 3
}|}]
```
```reason
let f: unit => int = [%bs.raw {|function () {
  return 3
}|}];
```

It is not correct any more, the fix would be 


```ocaml
let f : unit -> int = [%bs.raw{|function(param){
    return 3
}|}]
```

```reason
let f: unit => int = [%bs.raw {|function(param) {
  return 3
}|}];
```

Or 


```ocaml
let f : unit -> int [@bs] = [%bs.raw{|function(){
    return 3
}|}]
```

```reason
let f: (. unit) => int = [%bs.raw {|function() {
  return 3
}|}];
```

FFI is a double edge sword, it is a must have to ship your product, yet it is tricky, and there may be some undefined behavior you rely on but don't recognize, it is encouarged to always test your FFI in the boundary.




