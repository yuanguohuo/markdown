---
title: rust的Option细节
date: 2020-02-20 19:08:21
tags: [rust]
categories: rust 
---

Rust中Option使用细节。

<!-- more --> 

# `unwrap()`

```rust
let o: Option<String> = Some(String::from("unwrap() moves the Option's value"));
let s = o.unwrap(); //moved
println!("{}", s);
//println!("{:?}", o);   //value borrowed here after move
```

输出：
```
unwrap() moves the Option's value
```

# `expect()`

```rust
let o: Option<String> = Some(String::from("expect() moves the Option's value"));
let s = o.expect("oh no!"); //moved
println!("{}", s);
//println!("{:?}", o);   //value borrowed here after move
```

输出：
```
expect() moves the Option's value
```


# `unwrap_or()`

```rust
fn get_string() -> String {
    println!("get_string() is called");
    String::from("world")
}

let o: Option<String> = Some(String::from(
    "unwrap_or() moves the Option's value, and the default expression is eagerly evaluated",
));
let s = o.unwrap_or(get_string()); //moved, eagerly evaluated
println!("{}", s);
//println!("{:?}", o);   //value borrowed here after move
```

输出：
```
get_string() is called
unwrap_or() moves the Option's value, and the default expression is eagerly evaluated
```

# `unwrap_or_else()`

```rust
let o: Option<String> = Some(String::from(
    "unwrap_or_else() moves the Option's value, and the default expression is lazily evaluated",
));
let s = o.unwrap_or_else(|| {   //moved, lazily evaluated
    panic!("closure called");
    String::from("world")
});
println!("{}", s);
//println!("{:?}", o);   //value borrowed here after move
```

输出：
```
unwrap_or_else() moves the Option's value, and the default expression is lazily evaluated
```

# `map()`

```rust
let o: Option<String> = Some(String::from("map() moves the Option's value"));
let s: Option<usize> = o.map(|str| { //moved
    println!("{}", str);
    str.len()
});
println!("{:?}", s);
//println!("{:?}", o);   //value borrowed here after move
```

输出：
```
map() moves the Option's value
Some(30)
```

# `map_or()`

```rust
fn get_usize() -> usize {
    println!("get_usize() is called");
    123
}

let o: Option<String> = Some(String::from(
    "map_or() moves the Option's value, and the default expression is eagerly evaluated",
));
let s: usize = o.map_or(get_usize(), |str| {  //moved
    println!("{}", str);
    str.len()
});
println!("{:?}", s);
//println!("{:?}", o);   //value borrowed here after move
```

输出：
```
get_usize() is called
map_or() moves the Option's value, and the default expression is eagerly evaluated
82
```

# `map_or_else()`

```rust
let o: Option<String> = Some(String::from(
    "map_or_else() moves the Option's value, and the default expression is lazily evaluated",
));
let s: usize = o.map_or_else(
    || {
        panic!("closure called");
        321
    },
    |str| {
        println!("{}", str);
        str.len()
    },
);
println!("{:?}", s);
//println!("{:?}", o);   //value borrowed here after move
```

输出：
```
map_or_else() moves the Option's value, and the default expression is lazily evaluated
86
```
