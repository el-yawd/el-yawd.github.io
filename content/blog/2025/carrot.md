+++
title = "Building a programming language using SQLite's VM - Pt 1"
date = 2025-10-20
author = 'Diego Reis'
tags = [
    "Databases",
    "Programming languages",
]
# comments = true
+++

SQLite is cool.

It's the most deployed database in the world, it's tiny, expressive and its architecture is quite different from traditional relational databases. SQLite uses a Virtual Machine (VM) called Virtual DataBase Engine (VDBE) to interpret SQL, rather than [Volcano-style](https://paperhub.s3.amazonaws.com/dace52a42c07f7f8348b08dc2b186061.pdf) or [vectorization](https://www.vldb.org/pvldb/vol11/p2209-kersten.pdf). Do you know which other project uses VMs to interpret code? [Lua](https://www.lua.org/doc/jucs05.pdf), [Java](https://en.wikipedia.org/wiki/Java_virtual_machine), [Erlang](https://www.erlang.org/blog/a-brief-beam-primer/) and ~god forbid~ [JavaScript](https://nodejs.org/api/vm.html).

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

## Real Stuff

Now that you just did VMs 101, let's take a deeper look into how SQLite's works. Each
SQL statement is transformed in a sequence of instructions, for example:

```sql
sqlite> explain select 1 + 1;
addr  opcode         p1    p2    p3    p4             p5  comment
----  -------------  ----  ----  ----  -------------  --  -------------
0     Init           0     4     0                    0   Start at 4
1     Add            2     2     1                    0   r[1]=r[2]+r[2]
2     ResultRow      1     1     0                    0   output=r[1]
3     Halt           0     0     0                    0
4     Integer        1     2     0                    0   r[2]=1
5     Goto           0     1     0                    0
```

It works a bit different than our VM but the concept is the same, let's break this step by step (pun intendent):

1. Every program start at 0 with the `Init` instruction, the value in the register p2 indicates where should we
go next, in this case to instruction 4;
2. Here we define a integer constant, `1`, in the register 2 `r[2]`. Note the code generator is smart enough
to know that our lhs and rhs are the same number, so it only allocates a single constant.
3. `Goto` jumps to the instruction with the index defined in p2.
4. `Add` takes the value of p1 and p2 and stores the result in p3 (quite similar to what we did).
5. Roghtly `ResultRow` takes the registers p1 through p1+p2-1 and builds a single row of results.
6. `Halt` is equivalent to our `Return`, it ends the computation.

Now look what's the program to create a new table:

```SQL
sqlite> EXPLAIN CREATE TABLE t(x INTEGER);
addr  opcode         p1    p2    p3    p4             p5  comment
----  -------------  ----  ----  ----  -------------  --  -------------
0     Init           0     28    0                    0   Start at 28
1     ReadCookie     0     3     2                    0
2     If             3     5     0                    0
3     SetCookie      0     2     4                    0
4     SetCookie      0     5     1                    0
5     CreateBtree    0     2     1                    0   r[2]=root iDb=0 flags=1
6     OpenWrite      0     1     0     5              0   root=1 iDb=0
7     NewRowid       0     1     0                    0   r[1]=rowid
8     Blob           6     3     0                   0   r[3]= (len=6)
9     Insert         0     3     1                    8   intkey=r[1] data=r[3]
10    Close          0     0     0                    0
11    Close          0     0     0                    0
12    Null           0     4     5                    0   r[4..5]=NULL
13    Noop           2     0     4                    0
14    OpenWrite      1     1     0     5              0   root=1 iDb=0; sqlite_master
15    SeekRowid      1     17    1                    0   intkey=r[1]
16    Rowid          1     5     0                    0   r[5]= rowid of 1
17    IsNull         5     25    0                    0   if r[5]==NULL goto 25
18    String8        0     6     0     table          0   r[6]='table'
19    String8        0     7     0     t              0   r[7]='t'
20    String8        0     8     0     t              0   r[8]='t'
21    Copy           2     9     0                    0   r[9]=r[2]
22    String8        0     10    0     CREATE TABLE t(x INTEGER) 0   r[10]='CREATE TABLE t(x INTEGER)'
23    MakeRecord     6     5     4     BBBDB          0   r[4]=mkrec(r[6..10])
24    Insert         1     4     5                    0   intkey=r[5] data=r[4]
25    SetCookie      0     1     1                    0
26    ParseSchema    0     0     0     tbl_name='t' AND type!='trigger' 0
27    Halt           0     0     0                    0
28    Transaction    0     1     0     0              1   usesStmtJournal=1
29    Goto           0     1     0                    0
```

I won't go through every instruction but I think you got the idea.

Its implementation lies at [vdbe.c](https://github.com/sqlite/sqlite/blob/master/src/vdbe.c),
the equivalent of our `run` function is [sqlite3VdbeExec](https://github.com/sqlite/sqlite/blob/984e4468914b50bc77d1c1c931b913d93a9f6496/src/vdbe.c#L844),
which is basically a _"massive switch statement"_. The main interface of SQLite is [sqlite3_step](https://github.com/sqlite/sqlite/blob/984e4468914b50bc77d1c1c931b913d93a9f6496/src/vdbeapi.c#L895), every time we call it
it will perform a _step_ in the computation inside the VM, until it reaches an error or ends the result. Cool right?

## Why?

Now that we're SQLite experts we can start hacking on it. But as a great philosopher said once:

> Why?

As far as I can tell nobody did this before, so it's cool. Besides that, SQLite provides nice primitives and ACID semantics that no
other VM do. For instance, SQL has transactions, where roughly a set of statements either all succesfully happens and none happens,
we could use [function coloring](https://www.tedinski.com/2018/11/13/function-coloring.html) to create ✨_transactional functions_✨, so
if something goes wrong we can rollback our execution context to the point before doing the operation. Also, we could steal some ideas from
Lua and expose [tables](https://www.lua.org/pil/2.5.html) as a primitive data type.

I'm not a programming language expert (in fact this is my first PL),
so I'm not concerned with full correctness or even if this is a good idea in the first place, so let's get some fun :)


## Carrot, our language

What I want here is to detach the VM of SQL and be able to write programs like:

```Rust
fn fibo(x) {
    if x == 1 || x == 0 {
        return 1;
    }
    return fibo(x - 1) + fibo(x - 2);
}

/// Transactional functions(!). More on that later :)
txn fn pay() {
    let balance = user.balance();
    if balance < 0 {
        abort;
    }
    /// ...
}

print("Hello world");
```

Excited? Me too. Stay tune to the second part where we'll start to do it!
