- Feature Name: partial_borrow_syntax_sugar
- Start Date: 2019-01-16
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

By combining syntax sugar for anonymous struct arguments with syntax sugar for struct deconstruction+reconstruction, we can allow for partial borrows of structs without any changes to the type system or borrow checker.

# Motivation
[motivation]: #motivation

TODO: I think most people in the Rust internals board will have a good understanding of why partial borrows are useful and important for advancing the language. When this is ready for submitting as a formal RFC, I'll be sure to provide a complete and thorough motivation.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Below, we will use a series of simple examples to demonstrate the proposed syntax and behavior.

## Anonymous struct argument syntax sugar

```rust
fn add(v: {x: f32, y:f32}) -> f32 {
      v.x + v.y
}

let x = 1.; let y = 2.;
assert_eq!(add({x, y}), 3.);
```

## Partial borrow syntax sugar

```rust
fn add_y_to_x(v: {x: &mut f32, b: &f32}) {
    *v.x += *v.y;
}

struct V3 {
    x: f32,
    y: f32,
    z: f32,
}

let mut v3=V3 { x: 1., y: 2., z: 3. };
add_y_to_x(&mut v3);
assert_eq!(
    v3,
    V3 { x: 3., y: 2., z: 3. };
);
{
    // non-overlapping partial mutable borrows are allowed
    let z = &mut v3.z;
    add_y_to_x(&mut v3);
    assert_eq!(
        v3,
        V3 { x: 3., y: 2., z: 3. };
    );
}
{
    // overlapping partial mutable borrows are not allowed
    let x = &mut v3.x;
    add_y_to_x(&mut v3); // compile-fail: field `x` is already mutably borrowed
    *x += 1.;
}
```

## Partial borrow of self

```rust
impl V3 {
    fn xy_mut(self: {x: &mut f32, y: &mut f32}) -> (&mut f32, &mut f32) {
        (self.x, self.y)
    }

    fn z_mut(self: {z : &mut f32}) -> &mut f32 {
        self.z
    }
}

let v3=V3 { x: 1., y: 2., z: 3. };
assert_eq!(v3.xy_mut(), [&mut 1., &mut 2.]);
{
    // non-overlapping partial mutable borrows are allowed
    let (x, y) = v3.xy_mut();
    let z = v3.z_mut();
    assert_eq!(
        [x, y, z],
        [&mut 1., &mut 2., &mut 3.]
    );
}
{
    // overlapping partial mutable borrows are not allowed
    let x = &mut v3.x;
    let (x2, y) = v3.xy_mut(); // compile-fail: field `x` is already mutably borrowed
    *x += 1.;
}
```

## Partial borrow with explicit lifetimes

```rust
struct Foo<'a, 'b, 'c> {
    a: &'a mut f32,
    b: &'b mut f32,
    c: &'c mut f32,
}

fn add_b_to_a(foo: struct<'a, 'b> {a: &'a mut f32, b: &'b f32}) {
    *foo.a += foo.b;
}

let mut a = 1.;
{
    let mut b = 2.;
    let foo = Foo { &mut a, &mut b, &mut 3. };
    add_b_to_a(foo);
    assert_eq!(foo, Foo { &mut 3., &mut 2., &mut 3. });
    b += 1.;
}
a += 1.;
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Below, we show how the examples from the guide-level explanation can be desugared into currently-valid Rust syntax.

## Desugaring of `Anonymous struct argument syntax sugar`

```rust
struct __x_f32__y_f32 {
    x: f32,
    y: f32,
}

fn add(v: __x_f32__y_f32) -> f32 { ... }

let x = 1.; let y = 2.;
assert_eq!(add(__x_f32__y_f32 {x, y}), 3.);
```

## Desugaring of `Partial borrow syntax sugar`

```rust
struct __x_ref_mut_f32__y_ref_f32<'a> {
    x: &'a mut f32,
    y: &'a f32,
}
fn add_y_to_x<'a>(v: __x_ref_mut_f32__y_ref_f32<'a>) {
    *v.x += *v.y;
}

