---
title: Rust的pattern match 
date: 2020-01-13 23:53:30
tags: [rust]
categories: rust 
---

Rust中match随处可见，但是其中有一些细节值得注意：被match的对象可以是值也可以是引用，pattern可以是值也可以是引用，这就有4种组合，各自是什么行为呢？

<!-- more --> 

# 测试类

```rs
use std::fmt;

pub struct Foo<'a, 'b> {
    pub ival: i32,
    pub iref: &'a mut i32,
    pub sval: String,
    pub sref: &'b mut String,
}

impl<'a, 'b> Foo<'a, 'b> {
    pub fn new(ival: i32, iref: &'a mut i32, sval: String, sref: &'b mut String) -> Foo<'a, 'b> {
        Foo {
            ival,
            iref,
            sval,
            sref,
        }
    }
}

impl<'a, 'b> fmt::Display for Foo<'a, 'b> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(
            f,
            "Foo:[ival(i32): {}, iref(&mut i32): {}, sval(String): {}, sref(&mut String): {}]",
            self.ival, self.iref, self.sval, self.sref
        )
    }
}
```

# match对象和pattern都是值 (1)

```rs
    #[test]
    fn match_val_by_val() {
        println!("--------match_val_by_val--------");
        let mut i: i32 = 1;
        let mut s: String = "a".to_string();

        let f: Foo = Foo::new(1, &mut i, "a".to_string(), &mut s);
        println!("{}", f);

        match f {
            Foo {
                ival,
                iref,
                sval,
                sref,
            } => {
                //ival(type i32)         : copied from f.ival;
                //iref(type &mut i32)    : moved from f.iref; 注意: &mut和Option(&mut)不是Copy类型；&和Option(&)是Copy类型；
                //sval(type String)      : moved from f.sval;
                //sref(type &mut String) : moved from f.sref;

                *iref = *iref * 2;
                *sref = "b".to_string();
                println!("matched: [{} {} {} {}]", ival, *iref, sval, *sref);
            }
        }
        //println!("{}", f); //this would not compile, because f was moved;
    }
```

运行结果：

```
--------match_val_by_val--------
Foo:[ival(i32): 1, iref(&mut i32): 1, sval(String): a, sref(&mut String): a]
matched: [1 2 a b]
```

# match对象和pattern都是ref (2)

```rs
    #[test]
    fn match_ref_by_ref() {
        println!("--------match_ref_by_ref--------");
        let mut i: i32 = 8;
        let mut s: String = "hello".to_string();

        let f: Foo = Foo::new(8, &mut i, "hello".to_string(), &mut s);
        let r: &Foo = &f;
        println!("{}", r);

        /*
        match r {
            &Foo {ival, iref, sval, sref} => {
                //ival: copied from r.ival;
                //iref: would be moved from r.iref; but r is a mut reference, move is not allowed; 注意: &mut和Option(&mut)不是Copy类型；&和Option(&)是Copy类型；
                //sval: would be moved from r.sval; not allowed;
                //sref: would be moved from r.sref; not allowed;

                //this would be fine if all members of Foo are Copy. It can be thought this way:
                //rust would create a brand new Foo from r, as a result, all members would be
                //moved (unless it is Copy) from r; but on the other hand r is a reference
                //which cannot be moved. So, this will not compile if any member of r is not Copy;

                println!("matched: [{} {} {} {}]", ival, *iref, sval, *sref);
            }
        }
        */

        match r {
            &Foo {
                ival,
                ref iref,
                ref sval,
                ref sref,
            } => {
                //ival(type i32)          : copied from r.ival;
                //iref(type &&mut i32)    : borrowed from r.iref;
                //sval(type &String)      : borrowed from r.sval;
                //sref(type &&mut String) : borrowed from r.sref;

                println!("matched: [{} {} {} {}]", ival, **iref, *sval, **sref);
            }
        }
        println!("{}", f);
        println!("{}", r);
    }
```

运行结果：

```
--------match_ref_by_ref--------
Foo:[ival(i32): 8, iref(&mut i32): 8, sval(String): hello, sref(&mut String): hello]
matched: [8 8 hello hello]
Foo:[ival(i32): 8, iref(&mut i32): 8, sval(String): hello, sref(&mut String): hello]
Foo:[ival(i32): 8, iref(&mut i32): 8, sval(String): hello, sref(&mut String): hello]
```

# match对象是ref但pattern是值 (3)

```rs
    #[test]
    fn match_ref_by_val() {
        println!("--------match_ref_by_val--------");
        let mut i: i32 = 8;
        let mut s: String = "hello".to_string();

        let f: Foo = Foo::new(8, &mut i, "hello".to_string(), &mut s);
        let r: &Foo = &f;
        println!("{}", r);

        match r {
            Foo {
                ival,
                iref,
                sval,
                sref,
            } => {
                //ival(type &i32)           : borrowed from r.ival;
                //iref(type &&mut i32)      : borrowed from r.iref;
                //sval(type &String)        : borrowed from r.sval;
                //sref(type &&mut String)   : borrowed from r.sref;

                println!("matched: [{} {} {} {}]", *ival, **iref, *sval, **sref);
            }
        }
        println!("{}", f); //f was not moved;
    }
```

运行结果：

```
--------match_ref_by_val--------
Foo:[ival(i32): 8, iref(&mut i32): 8, sval(String): hello, sref(&mut String): hello]
matched: [8 8 hello hello]
Foo:[ival(i32): 8, iref(&mut i32): 8, sval(String): hello, sref(&mut String): hello]
```

# match对象是值但pattern是ref (4)

```rs
    #[test]
    fn match_val_by_ref() {
        println!("--------match_val_by_ref--------");
        let mut i: i32 = 8;
        let mut s: String = "hello".to_string();

        let f: Foo = Foo::new(8, &mut i, "hello".to_string(), &mut s);
        println!("{}", f);

        match f {
            //error here: expected struct `Foo`, found reference
            &Foo {
                ival,
                iref,
                sval,
                sref,
            } => {} //not allowed
        }
    }
```

运行结果：

```
error[E0308]: mismatched types
   --> src/test_match/test_struct_match.rs:149:13
    |
147 |           match f {
    |                 - this expression has type `test_match::test_struct_match::Foo<'_, '_>`
148 |               //error here: expected struct `Foo`, found reference
149 | /             &Foo {
150 | |                 ival,
151 | |                 iref,
152 | |                 sval,
153 | |                 sref,
154 | |             } => {} //not allowed
    | |_____________^ expected struct `test_match::test_struct_match::Foo`, found reference
    |
    = note: expected struct `test_match::test_struct_match::Foo<'_, '_>`
            found reference `&_`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0308`.
```
