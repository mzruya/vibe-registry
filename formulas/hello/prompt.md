# Hello CLI

Write a Rust CLI tool called `hello` that greets users.

## Requirements

- When run with no arguments: print "Hello, world!"
- When run with a name argument: print "Hello, <name>!"
- When run with `--shout` flag: print the greeting in ALL CAPS
- When run with `--count N` flag: print the greeting N times
- Use `clap` for argument parsing with derive macros
- The binary name must be `hello`

## Examples

```
$ hello
Hello, world!

$ hello Alice
Hello, Alice!

$ hello Alice --shout
HELLO, ALICE!

$ hello Alice --count 3
Hello, Alice!
Hello, Alice!
Hello, Alice!
```

## Technical Requirements

- Language: Rust
- Use clap v4 with derive macros for CLI parsing
- Create a complete Cargo.toml with the package name "hello"
- The code should compile with `cargo build --release`