struct V3 {
    x: f32,
    y: f32,
    z: f32,
}

let mut v3=V3 { x: 1., y: 2., z: 3. };
add_y_to_x(__x_ref_mut_f32__y_ref_f32 {x: &mut v3.x, y: &v3.y});
assert_eq!(
    v3,
    V3 { x: 3., y: 2., z: 3. };
);
{
    // non-overlapping partial mutable borrows are allowed
    let z = &mut v3.z;
    add_y_to_x(__x_ref_mut_f32__y_ref_f32 {x: &mut v3.x, y: &v3.y});
    assert_eq!(
        v3,
        V3 { x: 3., y: 2., z: 3. };
    );
}
{
    // overlapping partial mutable borrows are not allowed
    let x = &mut v3.x;
    add_y_to_x(__x_ref_mut_f32__y_ref_f32 {
        x: &mut v3.x,
        y: &v3.y
    }); // compile-fail: field `x` is already mutably borrowed
    *x += 1.;
}
```

## Desugaring of `Partial borrow of self`

```rust
struct __x_ref_mut_f32__y_ref_mut_f32<'a> {
    x: &'a mut f32,
    y: &'a mut f32,
}
struct __z_ref_mut_f32<'a> {
    z: &'a mut f32,
}
impl V3 {
    fn xy_mut<'a>(self_: __x_ref_mut_f32__y_ref_mut_f32<'a>) -> (&'a mut f32, &'a mut f32) {
        (self_.x, self_.y)
    }

    fn z_mut(self_: __z_ref_mut_f32) -> &mut f32 {
        self_.z
    }
}

let v3=V3 { x: 1., y: 2., z: 3. };
assert_eq!(
    V3::xy_mut(__x_ref_mut_f32__y_ref_mut_f32 { x: &mut v3.x, y: &v3.y }),
    [&mut 1., &mut 2.]
);
{
    // non-overlapping partial mutable borrows are allowed
    let (x, y) = V3::xy_mut(
        __x_ref_mut_f32__y_ref_mut_f32 { x: &mut v3.x, y: &v3.y }
    );
    let z = V3::z_mut(__z_ref_mut_f32 { z: &mut v3.z });
    assert_eq!(
        [x, y, z],
        [&mut 1., &mut 2., &mut 3.]
    );
}
{
    // overlapping partial mutable borrows are not allowed
    let x = &mut v3.x;
    let (x2, y) = V3::xy_mut(
        __x_ref_mut_f32__y_ref_mut_f32 { x: &mut v3.x, y: &v3.y }
    ); // compile-fail: field `x` is already mutably borrowed
    *x += 1.;
}
```

## Desugaring of `Partial borrow with explicit lifetimes`

```rust
struct Foo<'a, 'b, 'c> {
    a: &'a mut f32,
    b: &'b mut f32,
    c: &'c mut f32,
}

struct<'a, 'b> __a_ref_a_mut_f32__b_ref_b_f32<'a, 'b> {
    a: &'a mut f32,
    b: &'b mut f32,
}
fn add_b_to_a<'a, 'b>(foo: __a_ref_a_mut_f32__b_ref_b_f32<'a, 'b>) {
    *foo.a += foo.b;
}

let mut a = 1.;
{
    let mut b = 2.;
    let foo = Foo { &mut a, &mut b, &mut 3. };
    add_b_to_a(__a_ref_a_mut_f32__b_ref_b_f32 {
        a: &mut foo.a,
        b: &foo.b,
    });
    assert_eq!(foo, Foo { &mut 3., &mut 2., &mut 3. });
    b += 1.;
}
a += 1.;
```

# Drawbacks
[drawbacks]: #drawbacks

TODO

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Note that this syntax-sugar approach to partial borrowing is fully backwards compatible with full anonymous structural typing, which could be added later to allow implementing traits for, and calling trait methods on, anonymous structs, and implicit partial borrowing when calling such trait methods.

TODO

# Prior art
[prior-art]: #prior-art

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO

# Future possibilities
[future-possibilities]: #future-possibilities

TODO
