# Countdown Timer CLI

Write a Rust CLI tool called `countdown` that displays a countdown timer in the terminal.

## Requirements

- Accept a duration argument in seconds: `countdown 60`
- Also accept human-readable formats: `countdown 5m`, `countdown 1h30m`, `countdown 90s`
- Display a large, clear countdown in the terminal that updates every second
- Show remaining time in HH:MM:SS format
- When the countdown reaches zero, print a completion message and optionally play a bell sound (\x07)
- Use `--silent` flag to suppress the bell
- Use `--message` flag to set a custom completion message
- The binary name must be `countdown`

## Examples

```
$ countdown 10
  00:00:10
  # updates every second
  00:00:00
  Time's up!

$ countdown 5m --message "Break is over!"
  00:05:00
  # ...
  00:00:00
  Break is over!
```

## Technical Requirements

- Language: Rust
- Use clap v4 with derive macros for CLI parsing
- Use crossterm or similar for terminal manipulation (clearing lines)
- Create a complete Cargo.toml with the package name "countdown"
- The code should compile with `cargo build --release`
