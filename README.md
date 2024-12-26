# shallowclone

A Rust library providing traits for working with copy-on-write values efficiently.

### `ShallowClone` trait

This is basically the same as the standard [`Clone`](https://doc.rust-lang.org/std/clone/trait.Clone.html),
except that it's optimized for copy-on-write values so that they're not cloned. Shallow cloning a
[`Cow`](https://doc.rust-lang.org/std/borrow/enum.Cow.html) will always produce a `Borrowed` variant, either referencing the original
value if it was `Owned`, or just copying the reference if it already was `Borrowed`.

original | after `ShallowClone::shallow_clone(&'a T)`
--- | ---
`Cow::Owned(T)` | `Cow::Borrowed(&'a T)`
`Cow::Borrowed(&'b T)` | `Cow::Borrowed(&'a T)`

### `MakeOwned` trait

This is kind of a side effect of `ShallowClone`. It allows to convert any value that implements it
to an equivalent type which is `'static` - no references, completely self-sufficient. It does that by replacing
all inner `Cow`s with their owned variants, which allows to set the associated lifetimes to `'static`.

Obviously you can't use this if your type contains straight up references not in an enum like `Cow`.

### Variance stuff

You might notice, that if you have some deeply nested structures with `Cow`s, where your inner value contains
references too, for example:

```rust
struct Foo<'a> {
	inner: Cow<'a, [Bar<'a>]>,
}
struct Bar<'a> {
	id: &'a str,
	data: [u8; 1024],
}
```

Here we have a `Cow` of a slice of `Bar`s, and `Bar` also has a lifetime. If you try to shallow clone
`Foo`, it will not compile, because the shallow cloning will try to shorten the lifetime of the `Cow`, but not the
lifetime of `Bar` (shallow cloning only goes as deep as the first `Cow`, upon which it just replaces it with
a `Borrowed` variant). So you will end up with `Cow<'a, [Bar<'b>]>` and even though `'b` is clearly a subtype of `'a`
and you could expect the compiler to accept it, actually `Cow<'_, T>` is **invariant**[^1] over `T` so it won't compile.

One solution to this would be to have two lifetimes instead, like `Foo<'a, 'b>` but that quickly bubbles up in your
structures and makes everything much more complicated.

To solve this issue, this crate also introduces two **covariant** replacements for the standard `Cow`:
 - `CoCow<'a, T>` which is a general replacement for the standard `Cow`,
 - `CoCowSlice<'a, T>` which is a replacement for `Cow<'a, [T]>`.

These types don't rely on the `ToOwned` trait and have simpler, but less powerful definition (hence the need for
a an additional `CoCowSlice`). They are covariant over `T`, which allows `ShallowClone` to be used with nested
structures like described.

You can also easily make your own specialised `Cow` types and implement `ShallowClone` for them, if these two types
are not sufficient for your needs.

**Note** that for simple cases like `Cow<'a, str>` or `Cow<'a, [u8]>` there is no need for them and you can just use
the normal `std::borrow::Cow`.

[^1]: This is because `Cow<T>` requires that `T: ToOwned`, therefore if you subtyped `T` it could have a different
`ToOwned` implementation (or even none) and it would invalidate the type. The `CoCow` types in this crate do not
rely on any traits and avoid this limitation.
