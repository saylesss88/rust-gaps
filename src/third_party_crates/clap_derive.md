# Clap Derive

## Understanding `clap` by Building It Yourself

Most tutorials start with `clap` and show you how to use it. This one starts
without it.

Before reaching for [clap](https://crates.io/crates/clap), it's worth building
the same thing by hand. Parsing arguments manually forces you to see exactly
what `clap` is doing for you, and once you see that, the derive API stops
feeling like magic and starts feeling obvious.

We'll build `echor` (Credit:
[Command-Line Rust](https://github.com/kyclark/command-line-rust/tree/main/02_echor)),
a Rust implementation of the Unix `echo` command in three passes:

1. **Raw args**: `env::args()` directly in main, no abstractions
2. **Structured args**: a typed struct with a `build` method, same shape `clap`
   uses
3. **clap derive**: replace `build()` with `#[derive(Parser)]` and watch how
   little changes

Then we'll close by looking at what `clap` is actually doing internally, by
implementing a naive version of its three-trait pipeline.

## What problem does `clap` solve?

When a user runs your program, the OS hands it a flat list of strings:

```sh
cargo run -- hello world -n
# the program receives: ["target/debug/echor", "hello", "world", "-n"]
```

Your program has to:

1. **Parse**: decode what each string means (is `-n` a flag? is `hello`
   positional text?)
2. **Validate**: check the result makes sense (did the user provide any text at
   all?)
3. **Run**: use the validated result

`clap` handles steps one and two automatically. By the time your `main` logic
runs, you're guaranteed a valid, typed `Args` struct. That guarantee, and the
help text, error messages, and type coercion you get for free. (is the entire
value proposition).

## Pass 1: Raw args in `main`

`echo` takes literal text arguments and an optional `-n` flag to suppress the
trailing newline. Let's implement it with nothing but the standard library:

```rs
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();

    let mut omit_newline = false;
    let mut text: Vec<String> = Vec::new();

    // parsing: decide where each arg goes
    for arg in args.iter().skip(1) {
        if arg == "-n" {
            omit_newline = true;
        } else {
            text.push(arg.clone());
        }
    }

    // validation: checks that the result is usable
    if text.is_empty() {
        eprintln!("error: at least one argument required");
        std::process::exit(1);
    }

    // logic: use the validated result
    print!("{}{}", text.join(" "), if omit_newline { "" } else { "\n" });
}
```

Walking through the pieces:

- `env::args().collect()` gives you every string the OS passed in, including
  `args[0]` which is always the binary path, not user input
- `.skip(1)` drops that binary name so the loop only sees actual arguments
- `omit_newline` is a bool because `-n` is either present or not
- `text` is a `Vec<String>` because echo can take multiple words. Every non-flag
  arg gets pushed in
- `text.join(" ")` collapses the vec back into a single string:
  `["hello", "world"]` → "hello world"

```bash
cargo run -- hello world
# args[0] = "target/debug/echor"  (skipped)
# args[1] = "hello"               → text
# args[2] = "world"               → text
# Output: hello world
```

This works, but everything lives in `main`: the parsing variables, the loop, the
validation, and the output logic are all tangled together. That three-phase
structure: parse, validate, run is implicit. The next pass makes it explicit.

## Pass 2: Typed struct with `build()`

The same logic, reorganized around a struct:

```rs
use std::{env, process};

#[derive(Debug)]
struct Args {
    /// The text values to be printed to standard output
    text: Vec<String>,
    /// Flag indicating whether the trailing newline (`\n`) should be omitted
    omit_newline: bool,
}

impl Args {
    /// Parses and validates cli arguments manually from `env::args()`
    ///
    /// # Errors
    /// Returns a string slice error message if no text arguments are provided
    fn build() -> Result<Args, &'static str> {
        let mut args = Args {
            text: Vec::new(),
            omit_newline: false,
        };

        // Skip the first argument (the binary path) & iterate through user inputs
        for arg in env::args().skip(1) {
            if arg == "-n" {
                args.omit_newline = true;
            } else {
                args.text.push(arg);
            }
        }

        // Validate that required arguments are present
        if args.text.is_empty() {
            return Err("error: at least one argument required");
        }

        Ok(args)
    }
}

fn main() {
    // Parse arguments and handle validation errors
    let args = Args::build().unwrap_or_else(|err| {
        eprintln!("{err}");
        process::exit(1);
    });
    // Print the joined text, appending a newline unless `-n` was specified
    print!(
        "{}{}",
        args.text.join(" "),
        if args.omit_newline { "" } else { "\n" }
    )
}
```

The code does exactly the same thing as Pass 1. What changed is the structure:

- The shape of valid input is now explicit. `Args` documents what the program
  expects just by existing
- `build()` returns `Result`, so errors are values your program can handle
  rather than panics or silent failures
- `main` is now only responsible for error handling and output. The three phases
  are cleanly separated

## Pass 3: `clap` derive

Now replace `build` with `#[derive(Parser)]`:

```rs
use clap::Parser;

/// Rust version of `echo`
#[derive(Debug, Parser)]
#[command(author, version, about)]
struct Args {
    /// Input text
    text: Vec<String>,

    /// Do not print newline
    #[arg(short('n'))]
    omit_newline: bool,
}

fn main() {
    let args = Args::parse();

    print!(
        "{}{}",
        args.text.join(" "),
        if args.omit_newline { "" } else { "\n" }
    );
}
```

> [!NOTE] Notice that `main` now receives a clean `Args` value with no `Option`
> or `Result` to unwrap. This is the pattern `clap` enforces: by the time you
> reach your logic, parsing and validation are already done.

The struct is identical. `main` is very similar, besides the above note. The
only things that disappeared are `build()` and the `unwrap_or_else`, `clap`
handles both. What you get in exchange:

- Automatic `--help` and `--version` flags
- Typed error messages with usage hints when the user passes bad input
- Type coercion (`"42"` → `u32`, paths validated as `PathBuf`, etc.)
- Short and long flag variants for free (`-n` / `--omit-newline`)

The mapping from struct fields to argument types is direct:

| **Field type**               | **What clap expects**                    |
| :--------------------------- | :--------------------------------------- |
| `String`                     | exactly one positional value             |
| `Vec<String>`                | one or more positional values, collected |
| `bool`                       | a flag, present = true, absent = false   |
| `Option<String>`             | one optional value, absent = None        |
| `Option<PathBuf>`            | one optional path, absent = None         |
| `u8` with `ArgAction::Count` | how many times the flag was passed       |

The `#[arg(...)]` attributes are overrides on top of those defaults. Without
`#[arg(short('n'))]`, `clap` would infer the flag name from the field name
(`--omit-newline`). The attribute pins it to `-n`.

## Command-Line Argument Types

Before going further, it's worth naming the categories clap works with, because
the derive attributes map directly to them.

- [`clap` CLI concepts](https://docs.rs/clap/latest/clap/_concepts/index.html)

**Positional arguments**: identified by their position, not a name. Order
matters.

```sh
cp source.txt dest.txt
#   ^^^^^^^^^^ ^^^^^^^^
#   position 1  position 2
```

**Flags**: boolean switches, either present or not. Order doesn't matter.

```sh
echo -n hello
ls -l
ls --long    # same as -l, long form
```

Short flags use a single dash and one character `-n`. Long flags use double dash
and a word `--number`. They're typically the same flag, just two ways to write
it.

**Options**: like flags but take a value after them:

```sh
cargo run --example myexample
#          ^^^^^^^^ ^^^^^^^^^
#           option     value
cut -d "," -f 1
#      ^^^    ^
#      value  value
```

**Subcommands**: a word that selects a mode, then has its own args:

```sh
cargo build --release
#     ^^^^^
jj desc -m "message"
#  ^^^^^^
```

In a clap struct the field type and attributes map directly to these categories:

| **Arg type** | **Clap representation**                                |
| :----------- | :----------------------------------------------------- |
| Positional   | field with no `short`/`long`, e.g. `text: Vec<String>` |
| Flag         | `bool` field with `short`/`long`                       |
| Option       | `Option<T>` field with `short`/`long`                  |
| Subcommand   | `#[command(subcommand)]` enum field                    |

---

### What `clap` is actually doing: a naive implementation

- [clap/clap_builder/src/derive.rs](https://github.com/clap-rs/clap/blob/master/clap_builder/src/derive.rs#L31)

`clap`'s derive macro generates implementations of three traits. Here's what
each one does, translated into plain code without the macro:

```rs
use std::env;

#[derive(Debug)]
struct Args {
    text: Vec<String>,
    omit_newline: bool,
}

impl Args {
    // equivalent of CommandFactory::command()
    // collects raw argv, skipping the binary name
    // In real clap this builds a Command schema first,
    // then parses argv against it, producing ArgMatches.
    // Here we skip the schema and return raw strings directly.
    fn command() -> Vec<String> {
        env::args().skip(1).collect()
    }

    // FromArgMatches::from_arg_matches_mut()
    // Takes the parsed input and maps it onto struct fields.
    // In real clap the input is an ArgMatches (typed map of name → value).
    // Here we're working directly with raw strings.
    fn from_matches(raw: Vec<String>) -> Result<Self, String> {
        let mut text = Vec::new();
        let mut omit_newline = false;

        for arg in raw {
            if arg == "-n" {
                omit_newline = true;
            } else if arg.starts_with('-') {
                return Err(format!("unknown flag: {}", arg));
            } else {
                text.push(arg);
            }
        }

        // validation
        if text.is_empty() {
            return Err("error: at least one argument required".to_string());
        }

        Ok(Args { text, omit_newline })
    }

    // Parser::parse()
    // orchestrates the pipeline and handles errors
    fn parse() -> Self {
        let raw = Self::command();
        match Self::from_matches(raw) {
            Ok(args) => args,
            Err(e) => {
                eprintln!("{e}");
                std::process::exit(1);
            }
        }
    }
}

fn main() {
    let args = Args::parse();
    print!(
        "{}{}",
        args.text.join(" "),
        if args.omit_newline { "" } else { "\n" }
    );
}
```

Real `clap`'s pipeline has one more step our version compresses out:

```sh
argv
  → CommandFactory::command()     builds a schema, parses argv into ArgMatches
  → ArgMatches                    a generic typed map of name → value
  → FromArgMatches::from_arg_matches_mut()   maps ArgMatches onto your struct
  → Args
```

Ours goes straight from raw strings to `Args`, skipping `ArgMatches`. The reason
`clap` separates these is flexibility, you can use `ArgMatches` directly without
a struct if you want, or implement `FromArgMatches` on your own type. The derive
macro just generates all three trait implementations from your struct
definition.

The traits `clap` splits this across:

- [CommandFactory](https://docs.rs/clap/latest/clap/trait.CommandFactory.html):
  builds the Command schema
- [FromArgMatches](https://docs.rs/clap/latest/clap/trait.FromArgMatches.html):
  converts `ArgMatches` into your struct
- [Parser](https://docs.rs/clap/latest/clap/trait.Parser.html): orchestrates
  both and handles errors

---

### Structs vs enums: choosing the right shape

Use a struct when your program has one mode and all arguments apply to every run
(`echo`, `cat`, `ls`).

Use an enum when your program has multiple modes with different arguments per
mode:

```rs
#[derive(Parser)]
struct Cli {
    #[command(subcommand)]
    command: Option<Commands>,
}

#[derive(Subcommand)]
enum Commands {
    /// Add a file
    Add { path: PathBuf },
    /// Remove a file
    Remove { path: PathBuf, #[arg(short)] force: bool },
    /// List files
    List,
}
```

`Add`, `Remove`, and `List` each have different arguments. You can't represent
this cleanly in a flat struct because a struct implies all fields exist for
every invocation. An enum variant is exclusive: only one runs at a time.

Then in `main` you match on it:

```rs
match cli.command {
    Commands::Add { path } => { ... }
    Commands::Remove { path, force } => { ... }
    Commands::List => { ... }
}
```

Most real tools combine both. A top-level struct for global flags (like
`--verbose`, `--config`) with an enum field inside it for the subcommand,
exactly like the example above.

---

### Resources

- [docs.rs clap](https://docs.rs/clap/latest/clap/index.html)

- [derive tutorial](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html)

- [builder tutorial](https://docs.rs/clap/latest/clap/_tutorial/index.html)

**Looking at tools you're familiar with can help drive concepts home**

- [Example: git-like CLI (Derive API)](https://docs.rs/clap/latest/clap/_cookbook/git_derive/index.html)

- [Example: pacman-like CLI (Derive API)](https://docs.rs/clap/latest/clap/_cookbook/pacman/index.html)

- [betterdev cli arguments anatomy](https://betterdev.blog/command-line-arguments-anatomy-explained/)
