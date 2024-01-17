A Miden assembly program is just a sequence of instructions each describing a specific directive or an operation. You can use any combination of whitespace characters to separate one instruction from another.

In turn, Miden assembly instructions are just keywords which can be parameterized by zero or more parameters. The notation for specifying parameters is *keyword.param1.param2* - i.e., the parameters are separated by periods. For example, `push.123` instruction denotes a `push` operation which is parameterized by value `123`.

Miden assembly programs are organized into procedures. Procedures, in turn, can be grouped into modules.

## Procedures

A *procedure* can be used to encapsulate a frequently-used sequence of instructions which can later be invoked via a label. A procedure must start with a `proc.<label>.<number of locals>` instruction and terminate with an `end` instruction. For example:

```
proc.foo.2
    <instructions>
end
```

A procedure label must start with a letter and can contain any combination of numbers, ASCII letters, and underscores (`_`). The number of characters in the procedure label cannot exceed 100.

The number of locals specifies the number of memory-based local words a procedure can access (via `loc_load`, `loc_store`, and [other instructions](io-operations.md#random-access-memor)). If a procedure doesn't need any memory-based locals, this parameter can be omitted or set to `0`. A procedure can have at most $2^{16}$ locals, and the total number of locals available to all procedures at runtime is limited to $2^{30}$.

To execute a procedure, the `exec.<label>`, `call.<label>`, and `syscall.<label>` instructions can be used. For example:

```
exec.foo
```
The difference between using each of these instructions is explained in the [next section](execution-contexts.md#procedure-invocation-semantics).

A procedure may execute any other previously defined procedure, but it cannot execute itself or any of the subsequent procedures. Thus, recursive procedure calls are not possible. For example, the following code block defines a program with two procedures:

```
proc.foo
    <instructions>
end

proc.bar
    <instructions>
    exec.foo
    <instructions>
end

begin
    <instructions>
    exec.bar
    <instructions>
    exec.foo
end
```

### Dynamic procedure invocation

It is also possible to invoke procedures dynamically - i.e., without specifying target procedure labels at compile time. There are two instructions, `dynexec` and `dyncall`, which can be used to execute dynamically-specified code targets. Both instructions expect [MAST root](../../architecture/programs.md) of the target to be provided via the stack. The difference between `dynexec` and `dyncall` is that `dyncall` will [change context](execution-contexts.md) before executing the dynamic code target, while `dynexec` will cause the code target to be executed in the current context.

Dynamic code execution in the same context is achieved by setting the top $4$ elements of the stack to the hash of the dynamic code block and then executing the following instruction:

```sh
dynexec
```

This causes the VM to do the following:

1. Read the top 4 elements of the stack to get the hash of the dynamic target (leaving the stack unchanged).
2. Execute the code block which hashes to the specified target. The VM must know the specified code block and hash (they must be in the CodeBlockTable of the executing Program).

Dynamic code execution in a new context can be achieved similarly by setting the top $4$ elements of the stack to the hash of the dynamic code block and then executing the following instruction:

```sh
dyncall
```

!!! note
    In both cases, the stack is left unchanged. Therefore, if the dynamic code is intended to manipulate the stack, it should start by either dropping or moving the code block hash from the top of the stack.

## Modules

A *module* consists of one or more procedures. There are two types of modules: *library modules* and *executable modules* (also called *programs*).

### Library modules

Library modules contain zero or more internal procedures and one or more exported procedures. For example, the following module defines one internal procedure (defined with `proc` instruction) and one exported procedure (defined with `export` instruction):

```sh
proc.foo
    <instructions>
end

export.bar
    <instructions>
    exec.foo
    <instructions>
end
```

### Programs

Executable modules are used to define programs. A program contains zero or more internal procedures (defined with `proc` instruction) and exactly one main procedure (defined with `begin` instruction). For example, the following module defines one internal procedure and a main procedure:

```sh
proc.foo
    <instructions>
end

begin
    <instructions>
    exec.foo
    <instructions>
end
```

A program cannot contain any exported procedures.

When a program is executed, the execution starts at the first instruction following the `begin` instruction. The main procedure is expected to be the last procedure in the program and can be followed only by comments.

### Importing modules

To invoke a procedure from an external module, the module first needs to be imported using a `use` instruction. Once a module is imported, procedures from this module can be invoked via the regular `exec` or `call` instructions as `exec|call.<module>::<label>` where `label` is the name of the procedure. For example:

```sh
use.std::math::u64

begin
    push.1.0
    push.2.0
    exec.u64::checked_add
end
```

In the above example we import `std::math::u64` module from the [standard library](../stdlib/index.md). We then execute a program which pushes two 64-bit integers onto the stack, and then invokes a 64-bit addition procedure from the imported module.

We can also define aliases for imported modules. For example:

```sh
use.std::math::u64->bigint

begin
    push.1.0
    push.2.0
    exec.bigint::checked_add
end
```

The set of modules which can be imported by a program can be specified via a Module Provider when instantiating the [Miden Assembler](https://crates.io/crates/miden-assembly) used to compile the program.

#### Re-exporting procedures

A procedure defined in one module can be re-exported from a different module under the same or a different name. For example:

```sh
use.std::math::u64

export.u64::add
export.u64::mul->mul64

export.foo
    <instructions>
end
```

In addition to the locally-defined procedure `foo`, the above module also exports procedures `add` and `mul64` implementations of which will be identical to `add` and `mul` procedures from the `std::math::u64` module respectively.

## Constants

Miden assembly supports constant declarations. These constants are scoped to the module they are defined in and can be used as immediate parameters for Miden assembly instructions. Constants are supported as immediate values for the following instructions: `push`, `assert`, `assertz`, `asert_eq`, `assert_eqw`, `locaddr`, `loc_load`, `loc_loadw`, `loc_store`, `loc_storew`, `mem_load`, `mem_loadw`, `mem_store`, `mem_storew`.

Constants must be declared right after module imports and before any procedures or program bodies. A constant's name must start with an upper-case letter and can contain any combination of numbers, upper-case ASCII letters, and underscores (`_`). The number of characters in a constant name cannot exceed 100. 

A constant's value must be in the range between $0$ and $2^{64} - 2^{32}$ (both inclusive) and can be defined by an arithmetic expression using `+`, `-`, `*`, `/`, `//`, `(`, `)` operators and references to the previously defined constants. Here `/` is a field division and `//` is an integer division. Note that the arithmetic expression cannot contain spaces.

```sh
use.std::math::u64

const.CONSTANT_1=100
const.CONSTANT_2=200+(CONSTANT_1-50)
const.ADDR_1=3

begin
    push.CONSTANT_1.CONSTANT_2
    exec.u64::checked_add
    mem_store.ADDR_1
end
```

## Comments

Miden assembly allows annotating code with simple comments. There are two types of comments: single-line comments which start with a `#` (pound) character, and documentation comments which start with `#!` characters. For example:

```sh
#! This is a documentation comment
export.foo
    # this is a comment
    push.1
end
```

Documentation comments must precede a procedure declaration. Using them inside a procedure body is an error.
