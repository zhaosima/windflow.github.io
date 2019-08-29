---
title: explain one Rust iterator example
date: 2019-08-29 15:08:42
tags:
---
RUST教程中迭代器示例代码解析

## [示例](https://doc.rust-lang.org/stable/rust-by-example/error/iter_result.html#iterating-over-results)

```Rust
fn main() {
    let strings = vec!["tofu", "93", "18"];
    let possible_numbers: Vec<_> = strings
        .into_iter()
        .map(|s| s.parse::<i32>())
        .collect();
    println!("Results: {:?}", possible_numbers);
}
```
```
Results: [Err(ParseIntError { kind: InvalidDigit }), Ok(93), Ok(18)]
```

### Explain Step by Step
将这段代码拆解如下：

1. 定义
```Rust
let strings = vec!["tofu", "93", "18"]; 
```

2. 获取迭代器
```Rust
let iter = strings.into_iter();
```
此时 iter 的类型是 vec::IntoIter<&str>

3. map
```Rust
let map = iter.map(|s| s.parse::<i32>());
```
此时 map 的类型是 Map<vec::IntoIter<&str>, fn (&str) -> Result >

4. get numbers
```Rust
let numbers : Vec<_>= map.collect();
```
collect 是 Iterator trait 自带的默认函数，而 Map<,> 实现了 Iterator。
```Rust
fn collect<B: FromIterator<Self::Item>>(self) -> B where Self: Sized {
        FromIterator::from_iter(self)
}
```
代入上面的代码可得
```Rust
fn collect(self_map : Iterator) -> Vec<Result> {
        Vec<Result>::from_iter(self_map)
}
```
这里就显现出 Rust 编译器的厉害之处了，由于我们指定了 B 的类型 (查看 numbers 的定义)，并且 Vec 实现了 FromIterator trait, 所以这里的 FromIterator::from_iter 会被替换为 Vec<Result>::from_iter。

5. Vec<T>::from_iter

```Rust
impl<T> FromIterator<T> for Vec<T> {
    #[inline]
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Vec<T> {
        <Self as SpecExtend<T, I::IntoIter>>::from_iter(iter.into_iter())
    }
}
```

此处需要注意的一点, 每个实现了 Iterator 的类型都默认实现了 IntoIterator trait, into_iter 返回是 self， 所以还是 map。

```Rust
impl<T, I> SpecExtend<T, I> for Vec<T>
    where I: Iterator<Item=T>,
{
    default fn from_iter(mut iterator: I) -> Self {
        let mut vector = match iterator.next() {
            None => return Vec::new(),
            Some(element) => {
                let (lower, _) = iterator.size_hint();
                let mut vector = Vec::with_capacity(lower.saturating_add(1));
                unsafe {
                    ptr::write(vector.get_unchecked_mut(0), element);
                    vector.set_len(1);
                }
                vector
            }
        };
        <Vec<T> as SpecExtend<T, I>>::spec_extend(&mut vector, iterator);
        vector
    }

    default fn spec_extend(&mut self, iter: I) {
        self.extend_desugared(iter)
    }
}

impl<T> Vec<T> {
    fn extend_desugared<I: Iterator<Item = T>>(&mut self, mut iterator: I) {
        while let Some(element) = iterator.next() {
            let len = self.len();
            if len == self.capacity() {
                let (lower, _) = iterator.size_hint();
                self.reserve(lower.saturating_add(1));
            }
            unsafe {
                ptr::write(self.get_unchecked_mut(len), element);
                self.set_len(len + 1);
            }
        }
    }
    ...
}
```
只要从 iterator 中取出的元素值不是 None, 就不断地将值插入新建的 Vec 中。

6. Map 是如何实现 Iterator trait
```Rust
pub struct Map<I, F> {
    iter: I,
    f: F,
}

impl<B, I: Iterator, F> Iterator for Map<I, F> where F: FnMut(I::Item) -> B {
    type Item = B;

    #[inline]
    fn next(&mut self) -> Option<B> {
        self.iter.next().map(&mut self.f)
    }
    ...
}

// impl for Option
pub fn map<U, F: FnOnce(T) -> U>(self, f: F) -> Option<U> {
        match self {
            Some(x) => Some(f(x)),
            None => None,
        }
    }
```
next() 拿到内部迭代器的 next value，并且通过 Option::map 转成另一个 Option<B\> (B 就是 Result), 内部迭代器就是 vec::IntoIter。

