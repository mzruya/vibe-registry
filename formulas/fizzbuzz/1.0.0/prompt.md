# FizzBuzz CLI

Write a Rust CLI tool called `fizzbuzz` that plays the FizzBuzz game with style.

## Requirements

- Default: print FizzBuzz from 1 to 100
- Accept `--from N` and `--to N` to set the range
- Accept `--fizz N` and `--buzz N` to customize the divisors (default 3 and 5)
- Numbers that are divisible by the fizz number print "Fizz"
- Numbers that are divisible by the buzz number print "Buzz"
- Numbers divisible by both print "FizzBuzz"
- Other numbers print themselves
- Use colored output: Fizz in green, Buzz in blue, FizzBuzz in magenta, numbers in default
- Accept `--no-color` flag to disable colors
- The binary name must be `fizzbuzz`

## Examples

```
$ fizzbuzz --to 15
1
2
Fizz
4
Buzz
Fizz
7
8
Fizz
Buzz
11
Fizz
13
14
FizzBuzz
```

## Technical Requirements

- Language: Rust
- Use clap v4 with derive macros for CLI parsing
- Use the `colored` crate for terminal colors
- Create a complete Cargo.toml with the package name "fizzbuzz"
- The code should compile with `cargo build --release`
