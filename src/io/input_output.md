# Input and Output (I/O)

Before writing any Rust, it helps to understand what I/O actually is at the
operating system level, because `std::io` is just a safe wrapper around these
primitives.

## What I/O Is

Every program that runs on your system gets three open "channels" from the OS
automatically:

| **Stream** | **Number** | **Purpose**                        |
| :--------- | :--------- | :--------------------------------- |
| **stdin**  | O          | Data coming _into_ your program    |
| **stdout** | 1          | Normal output going _out_          |
| **stderr** | 2          | Errors and diagnostics going _out_ |

These are called **file descriptors**, and despite the name, they don't have to
be files. They can be your terminal, a file on disk, a network socket, or the
output of another program. The OS abstracts all of that away behind a uniform
interface.

**The Terminal is the Default**

When you run a program in your shell without doing anything special, all three
streams connect to your terminal:

- You type → that goes to **stdin**
- The program prints → that goes to **stdout** → you see it
- The program errors → that goes to **stderr** → you also see it (but
  separately)

This is why you can see both normal output and error messages in the same
terminal window. They're two distinct streams that just happen to both display
there by default.

## Redirection and Pipes

The shell lets you rewire these streams. This is where I/O becomes composable:

```sh
# Send stdout to a file instead of the terminal
echo "hello" > output.txt

# Feed a file into stdin instead of the keyboard
sort < names.txt

# Pipe stdout of one program into stdin of another
cat names.txt | sort | uniq
```

That last example chains three programs together. Each one reads from stdin and
writes to stdout.None of them know or care what's on the other end. This is the
Unix philosophy: small programs that do one thing, composable via streams.

### stderr Stays Separate

Notice that `>` only redirects stdout. stderr keeps going to your terminal:

```sh
# This saves stdout to a file, but errors still print to your screen
cargo build > build_output.txt

# To redirect stderr too:
cargo build > build_output.txt 2> errors.txt

# Or merge both into one file:
cargo build > all_output.txt 2>&1
```

This separation matters: tools that process your output (like `grep` or `awk`)
read from stdout. You don't want error messages polluting that stream.

## I/O in Rust

Rust's `std::io` module gives you handles to those three OS streams you just
read about. The mapping is direct:

| **OS Stream** | **Rust**            |
| :------------ | :------------------ |
| stdin (0)     | `std::io::stdin()`  |
| stdout (1)    | `std::io::stdout()` |
| stderr (2)    | `std::io::stderr()` |

**stdin Is Not Your Command-Line Arguments**

This trips up a lot of newcomers. When you run:

```sh
cargo run myfile.txt
```

`myfile.txt` does not arrive via stdin. It's a command-line argument. A string
passed to your program by the shell before it even starts. You access those
through `std::env::args()`, which we'll cover in the next section.

stdin is a stream: data that flows into your program while it's running. That
happens through pipes and redirection:

```sh
# This sends data through stdin
echo "hello" | cargo run

# So does this
cargo run < myfile.txt

# This does NOT: "myfile.txt" is an argument, not stdin
cargo run myfile.txt
```

A program like `cat` handles both: if you give it a filename as an argument it
opens that file itself, but if you pipe data in it reads from stdin. That dual
behavior is a common Unix convention, and you'll implement exactly that pattern
later when we get to argument parsing.

### Reading from stdin

The simplest case: read a line the user types.

```rs
use std::io;
use std::io::BufRead;

fn main() {
    let stdin = io::stdin();
    let mut line = String::new();

    stdin.lock().read_line(&mut line).expect("failed to read line");

    println!("You typed: {line}");
}
```

A few things worth noticing:

- `stdin()` returns a handle to the stdin stream
- `.lock()` gives you exclusive access to it, stdin is shared across threads, so
  you lock it before reading
- `read_line` appends into a `String` and includes the trailing `\n`. You'll
  often want `.trim()` when you use the value
- The `BufRead` trait is what provides `read_line`. It has to be in scope for
  the method to be available

**Why BufRead?**

Reading from a stream one byte at a time is expensive. Each read is a syscall
into the OS kernel.Buffering solves this by reading a chunk at a time into
memory, then serving your code from that buffer.

`stdin()` is already buffered by default. When you call `.lock()` you get a
`StdinLock`, which implements `BufRead`. For files and other readers you'll
often wrap them yourself:

```rust
use std::io::{self, BufRead, BufReader};
use std::fs::File;

fn main() {
    let file = File::open("names.txt").expect("could not open file");
    let reader = BufReader::new(file);

    for line in reader.lines() {
        let line = line.expect("failed to read line");
        println!("{line}");
    }
}
```

The same `BufRead` trait works whether you're reading from stdin, a file, or
anything else that implements Read. That's the point, your code doesn't need to
care what's on the other end.

## Writing to stdout and stderr

`println!` and `eprintln!` cover most cases, but you can also write to the
handles directly:

```rust
use std::io::{self, Write};

fn main() {
    let mut stdout = io::stdout();
    let mut stderr = io::stderr();

    writeln!(stdout, "this goes to stdout").unwrap();
    writeln!(stderr, "this goes to stderr").unwrap();
}
```

One gotcha with stdout: it's line-buffered by default when connected to a
terminal. Output is flushed when a newline is written. If you need output to
appear immediately without a newline. Like a prompt, you have to flush manually:

```rust
use std::io::{self, Write};

fn main() {
    print!("Enter your name: ");
    io::stdout().flush().unwrap();

    let mut name = String::new();
    io::stdin().lock().read_line(&mut name).unwrap();
    println!("Hello, {}!", name.trim());
}
```

Without the `flush()` call the prompt might not appear before the program blocks
waiting for input.

## Reading All of stdin

When your program is used in a pipe, you often want to process everything coming
in rather than one line at a time:

```rust
use std::io::{self, Read};

fn main() {
    let mut input = String::new();
    io::stdin().lock().read_to_string(&mut input).unwrap();

    println!("Got {} bytes", input.len());
}
```

You can test this directly from the shell:

```sh
echo "hello world" | cargo run
```

This is the pattern most Unix-style Rust tools are built on. Read from stdin,
write to stdout, report errors to stderr.
