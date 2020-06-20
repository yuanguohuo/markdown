---
title: Rust的pattern match 
date: 2020-01-13 23:53:30
tags: [rust]
categories: rust 
---

Rust中match随处可见，但是其中有一些细节值得注意：被match的对象可以是值也可以是引用，pattern可以是值也可以是引用，这就有4种组合，各自是什么行为呢？

<!-- more --> 

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {inlineMath: [['$','$'], ['\\(','\\)']]}
});
</script>

<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>

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

pub struct Bar<T> {
    pub val: T,
}

impl<T> Bar<T> {
    pub fn new(val: T) -> Self {
        Bar { val }
    }
}

impl<T: fmt::Display> fmt::Display for Bar<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Bar:[val: {}]", self.val)
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

        let b = Bar::new(i);
        match b {
            Bar { val } => println!("matched Bar({})", val),
        }
        println!("{}", b);

        let b1 = Bar::new(String::from("hello"));
        match b1 {
            Bar { val } => println!("matched Bar({})", val),
        }
        //println!("{}", b1); //value borrowed here after partial move
    }
```

运行结果：

```
--------match_val_by_val--------
Foo:[ival(i32): 1, iref(&mut i32): 1, sval(String): a, sref(&mut String): a]
matched: [1 2 a b]
matched Bar(2)
Bar:[val: 2]
matched Bar(hello)
ok
```

这种match，相当于每个成员被复制（move或copy），但整体object没有被复制（move或copy），也就是说，`match obj`的时候：

- `obj`不会被复制（move或copy）；
- `obj`里的每个成员被复制（move或copy）；

之前我一直认为是在pattern那里构造一个新对象，`obj`被复制（move或copy）到那里。这是不对的。看上面例子中的`b`：假如像我之前认为的那样，`b`在`match`的时候就被复制（就是move，因为`Bar`不是Copy），之后打印它就会编译失败。所以，强调一下：`match obj`的时候，`obj`这个对象本身（作为一个整体）没有被复制（move或copy）：

- 若`obj`里的每个成员都是Copy类型，那么`match obj`之后，`obj`是完好无损的（`obj`本身没被move或copy，成员被copy了而已），例如`b`；
- 若`obj`里存在非Copy类型的成员，那么`match obj`之后，`obj`就被`partial move`了（partial就是指那些不是Copy类型的成员）例如`f`和`b1`。可以使用`ref`关键字来引用非Copy成员，而避免复制（move）；

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

        let b = Bar::new(i);
        let rb = &b;
        match rb {
            &Bar { val } => println!("matched Bar({})", val),
        }
        println!("{}", b);
    }
```

运行结果：

```
--------match_ref_by_ref--------
Foo:[ival(i32): 8, iref(&mut i32): 8, sval(String): hello, sref(&mut String): hello]
matched: [8 8 hello hello]
Foo:[ival(i32): 8, iref(&mut i32): 8, sval(String): hello, sref(&mut String): hello]
Foo:[ival(i32): 8, iref(&mut i32): 8, sval(String): hello, sref(&mut String): hello]
matched Bar(8)
Bar:[val: 8]
ok
```

和`match_val_by_val`一致，只是不会发生`partial move`而是会编译失败（因为引用不能被move）：

- 若`obj`里的每个成员都是Copy类型，那么`match &obj`之后，`obj`是完好无损的（`obj`本身没被move或copy——显而易见，我们match的是引用，都没有ownership谈何move——成员被copy了而已），例如`b`；
- 若`obj`里存在非Copy类型的成员，那么`match &obj`会编译失败（需要move，但我们match的是引用，没有ownership），例如注释掉的个`match`语句。可以使用`ref`关键字来引用非Copy成员，而避免复制（move）；

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
ok
```

这种最简单：

- `match &obj`不可能使obj被move（显而易见，我们match的是引用，没有ownership）；
- pattern中的每个字段都是`obj`中对应字段的引用；

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
   --> src/test_match/test_struct_match.rs:179:13
    |
177 |           match f {
    |                 - this expression has type `test_match::test_struct_match::Foo<'_, '_>`
178 |               //error here: expected struct `Foo`, found reference
179 | /             &Foo {
180 | |                 ival,
181 | |                 iref,
182 | |                 sval,
183 | |                 sref,
184 | |             } => {} //not allowed
    | |_____________^ expected struct `test_match::test_struct_match::Foo`, found reference
    |
    = note: expected struct `test_match::test_struct_match::Foo<'_, '_>`
            found reference `&_`

error: aborting due to previous error
```

# 小结 (5)

对于<u>(1)match对象和pattern都是值</u>和<u>(2)match对象和pattern都是ref</u>这两种情况来说：Rust**试图**对obj的成员一一复制（move或copy），**仅此而已，不复制obj这个变量本身**。对于<u>(1)</u>可能导致对象被`partial move`；对于<u>(2)</u>可能导致编译错误。两者都是由obj中存在非Copy成员，对它们的复制（即move）导致的。且两者都可以使用`ref`关键字来引用非Copy成员，而避免复制（move）它们。
所以，<u>(3)match对象是ref但pattern是值</u>有点像是一个优化：所有成员，无论是不是Copy，都不复制（move或copy），一律引用。
个人理解是，*(2)*应该少用，根据程序的上下文：若match以后永远不再需要访问obj就使用<u>(1)</u>；否则，还需要保留obj就使用<u>(3)</u>。
