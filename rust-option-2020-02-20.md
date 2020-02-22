---
title: Rust的Option细节
date: 2020-02-20 19:08:21
tags: [rust]
categories: rust 
---

Rust中Option使用细节。

<!-- more --> 

# 分类 

## unwrap系列

|函数             |功能                                   |move  |panic  |evaluate   |
|-----------------|---------------------------------------|------|-------|-----------|
|`expect`         |取出`v`或panic                         |Y     |Y      |N/A        |
|`unwrap`         |取出`v`或panic                         |Y     |Y      |N/A        |
|`unwrap_or`      |取出`v`或默认值                        |Y     |N      |eagerly    |
|`unwrap_or_else` |取出`v`或默认值                        |Y     |N      |lazily     |

## map系列

|函数             |功能                                          |move  |panic  |evaluate   |
|-----------------|----------------------------------------------|------|-------|-----------|
|`map`            |把本`Option<T>`转化成`Option<U>`或`None`      |Y     |       |N/A        |
|`map_or`         |把本`Option<T>`转化成`U`或默认值              |Y     |       |eagerly    |
|`map_or_else`    |把本`Option<T>`转化成`U`或默认值              |Y     |       |lazily     |

## ok系列

|函数             |功能                                          |move  |panic  |evaluate   |
|-----------------|----------------------------------------------|------|-------|-----------|
|`ok_or`          |把本`Option<T>`转化成`Result<T, E>`           |Y     |       |eagerly    |
|`ok_or_else`     |把本`Option<T>`转化成`Result<T, E>`           |Y     |       |lazily     |

## 逻辑运算系列

|函数             |功能                                                       |move  |panic  |evaluate   |
|-----------------|-----------------------------------------------------------|------|-------|-----------|
|`and`            |本`Option`和参数`optb`有一个为`None`结果就为`None`         |Y     |       |eagerly    |
|`and_then`       |本`Option`和参数`f()`有一个为`None`结果就为`None`          |Y     |       |lazily     |
|`or`             |本`Option`和参数`optb`有一个为`Some`结果就为`Some`         |Y     |       |eagerly    |
|`or_else`        |本`Option`和参数`f()`有一个为`Some`结果就为`Some`          |Y     |       |lazily     |
|`xor`            |本`Option`和参数`optb`有且只有一个为`Some`结果才为`Some`   |Y     |       |N/A        |

# 细节

## `expect()`

如果本`Option`是`Some`，就把值**move**出来；否则就以给定的`msg`来panic，和`unwrap`非常类似，只是panic产生的`msg`不同。

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

## `unwrap()`

如果本`Option`是`Some`，就把值**move**出来；否则就panic，和`expect`非常类似，只是panic产生的`msg`不同。

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

## `unwrap_or()`

如果本`Option`是`Some`，就把值**move**出来；否则返回`default`。注意：`default`会被**eagerly evaluated**，也就是说，无论本`Option`是`Some`还是`None`，`default`都先被求值。下面的`unwrap_or_else`与之相反。

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

## `unwrap_or_else()`

如果本`Option`是`Some`，就把值**move**出来；否则返回`f()`。注意：`f()`会被**lazily evaluated**，也就是说，当且仅当本`Option`是一个`None`时`f()`才被求值。上面的`unwrap_or`与之相反。

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

## `map()`

通过一个函数`f(T)->U`把本`Option<T>`转化成`Option<U>`（**而不是`U`**）；若本`Option`为`None`，则返回`None`而不panic；

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

## `map_or()`

通过一个函数`f(T)->U`把本`Option<T>`转化成`U`（**而不是`Option<U>`**）；若本`Option`为`None`，则返回`default`。注意：`default`会被**eagerly evaluated**，也就是说，无论是本`Option`是`Some`还是`None`，`default`都先被求值。下面的`map_or_else`与之相反。

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

## `map_or_else()`

通过一个函数`f(T)->U`把本`Option<T>`转化成`U`（**而不是`Option<U>`**）；若本`Option`为`None`，则返回`default`。注意：`default`会被**lazily evaluated**，也就是说，当且仅当本`Option`是一个`None`时`default`才被求值。上面的`map_or`与之相反。

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

## `ok_or()`

把本`Option<T>`转化成`Result<T, E>`：若为`Some(v)`则结果为`Ok(v)`；若为`None`则结果为`Err(err)`。注意：`err`会被**eagerly evaluated**，也就是说，无论本`Option`是`Some`还是`None`，`err`都先被求值。下面的`ok_or_else`与之相反。

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

## `ok_or_else()`

把本`Option<T>`转化成`Result<T, E>`：若为`Some(v)`则结果为`Ok(v)`；若为`None`则结果为`Err(err)`。注意：`err`会被**lazily evaluated**，也就是说，当且仅当本`Option`是一个`None`时`err`才被求值。上面的`ok_or`与之相反。

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

## `and()`

若本`Option`是`Some`则返回参数给定的`optb`；否则，本`Option`为`None`，返回`None`。注意：`optb`会被**eagerly evaluated**，也就是说，无论本`Option`是`Some`还是`None`，`optb`都先被求值。下面的`and_then`与之相反。

可以理解成：`*this && optb`，二者若有一个为`None`则结果就是`None`，且逻辑表达式求值时**不短路**，即无论`*this`的结果是`true`还是`false`都要对`optb`求值。

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

## `and_then()`

若本`Option`是`Some(v)`则返回参数给定的`f(v)`；否则，本`Option`为`None`，返回`None`。注意：`f()`会被**lazily evaluated**，也就是说，当且仅当本`Option`是一个`Some`时`f()`才被求值。上面的`and`与之相反。

可以理解成：`*this && f()`，二者若有一个为`None`则结果就是`None`，且逻辑表达式求值时**会短路**，即若`*this`为`false`，则不调用`f()`，直接返回；

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

## `or()`

若本`Option`是`Some`则返回本`Option`；否则，返回参数给定的`optb`。注意：`optb`会被**eagerly evaluated**，也就是说，无论本`Option`是`Some`还是`None`，`optb`都先被求值。下面的`or_else`与之相反。

可以理解成：`*this || optb`，二者若有一个为`Some`则结果就是`Some`，且逻辑表达式求值时**不短路**，即无论`*this`的结果是`true`还是`false`都要对`optb`求值。

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

## `or_else()`

若本`Option`是`Some`则返回本`Option`；否则，返回参数给定的`f()`。注意：`f()`会被**lazily evaluated**，也就是说，当且仅当本`Option`是一个`None`时`f()`才被求值。上面的`or`与之相反。

可以理解成：`*this || f()`，二者若有一个为`Some`则结果就是`Some`，且逻辑表达式求值时**会短路**，即若`*this`为`true`，则不调用`f()`，直接返回；

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

## `xor()`

当且仅当本`Option`和参数`optb`二者之中有一个`Some`，才返回这个`Some`；否则返回`None`。注意，`optb`一定会被求值，因为只有求值才知道它是`Some`还是`None`。

可以理解成：`*this ^ optb`。

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