7. vec::IntoIter
```Rust
pub struct IntoIter<T> {
    buf: NonNull<T>,
    phantom: PhantomData<T>,
    cap: usize,
    ptr: *const T,
    end: *const T,
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    #[inline]
    fn next(&mut self) -> Option<T> {
        unsafe {
            if self.ptr as *const _ == self.end {
                None
            } else {
                if mem::size_of::<T>() == 0 {
                    // purposefully don't use 'ptr.offset' because for
                    // vectors with 0-size elements this would return the
                    // same pointer.
                    self.ptr = arith_offset(self.ptr as *const i8, 1) as *mut T;

                    // Make up a value of this ZST.
                    Some(mem::zeroed())
                } else {
                    let old = self.ptr;
                    self.ptr = self.ptr.offset(1);

                    Some(ptr::read(old))
                }
            }
        }
    }
    ...
}
```
通过指针移动读取每个 index 的值

8. 示例完结

可以看出在最终调用 collect() 之前，没有任何的元素读取，每一步操作返回的是有操作语义的 “取值子”。在 collect 之时，迭代器配合 FromIterator, 层层深入，最终取得了原始值，并通过 map 之后层层析出而后插入结果。


## [更有趣的例子](https://doc.rust-lang.org/stable/rust-by-example/error/iter_result.html#fail-the-entire-operation-with-collect)

```Rust
fn main() {
    let strings = vec!["tofu", "93", "18"];
    let numbers: Result<Vec<_>, _> = strings
        .into_iter()
        .map(|s| s.parse::<i32>())
        .collect();
    println!("Results: {:?}", numbers);
}
```
```
Results: Err(ParseIntError { kind: InvalidDigit })
```
这里和前面的例子几乎一摸一样，不同的只是返回值的类型声明变了，但得到的结果完全不同。
有了前面的分析，可以很容易想到为了能否符合 collect() 函数的约束， Result 必然实现了 FromIterator。

```Rust
fn from_iter<I: IntoIterator<Item=Result<A, E>>>(iter: I) -> Result<V, E> {
        struct Adapter<Iter, E> {
            iter: Iter,
            err: Option<E>,
        }

        impl<T, E, Iter: Iterator<Item=Result<T, E>>> Iterator for Adapter<Iter, E> {
            type Item = T;

            #[inline]
            fn next(&mut self) -> Option<T> {
                match self.iter.next() {
                    Some(Ok(value)) => Some(value),
                    Some(Err(err)) => {
                        self.err = Some(err);
                        None
                    }
                    None => None,
                }
            }

            fn size_hint(&self) -> (usize, Option<usize>) {
                let (_min, max) = self.iter.size_hint();
                (0, max)
            }
        }

        let mut adapter = Adapter { iter: iter.into_iter(), err: None };
        let v: V = FromIterator::from_iter(adapter.by_ref());

        match adapter.err {
            Some(err) => Err(err),
            None => Ok(v),
        }
    }
```
let v: V = FromIterator::from_iter(adapter.by_ref()); 会被替换成 Vec::from_iter。
根据前面我们的分析，会进入 Adapter 的 next(), 当 iter 取值正常的时候插入 Vec, 当有错时记录错误并返回 None。 返回 None 则意味着 Vec::from_iter 的终止，此时虽然 v:Vec 记录了值，但由于错误的存在， Result 抛弃了 v 而返回了 Err。


### 编后语
阅读网页版的 Rust 源码不是种很好的体验，如果有 IDE 的帮助会省事不少。最大的收获是理解对 trait 具体实现的调用。

另外有个小窍门，如何查看某个变量 x 的类型，只需要给 x 调用一个不存在的方法就可以了。例如：
x.do(); 编译器会报错并给出变量的类型。
