---
title: Rust的pattern match 
date: 2020-01-13 23:53:30
tags: [rust]
categories: rust 
---

Rust中match随处可见，但是其中有一些细节值得注意：被match的对象可以是值也可以是引用，pattern可以是值也可以是引用，这就有4种组合，各自是什么行为呢？

<!-- more --> 

# 测试类 (1)

```rs
struct Foo<'a, 'b> {
    pub ival: i32,
    pub iref: &'a mut i32,
    pub sval: String,
    pub sref: &'b mut String,
}

impl<'a, 'b> Foo<'a, 'b> {
    fn new(ival:i32, iref: &'a mut i32, sval: String, sref: &'b mut String) -> Foo<'a, 'b> {
        Foo {ival, iref, sval, sref}
    }
    fn dump(&self) {
        println!("Foo: [{} {} {} {}]", self.ival, *self.iref, self.sval, *self.sref);
    }
}
```

# match对象和pattern都是值 (2)

```rs
fn match_val_by_val() {
    println!("--------match_val_by_val--------");
    let mut i: i32 = 1;
    let mut s: String = "a".to_string();

    let f: Foo = Foo::new(1, &mut i, "a".to_string(), &mut s);
    f.dump();

    match f {
        Foo {ival, iref, sval, sref} => {
            //ival(type i32)         : copied from f.ival;
            //iref(type &mut i32)    : moved from f.iref;
            //sval(type String)      : moved from f.sval;
            //sref(type &mut String) : moved from f.sref;

            *iref = *iref * 2;
            *sref = "b".to_string();
            println!("matched: [{} {} {} {}]", ival, *iref, sval, *sref);
        }
    }
    //f.dump();  //this would not compile, because f was moved;
}
```

运行结果是：

```
--------match_val_by_val--------
Foo: [1 1 a a]
matched: [1 2 a b]
```

# match对象和pattern都是ref (3)

```rs
fn match_ref_by_ref() {
    println!("--------match_ref_by_ref--------");
    let mut i: i32 = 8;
    let mut s: String = "hello".to_string();

    let f: Foo = Foo::new(8, &mut i, "hello".to_string(), &mut s);
    let r: &Foo = &f;
    r.dump();

    /*
    match r {
        &Foo {ival, iref, sval, sref} => {
            //ival: copied from r.ival;
            //iref: would be moved from r.iref; but r is a reference, move is not allowed;
            //sval: would be moved from r.sval; not allowed;
            //sref: would be moved from r.sref; not allowed;

            //this would be fine if all members of Foo are copiable. It can be thought this way:
            //rust tries to create a brand new Foo object from *r, this would fail if any memeber 
            //of *r is not copiable;

            println!("matched: [{} {} {} {}]", ival, *iref, sval, *sref);
        }
    }
    */

    match r {
        &Foo {ival, ref iref, ref sval, ref sref} => {
            //ival(type i32)          : copied from r.ival;
            //iref(type &&mut i32)    : borrowed from r.iref; 
            //sval(type &String)      : borrowed from r.sval; 
            //sref(type &&mut String) : borrowed from r.sref; 

            println!("matched: [{} {} {} {}]", ival, **iref, *sval, **sref);
        }
    }
}
```

运行结果是：

```
--------match_ref_by_ref--------
Foo: [8 8 hello hello]
matched: [8 8 hello hello]
```

# match对象是ref但pattern是值 (4)

```rs
fn match_ref_by_val() {
    println!("--------match_ref_by_val--------");
    let mut i: i32 = 8;
    let mut s: String = "hello".to_string();

    let f: Foo = Foo::new(8, &mut i, "hello".to_string(), &mut s);
    let r: &Foo = &f;
    r.dump();

    match r {
        Foo {ival, iref, sval, sref} => {
            //ival(type &i32)           : borrowed from r.ival;
            //iref(type &&mut i32)      : borrowed from r.iref;
            //sval(type &String)        : borrowed from r.sval;
            //sref(type &&mut String)   : borrowed from r.sref;

            println!("matched: [{} {} {} {}]", *ival, **iref, *sval, **sref);
        }
    }
    f.dump(); //f was not moved;
}
```

运行结果是：

```
--------match_ref_by_val--------
Foo: [8 8 hello hello]
matched: [8 8 hello hello]
Foo: [8 8 hello hello]
```

# match对象是值但pattern是ref (5)

```rs
fn match_val_by_ref() {
    println!("--------match_val_by_ref--------");
    let mut i: i32 = 8;
    let mut s: String = "hello".to_string();

    let f: Foo = Foo::new(8, &mut i, "hello".to_string(), &mut s);
    f.dump();

    match f {
        &Foo {ival, iref, sval, sref} => {}  //not allowed
    }
}
```

运行结果是：

```
error[E0308]: mismatched types
   --> src/main.rs:109:9
    |
109 |         &Foo {ival, iref, sval, sref} => {}  //not allowed
    |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `Foo`, found reference
    |
    = note: expected type `Foo<'_, '_>`
               found type `&_`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0308`.
error: could not compile `pattern`.

To learn more, run the command again with --verbose.
```
