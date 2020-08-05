---
layout: post
title: Building a Brainf*ck Compiler with Rust and LLVM
catagories: Rust, LLVM, Programming Languages, Tutorial
---

[LLVM](https://llvm.org/) is a collection of compiler toolchain technologies  used to compile dynamic and static programming languages. It's heavly used in programming languages, such as [Clang](https://clang.llvm.org/), [Emscripten](https://emscripten.org/), and [Rust](https://www.rust-lang.org/)!

This tutorial is going to go over how to create a simple compiler using LLVM for [Brainf*ck](https://en.wikipedia.org/wiki/Brainfuck) in Rust. The completed code can be found here: [brainfrick-rust](https://github.com/benkonz/brainfrick-rust)

# Brainf*ck Introduction

[Brainf-ck](https://en.wikipedia.org/wiki/Brainfuck) is a very simple, turing complete, programming language that is made up of 8 commands. Each command is a single character, so parsing out an entire brainf*ck file is easy. Just read every character and match the character with a command. If the character isn't a command, just skip it and move to the next character.

## Commands

|Character|Meaning|
|-|-|
|`>`| increment data pointer |
|`<`| decrement data pointer |
|`+`| increment the byte at the data pointer |
|`-`| decrement the byte at the data pointer |
|`.`| output the byte at the data pointer to `stdout` |
|`,`| accept one byte of input from `stdin`, storing it's value in the byte at the data pointer |
|`[`| if the byte at the data pointer is zero, then move the program counter to instruction after the matching `]`, otherwise move to the next instruction |
|`]`| if the byte at the data pointer is nonzero, then move the program counter to the instruction after the matching `[`, otherwise move to the next instruction |

## Sample Program

This simple program is going to add the current cell's value to the next cell

```brainfuck
[->+<]
```

Think of `[` and `]` as `while (*data_ptr) {` and `}`, respectivly.
We decrement the current cell's value with `-`, then increment the `data_ptr` with `>`, then increment the new value at our data pointer with `+`, then decrement the `data_ptr` back to where it was at the start of the loop with `-`.

Since this is surrounded in a `[` `]`, we are going to do this until the value at the `data_ptr` is `0`, effectivly adding the value at the `data_ptr` to the next value in memory.

# Project Setup

This project uses [LLVM-10](https://releases.llvm.org/10.0.0/docs/ReleaseNotes.html), as well as [Cargo](https://crates.io/).

Start with:

```
cargo new --bin brainfrick-rust
```

then edit the `Cargo.toml` file and add this to the dependencies:

```toml
inkwell = { git = "https://github.com/TheDan64/inkwell", branch = "llvm10-0" }
clap = "2.33"
```

`clap` is used to make parsing command line args easier

Verify that everything works by running:

```
cargo build
```

Unless you've already installed the LLVM-10 development files on your machine, you will probably get an error message complaining about being unable to find LLVM-10. If that's the case, just follow the "Compiling LLVM" instructions [here](https://crates.io/crates/llvm-sys).


# Your First Executable

We're going to start by making a simple executable that just returns `0`.

To start, we'll parse out the command line args. The first argument will be our input file and the `-o OUTPUT` will be our output file.

```rust
#[macro_use]
extern crate clap;

use clap::{App, Arg};

fn main() -> Result<(), String> {
     let matches = App::new(crate_name!())
        .version(crate_version!())
        .author(crate_authors!())
        .about(crate_description!())
        .arg(
            Arg::with_name("INPUT")
                .help("source bf file to compile")
                .required(true)
                .index(1),
        )
        .arg(
            Arg::with_name("output")
                .short("o")
                .help("output filename")
                .takes_value(true)
                .required(true),
        )
        .get_matches();
}
```

Next, we'll create our `context`, `module`, and `builder`

```rust
use inkwell::context::Context;

let context = Context::create();
let module = context.create_module("brainfrick_rust");
let builder = context.create_builder();
```

After that, we'll create a main function type, which takes no arguments and returns an `i32` and add it to our module

```rust
use inkwell::module::Linkage;

let i32_type = context.i32_type();
let main_fn_type = i32_type.fn_type(&[], false);
let main_fn = module
                .add_function("main", main_fn_type, Some(Linkage::External));
```

Next, we'll set the builder's instruction pointer to the main function and use it to build a return instruction

```rust
let basic_block = context.append_basic_block(main_fn, "entry");
builder.position_at_end(basic_block);

// our compiler will go here

let i32_zero = i32_type.const_int(0, false);
builder.build_return(Some(&i32_zero));
```

Finally, we'll have to setup LLVM to convert the builder to machine code and write that result to a file

```rust
use inkwell::targets::{
    CodeModel, FileType, InitializationConfig, RelocMode, Target, TargetMachine,
};
use inkwell::OptimizationLevel;

Target::initialize_all(&InitializationConfig::default());
// use the host machine as the compilation target
let target_triple = TargetMachine::get_default_triple();
let cpu = TargetMachine::get_host_cpu_name().to_string();
let features = TargetMachine::get_host_cpu_features().to_string();

// make a target from the triple
let target = Target::from_triple(&target_triple)
.map_err(|e| format!("{:?}", e))?;

// make a machine from the target
let target_machine = target
    .create_target_machine(
        &target_triple,
        &cpu,
        &features,
        OptimizationLevel::Default,
        RelocMode::Default,
        CodeModel::Default,
    )
    .ok_or_else(|| "Unable to create target machine!".to_string())?;

// use the machine to convert our module to machine code and write the result to a file
let output_filename = matches.value_of("output").unwrap();
target_machine
    .write_to_file(&module, FileType::Object, output_filename.as_ref())
    .map_err(|e| format!("{:?}", e))?;
```

The `FileType::Object` can be changed to `FileType::Assembly` to write human-readable machine instructions, which is very useful for debugging your compiler

## Running the executable

By setting the file type to `FileType::Object`, LLVM will produce an [Object File](https://en.wikipedia.org/wiki/Object_file). That file cannot be run by itself, and needs to be linked togeather by a linker. Typically, this is ued by [ld](https://linux.die.net/man/1/ld), but the right arguments to `ld` varies based on Linux disro. The easier way to link your object file is with [gcc](https://gcc.gnu.org/), which, internally, calls `ld` with the right args.

```
cargo run INPUT_FILE -o OUTPUT_FILE
gcc OUTPUT_FILE -o OUTPUT_FILE.exe
./OUTPUT_FILE.exe
echo $?
0
```

The `echo $?` should print out the status code of the last program. You can modify the `let i32_zero = i32_type.const_int(0, false);` line to return any integer. 

# Standard Library Functions

Before we start compiling our source program, we need to initialize the memory that our brainf*ck programs will use. This means calling the [calloc](https://en.cppreference.com/w/c/memory/calloc) libc call.

After we've initialized our `builder`, we're going to add this code

```rust
use inkwell::AddressSpace;

let i64_type = context.i64_type();
let i8_type = context.i8_type();
let i8_ptr_type = i8_type.ptr_type(AddressSpace::Generic);

let calloc_fn_type = i8_ptr_type.fn_type(&[i64_type.into(), i64_type.into()], false);
let calloc_fn = module
    .add_function("calloc", calloc_fn_type, Some(Linkage::External));

```

We're also going to want to add the `getchar` and `putchar` functions, which are going to get used by the `.` and  `,` commands.


```rust
let getchar_fn_type = i32_type.fn_type(&[], false);
let getchar_fn =module
    .add_function("getchar", getchar_fn_type, Some(Linkage::External));

let putchar_fn_type = i32_type.fn_type(&[i32_type.into()], false);
let putchar_fn =module
    .add_function("putchar", putchar_fn_type, Some(Linkage::External));
```

Next, we're going to call `calloc` and initialize our `data` and `ptr` variables, which will both point to the result of our `calloc` call. After the `builder.position_at_end(basic_block);` line, we're going to add this code:

```rust
let i8_type = context.i8_type();
let i8_ptr_type = i8_type.ptr_type(AddressSpace::Generic);

let data = builder.build_alloca(i8_ptr_type, "data");
let ptr = builder.build_alloca(i8_ptr_type, "ptr");

let i64_type = context.i64_type();
let i64_memory_size = i64_type.const_int(30_000, false);
let i64_element_size = i64_type.const_int(1, false);

let data_ptr = builder.build_call(
    calloc_fn,
    &[i64_memory_size.into(), i64_element_size.into()],
    "calloc_call",
);
let data_ptr_result: Result<_, _> = data_ptr.try_as_basic_value().flip().into();
let data_ptr_basic_val =
    data_ptr_result.map_err(|_| "calloc returned void for some reason!")?;

builder.build_store(data, data_ptr_basic_val);
builder.build_store(ptr, data_ptr_basic_val);
```

Finally, we're going to call [free](https://en.cppreference.com/w/c/memory/free) on our `data` pointer. This isn't really necessary, since the program ends right after, but it is good practice. LLVM has a convience function called `build_free`, so we don't need to add the free function to our module.

```rust
  builder
    .build_free(builder.build_load(data, "load").into_pointer_value());

```

Let's verify that everything is working correctly. Recompile our code and make another executable with `cargo run`. Re-link it with `gcc`, then run `ltrace ./OUTPUT_FILE.exe`. You should see something like this:

```
calloc(30000, 1)        = 0x55a9ec9482a0
free(0x55a9ec9482a0)    = <void>
+++ exited (status 0) +++
```

`ltrace` traces all of the library calls of an executable, so we should see a call to `calloc` and a call to `free`.

We'll need to setup some boilerplate code to read in our input file and iterate through all of the characters. After `builder.build_store(ptr, data_ptr_basic_val);`, add

```rust
use std::fs::File;
use std::io::prelude::*;
use std::collections::VecDeque;
use inkwell::basic_block::BasicBlock;

let source_filename = matches.value_of("INPUT").unwrap();
let mut f = File::open(source_filename).map_err(|e| format!("{:?}", e))?;
let mut program = String::new();
f.read_to_string(&mut program)
    .map_err(|e| format!("{:?}", e))?;

let mut while_blocks = VecDeque::new();

for command in program.chars() {
    match command {
        '>' => build_add_ptr(&context, &builder, 1, &ptr),
        '<' => build_add_ptr(&context, &builder, -1, &ptr),
        '+' => build_add(&context, &builder, 1, &ptr),
        '-' => build_add(&context, &builder, -1, &ptr),
        '.' => build_put(&context, &builder, &putchar_fn, &ptr),
        ',' => build_get(&context, &builder, &getchar_fn, &ptr)?,
        '[' => build_while_start(&context, &builder, &main_fn, &ptr, &mut while_blocks),
        ']' => build_while_end(&context, &builder, &mut while_blocks)?,
        _ => (),
    }
}
```

we will fill out these functions in the next sections

# Compiling '>' and '<'

The `>` and `<` operations are similar enough that we can make a function that moves the `ptr` by an `amount` parameter.

To move the pointer around, we'll use the `build_in_bounds_gep`, which takes a pointer and a list of ordered indexes. The function is marked as `unsafe` because it is very likely to SEGFAULT if we index outside the bounds of the array. This code doesn't do any size checks, so it is likely that indexing outside the `30_000` bytes of memory from our `calloc` call will SEGFAULT.

```rust
use inkwell::builder::Builder;
use inkwell::values::PointerValue;

fn build_add_ptr(
        context: &Context, 
        builder: &Builder, 
        amount: i32, 
        ptr: &PointerValue
    ) {
    let i32_type = context.i32_type();
    let i32_amount = i32_type.const_int(amount as u64, false);
    let ptr_load = builder
        .build_load(*ptr, "load ptr")
        .into_pointer_value();
    // unsafe because we are calling an unsafe function, since we could index out of bounds of the calloc
    let result = unsafe {
        builder
            .build_in_bounds_gep(ptr_load, &[i32_amount], "add to pointer")
    };
    builder.build_store(*ptr, result);
}
```

# Compiling '+' and '-'

The `+` and `-` operations are similar enough that we can make a function that adds the value that `ptr` is pointing to by an `amount` parameter.

First, we'll load the pointer variable, then we'll load the value at that that pointer points to. Then, we use `build_int_add` to add that value to `amount`. 

```rust
fn build_add(
        context: &Context, 
        builder: &Builder, 
        amount: i32, 
        ptr: &PointerValue
    ) {
    let i8_type = context.i8_type();
    let i8_amount = i8_type.const_int(amount as u64, false);
    let ptr_load = builder
        .build_load(*ptr, "load ptr")
        .into_pointer_value();
    let ptr_val = builder.build_load(ptr_load, "load ptr value");
    let result = builder
            .build_int_add(ptr_val.into_int_value(), i8_amount, "add to data ptr");
    builder.build_store(ptr_load, result);
}
```

# Compiling '.' and ','

Here, we'll make use of the `getchar_fn` and `putchar_fn` functions.

## Compiling '.'

```rust
use inkwell::values::FunctionValue;

fn build_get(
        context: &Context, 
        builder: &Builder, 
        getchar_fn: &FunctionValue, 
        ptr: &PointerValue
    ) -> Result<(), String> {
    // call getchar
    let getchar_call = builder
        .build_call(*getchar_fn, &[], "getchar call");

    // get the result of getchar and store it in our pointer's value
    let getchar_result: Result<_, _> = getchar_call.try_as_basic_value().flip().into();
    let getchar_basicvalue =
        getchar_result.map_err(|_| "getchar returned void for some reason!")?;
    let i8_type = context.i8_type();
    // truncate since we are converting i8's to i32's
    let truncated = builder.build_int_truncate(
        getchar_basicvalue.into_int_value(),
        i8_type,
        "getchar truncate result",
    );
    let ptr_value = builder
        .build_load(*ptr, "load ptr value")
        .into_pointer_value();
    builder.build_store(ptr_value, truncated);

    Ok(())
}
```

## Compiling ','

```rust
fn build_put(
        context: &Context, 
        builder: &Builder, 
        putchar_fn: &FunctionValue, 
        ptr: &PointerValue
    ) {
    // call putchar
    let char_to_put = builder.build_load(
    builder
        .build_load(*ptr, "load ptr value")
        .into_pointer_value(),
        "load ptr ptr value",
    );
    // sign-extend, since we are conveting from i8's to i32's
    let s_ext = builder.build_int_s_extend(
        char_to_put.into_int_value(),
        context.i32_type(),
        "putchar sign extend",
    );
    builder
        .build_call(*putchar_fn, &[s_ext.into()], "putchar call");
}
```

# Compiling '[' and ']'

For our `[` and `]`, we're going to add this struct to `main.rs`:

```rust
struct WhileBlock<'ctx> {
    while_start: BasicBlock<'ctx>,
    while_body: BasicBlock<'ctx>,
    while_end: BasicBlock<'ctx>,
}
```

## Compiling '['

We're going to use a `VecDeq<WhileBlock>` to keep track of which `]` corresponds to which `[`. Each `[`, we will push to the `VecDeq`, and each `]`, we will pop from the top of our `VecDeq`.

A `WhileBlock` is composed of three basic blocks. 

- `while_start` corresponds to the zero check. If the value at the data pointer is zero, we will jump to the end of the while block, otherwise, we will continue to the `while_body` block.

- `while_body` block corresponds to the instructions between `[` and `]`.

- `while_end` is all of the instructions after the `]`.

```rust
use inkwell::IntPredicate;

 fn build_while_start<'ctx>(
        context: &'ctx Context,
        builder: &Builder,
        main_fn: &FunctionValue,
        ptr: &PointerValue,
        while_blocks: &mut VecDeque<WhileBlock<'ctx>>,
    ) {
        // create the while block
        let num_while_blocks = while_blocks.len() + 1;
        let while_block = WhileBlock {
            while_start: context.append_basic_block(
                *main_fn, format!("while_start {}", num_while_blocks).as_str(),
            ),
            while_body: context.append_basic_block(
                *main_fn, format!("while_body {}", num_while_blocks).as_str(),
            ),
            while_end: context.append_basic_block(
                *main_fn, format!("while_end {}", num_while_blocks).as_str(),
            ),
        };
        while_blocks.push_front(while_block);
        let while_block = while_blocks.front().unwrap();

        builder
            .build_unconditional_branch(while_block.while_start);
        builder.position_at_end(while_block.while_start);

        // compare the value at ptr with zero
        let i8_type = context.i8_type();
        let i8_zero = i8_type.const_int(0, false);
        let ptr_load = builder
            .build_load(*ptr, "load ptr")
            .into_pointer_value();
        let ptr_value = builder
            .build_load(ptr_load, "load ptr value")
            .into_int_value();
        let cmp = builder.build_int_compare(
            IntPredicate::NE,
            ptr_value,
            i8_zero,
            "compare value at pointer to zero",
        );

        // jump to the while_end if the data at ptr was zero
        builder
            .build_conditional_branch(cmp, while_block.while_body, while_block.while_end);
        builder.position_at_end(while_block.while_body);
    }
```

## Compiling ']'

This function is fairly streightforward. We are just jumping back to the `start_block` of the matching `[`. The `start_block` will then re-compare the value at our `ptr`, and jump to the instruction after the `]` if it is zero.

```rust
    fn build_while_end<'ctx>(
        builder: &Builder,
        while_blocks: &mut VecDeque<WhileBlock<'ctx>>
        ) -> Result<(), String> {
        if let Some(while_block) = while_blocks.pop_front() {
            builder
                .build_unconditional_branch(while_block.while_start);
            builder.position_at_end(while_block.while_end);
            Ok(())
        } else {
            Err("error: unmatched `]`".to_string())
        }
    }

```

# Wrap Up

That's it! You should now have a functioning brainf*ck compiler using LLVM written in Rust! 

Verify everything is working by testing the examples on [wikipedia](https://en.wikipedia.org/wiki/Brainfuck#Examples)

## Addition

```
cargo run addition.bf -o output.o
gcc output.o
./a.out
7
```

## Hello, World!

```
cargo run hello.bf -o output.o
gcc output.o
./a.out
Hello World!
```

## ROT13 Cipher

```
cargo run ROT13.bf -o output.o
gcc output.o
./a.out
asdf
nfqs
wrwer
jejre
^C⏎ 
```

## Other Examples

There are loads of other cool Brainf*ck programs that people have written, such as: [mandelbrot.b](http://esoteric.sange.fi/brainfuck/utils/mandelbrot/mandelbrot.b), [hanoi.bf](http://www.clifford.at/bfcpu/hanoi.bf), and [Conway's Game of Life](http://www.linusakesson.net/programming/brainfuck/)!
