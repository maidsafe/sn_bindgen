# rusty-cheddar

A rustc compiler drop-in for automatically generating C header files from Rust
source files.

## Installation

rusty-cheddar is not yet on [crates.io](https://crates.io) but installation is simple with `cargo
install`:

    cargo install --git https://github.com/Sean1708/rusty-cheddar

or if you're a Homebrew user let me plug [cargo-brew](https://github.com/Sean1708/cargo-brew):

    cargo brew --git https://github.com/Sean1708/rusty-cheddar

Note that compiler drop-ins will only work on nightly releases of Rust. If you want to run multiple
versions of Rust you might consider [multirust](https://github.com/brson/multirust) or it's [pure
Rust equivalent](https://github.com/Diggsey/multirust-rs).

## Usage

Currently rusty-cheddar can only be run on a single rust file and does not yet support writing
output to a file. There are plans to make rusty-cheddar `cargo` aware and possibly even a full
`rustc` replacement, but there are ongoing issues with these. rusty-cheddar also targets C99 or
later (for sane single line comments and use of `stdint.h` and `stdbool.h`), if you really really
really really really have to use an older standard then please open an issue at the [repo] and I
will begrudgingly figure out how to implement support for it (after arguing with you lots and lots).

The usage is fairly simple, first you should write a rust file with a C callable API (rusty-cheddar
does not yet support any error checking or warnings but this is planned):

```rust
// File: ./capi.rs
#![allow(non_snake_case)]

/// Typedef docstrings!
pub type Kg = f32;

/// This is documentation!!!!
///
/// With two whole lines!!
#[repr(C)]
pub enum Eye {
    Blue = 1,
    /// Unfortunately variant docstrings are not indented properly yet.
    Green,
    Red,
}

/// Doccy doc for a struct-a-roony!!
#[repr(C)]
pub struct Person {
    age: i8,
    eyes: Eye,
    weight: Kg,
    height: f32,
}

/// Private functions are ignored.
#[no_mangle]
extern fn private_c(age: i8) {
    println!("Creating a {} year old!", age);
}

/// As are non-C functions
pub fn public_rust(weight_lbs: f32) {
    println!("Who weighs {} lbs.", weight_lbs);
}

#[no_mangle]
/// Function docs.
pub extern fn Person_create(age: i8, eyes: Eye, weight_lbs: f32, height_ins: f32) -> Person {
    private_c(age);
    public_rust(weight_lbs);
    Person {
        age: age,
        eyes: eyes,
        weight: weight_lbs * 0.45,
        height: height_ins * 0.0254,
    }
}

#[no_mangle]
pub extern fn Person_describe(person: Person) {
    let eyes = match person.eyes {
        Eye::Blue => "blue",
        Eye::Green => "green",
        Eye::Red => "red",
    };
    println!(
        "The {}m {} year old weighed {}kg and had {} eyes.",
        person.height, person.age, person.weight, eyes,
    );
}
```

Then just call `cheddar` on the file:

```
$ cheddar capi.rs
#ifndef cheddar_gen_cheddar_h
#define cheddar_gen_cheddar_h

#ifdef __cplusplus
extern "C" {
#endif

#include <stdint.h>
#include <stdbool.h>

/// Typedef docstrings!
typedef float Kg;

/// This is documentation!!!!
///
/// With two whole lines!!
typedef enum Eye {
	Blue = 1,
/// Unfortunately variant docstrings are not indented properly yet.
	Green,
	Red,
} Eye;

/// Doccy doc for a struct-a-roony!!
typedef struct Person {
	int8_t age;
	Eye eyes;
	Kg weight;
	float height;
} Person;

/// Function docs.
Person Person_create(int8_t age, Eye eyes, float weight_lbs, float height_ins);

void Person_describe(Person person);

#ifdef __cplusplus
}
#endif

#endif

$ cheddar capi.rs > capi.h
```

### Typedefs

rusty-cheddar converts `pub type A = B` into `typedef B A;`. Types containing generics are ignored.

### Enums

rusty-cheddar will convert public enums which are marked `#[repr(C)]`. If the enum is generic or
contains tuple or struct variants then `cheddar` will fail. rusty-cheddar should correctly handle
explicit discriminants.

### Structs

Structs are handled very similarly to enums, they must be marked `#[repr(C)]` and they must not
contain generics (this currently only checked at the struct-level, generic fields are not checked).

### Functions

For rusty-cheddar to pick up on a function declaration it must be public, marked `#[no_mangle]` and
have one of the following ABIs:

- C
- Cdecl
- Stdcall
- Fastcall
- System

I'm not totally up to speed on calling conventions so if you believe one of these has been including
in error, or if one has been omitted, then please open an issue at the [repo].

rusty-cheddar will fail on functions which are marked as diverging (`-> !`).


[repo]: https://github.com/Sean1708/rusty-cheddar
