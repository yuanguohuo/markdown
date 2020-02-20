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

# `ok_or()`

```rust
fn get_usize() -> usize {
    println!("get_usize() is called");
    123
}

let o: Option<String> = Some(String::from("ok_or() moves the Option's value, and the error expression is eagerly evaluated"));
let r: Result<String, usize> = o.ok_or(get_usize());
println!("{:?}", r);
//println!("{:?}", o); //value borrowed here after move
```

输出：
```
get_usize() is called
Ok("ok_or() moves the Option\'s value, and the error expression is eagerly evaluated")
```

# `ok_or_else()`

```rust
let o: Option<String> = Some(String::from("ok_or_else() moves the Option's value, and the error expression is lazily evaluated"));
let r: Result<String, usize> = o.ok_or_else(||{
    panic!("closure called");
    123
});
println!("{:?}", r);
//println!("{:?}", o); //value borrowed here after move
```

输出：
```
Ok("ok_or_else() moves the Option\'s value, and the error expression is lazily evaluated")
```

# `and()`

```rust
fn get_option() -> Option<usize> {
    println!("get_option() is called");
    Some(123)
}

{
    let o: Option<String> = Some(String::from("and() moves the Option's value, and the expression is eagerly evaluated"));
    let u: Option<usize> = o.and(get_option());
    println!("{:?}", u);
    //println!("{:?}", o); //value borrowed here after move
}

{
    let o: Option<String> = None;
    let u: Option<usize> = o.and(get_option());
    println!("{:?}", u);
    //println!("{:?}", o); //value borrowed here after move
}
```

输出：
```
get_option() is called
Some(123)
get_option() is called
None
```

# `and_then()`

```rust
{
    let o: Option<String> = Some(String::from("and_then() moves the Option's value"));
    let u: Option<usize> = o.and_then(|str|{
        Some(str.len())
    });
    println!("{:?}", u);
    //println!("{:?}", o); //value borrowed here after move
}

{
    let o: Option<String> = None;
    let u: Option<usize> = o.and_then(|str|{
        panic!("closure called");
        Some(str.len())
    });
    println!("{:?}", u);
    //println!("{:?}", o); //value borrowed here after move
}
```

输出：
```
Some(35)
None
```

# `or()`

```rust
fn get_str_option() -> Option<String> {
    println!("get_str_option() is called");
    Some(String::from("world"))
}

{
    let o: Option<String> = Some(String::from("or() moves the Option's value, and the expression is eagerly evaluated"));
    let u: Option<String> = o.or(get_str_option());
    println!("{:?}", u);
    //println!("{:?}", o); //value borrowed here after move
}

{
    let o: Option<String> = None;
    let u: Option<String> = o.or(get_str_option());
    println!("{:?}", u);
    //println!("{:?}", o); //value borrowed here after move
}
```

输出：
```
get_str_option() is called
Some("or() moves the Option\'s value, and the expression is eagerly evaluated")
get_str_option() is called
Some("world")
```


# `or_else()`

```rust
{
    let o: Option<String> = Some(String::from("or_else() moves the Option's value, and the expression is lazily evaluated"));
    let u: Option<String> = o.or_else(||{
        panic!("closure is called");
        Some(String::from("world"))
    });
    println!("{:?}", u);
    //println!("{:?}", o); //value borrowed here after move
}

{
    let o: Option<String> = None;
    let u: Option<String> = o.or_else(||{
        Some(String::from("world"))
    });
    println!("{:?}", u);
    //println!("{:?}", o); //value borrowed here after move
}
```

输出：
```
Some("or_else() moves the Option\'s value, and the expression is lazily evaluated")
Some("world")
```

# `xor()`

```rust
fn get_str_option() -> Option<String> {
    println!("get_str_option() is called");
    Some(String::from("world"))
}

{
    let o: Option<String> = Some(String::from("xor() moves the Option's value"));
    let u: Option<String> = o.xor(get_str_option());
    println!("{:?}", u);
    //println!("{:?}", o); //value borrowed here after move
}

{
    let o: Option<String> = None;
    let u: Option<String> = o.xor(get_str_option());
    println!("{:?}", u);
    //println!("{:?}", o); //value borrowed here after move
}
```

输出：
```
get_str_option() is called
None
get_str_option() is called
Some("world")
```
