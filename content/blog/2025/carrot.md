+++
title = "A programming language built ontop of SQLite's VM - Pt 1"
date = 2025-10-20
author = 'Diego Reis'
tags = [
    "Databases",
    "Programming languages",
]
# comments = true
+++

SQLite is cool. It's the most deployed database in the world, it's tiny, expressive and its architecture is quite different than traditional relational databases. SQLite uses a Virtual Machine (VM) called Virtual DataBase Engine (VDBE) to interpret SQL, rather than [Volcano-style](https://paperhub.s3.amazonaws.com/dace52a42c07f7f8348b08dc2b186061.pdf) or [vectorization](https://www.vldb.org/pvldb/vol11/p2209-kersten.pdf). Do you know which other project uses VMs to interpret code? [Lua](https://www.lua.org/doc/jucs05.pdf), [Java](https://en.wikipedia.org/wiki/Java_virtual_machine), [Erlang](https://www.erlang.org/blog/a-brief-beam-primer/) and ~god forbid~ [JavaScript](https://nodejs.org/api/vm.html).

But what if we used SQLite’s VM to build a programming language? That's what this series is about.


## Introduction

So, what's a VM?

It's basically an _*emulation of how computers work*_, you can have emulators for real world computer systems with full OS and driver support (like QEMU),
simple CPU emulators (like this [8080 emulator](https://github.com/el-yawd/emulator-8080)) or even ✨_abstract machines_✨ that only exist on paper but can, through emulation, run real code.

For instance, the instructions ran by Lua are defined [here](https://www.lua.org/source/5.4/lopcodes.h.html), this is the instruction to add two numbers:

```C
// It adds the values in registers B and C and save it on register A
OP_ADD,/*       A B C   R[A] := R[B] + R[C] */
```

SQLite also defines its own [instruction format](https://sqlite.org/opcode.html), with simple instructions like [Add](https://sqlite.org/opcode.html#Add) and fancier ones like [CreateBtree](https://sqlite.org/opcode.html#CreateBtree).

Choosing which instructions your Instruction Set should have is a non-trivial problem, it's a balance between performance and simplicity specially considering other
architecture decisions like: Stack vs Register based, dispatch strategy, whether or not to do JIT, etc.

VMs are powerful, they allow you to encode full Turing complete programs with a relatively simple structure, you just need a Program Counter (PC), an Instruction Set
Architecture (ISA) and a form of storing your data/opcodes (e.g pilling them onto a stack or using virtual registers), and... you're done.

We'll start by building a tiny VM in Rust to keep things less abstract. Once we've got that foundation we'll look at how SQLite's VM works. but if you truly want to understand this I couldn't recommend a better book than [Crafting Interpreters](https://craftinginterpreters.com/) -- the second part :).

## The DVM (Dumb Virtual Machine)

> "What I cannot create I cannot understand"
>               -- Richard Feynman

We're going to do a simple VM that just executes mathematical expressions. We can define our dumb VM and instruction format like this:

```Rust
struct DVM {
    pc: usize,
    registers: [u32; 16],
    program: Vec<Insn>,
}

enum Insn {
    Add { lhs: u32, rhs: u32, dest: u32 },
    Sub { lhs: u32, rhs: u32, dest: u32 },
    Mul { lhs: u32, rhs: u32, dest: u32 },
    Div { lhs: u32, rhs: u32, dest: u32 },
    LoadImm { dest: u32, value: u32 },
    Return,
}
```

Running a program is dead simple, just execute the instruction in the position
indicated by pc's value and increment pc:

```Rust
pub fn run(&mut self) {
        while self.pc < self.program.len() {
            match &self.program[self.pc] {
                Insn::Add { lhs, rhs, dest } => {
                    let result =
                        self.registers[*lhs as usize].wrapping_add(self.registers[*rhs as usize]);
                    self.registers[*dest as usize] = result;
                    self.pc += 1;
                }
                Insn::Sub { lhs, rhs, dest } => {
                    let result =
                        self.registers[*lhs as usize].wrapping_sub(self.registers[*rhs as usize]);
                    self.registers[*dest as usize] = result;
                    self.pc += 1;
                }
                Insn::Mul { lhs, rhs, dest } => {
                    let result =
                        self.registers[*lhs as usize].wrapping_mul(self.registers[*rhs as usize]);
                    self.registers[*dest as usize] = result;
                    self.pc += 1;
                }
                Insn::Div { lhs, rhs, dest } => {
                    let result = self.registers[*lhs as usize] / self.registers[*rhs as usize];
                    self.registers[*dest as usize] = result;
                    self.pc += 1;
                }
                Insn::LoadImm { dest, value } => {
                    self.registers[*dest as usize] = *value;
                    self.pc += 1;
                }
                Insn::Return => break,
            }
        }
    }
```

Yep, the heart of our VM is just a match inside a for loop, no magic at all. The task
of our parser is to transform the language defined by our syntax in a sequence of instructions. A simple architecture for this is:

```
Code
   ↓
Parser → AST
   ↓
Code generator
   ↓
VM bytecode
   ↓
Virtual Machine executes instructions
```

For instance, `2 + 2` results in the program:

```Rust
[
    LoadImm { dest: 1, value: 2 },
    LoadImm { dest: 2, value: 2 },
    Add { lhs: 1, rhs: 2, dest: 3 },
    Return
]
```

```Bash
> 2 + 2
4
```

You can check out the full implementation [here](https://github.com/el-yawd/dvm)

# Real Stuff

Now that you just did VMs 101, let's take a deeper look into how SQLite works.
