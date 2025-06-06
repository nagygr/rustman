# Installation

## Installing for the host architecture

Install `rustup`, the official installation utility of Rust. On Arch-based
distros, issue:

```bash
pacman -S rustup
```

Install the stable version and set it as default:

```bash
rustup default stable
```

## Toolchain for cross-compiling

The native toolchain for the given platform needs to be installed separately.
For example, to be able to compile to Windows, [mingw-w64-gcc][1] needs to be
installed.

Then the Rust target needs to be added by issuing:

```bash
rustup target add x86_64-pc-windows-gnu
```

Furthermore, Cargo needs to be told where the linker and `ar` are. For this, you
need to edit `~/.cargo/config.toml`:

```toml
[target.x86_64-pc-windows-gnu]
linker = "/usr/bin/x86_64-w64-mingw32-gcc"
ar = "/usr/bin/x86_64-w64-mingw32-ar"
```

>	**Note**
>
>	These paths are valid for Arch Linux and Manjaro. On other systems these
>	tools might be located elsewhere.
>
>   Also note, that in earlier versions the file was called `~/.cargo/config`
>   but it was renamed to reflect the format and now a compiler warning is
>   issued if the extension is missing.

# Working with projects

## Creating a new project with Cargo

Cargo is the project management tool of Rust.

A new project can be created by issuing:

```bash
cargo new <project-name>
```

This command creates a `Cargo.toml` file that describes the project and lists
its dependencies. It also create a `src` directory where the source files go.
There will already be a `main.rs` file with the `hello world` program:

```rust
fn main() {
    println!("Hello, world!");
}
```

This command also initializes a `git` repository in the root directory.

## Building with Cargo

The simplest way to build is by issuing:

```bash
cargo build
```

This will use the default toolchain to build the executable in debug mode and
will place it to: `target/debug/<executable-name>`, where the name of the
executable is, by default, the same as that of the project.

If a release build is needed:

```bash
cargo build --release
```

The executable will be here: `target/release/<executable-name>`.

If another toolchain should be used, e.g. for cross-compiling:

```bash
cargo build --release --target "x86_64-pc-windows-gnu"
```

The path of the executable in this case will also include the target:
`target/x86_64-pc-windows-gnu/release/first.exe`.

To get the available targets, run:

```bash
rustc --print target-list
```

## Setting compiler flags

Compiler flags can be directly set in the argument list of `rustc`. Cargo can
pass on the arguments to it, and there are several ways to achieve this. One of
this is by setting an environment variable called `RUSTFLAGS`:

```bash
RUSTFLAGS="-C opt-level=3" cargo build --release
```

The line above instructs the compiler to use the highest optimization level.

## Adding dependencies

Dependencies are added to `Cargo.toml`. The `cargo build` command automatically
downloads and links them to the project.

The simplest syntax is:

```toml
[dependencies]
time = "0.1.12"
```

where the string contains the *semver* of the crate. In some cases, more
details are needed, e.g.:

```toml
[dependencies]
gio = { version = "0.9.1", features = ["v2_44"] }
gtk = { version = "0.9.2", features = ["v3_16"] }
```

## Separating code into modules and libraries

Files are handled as modules if they have the same name of the module.
Directories can act as intermediate modules but they require a rust source file
with same name (and `.rs` extension) that exports the modules within.

The sources below reside in the following directory structure:

```
src/
	main.rs
	graphs.rs
	graphs/
		trees.rs
```

-	`main.rs`:

	```rust
	mod graphs;

	use crate::graphs::trees;

	fn main() {
		let rbt = trees::RedBlack::new();

		println!("Number of nodes: {}", rbt.nodes());
	}
	```

-	`graphs.rs`

	```rust
	pub mod trees;
	```

-	`grpahs/trees.rs`:

	```rust
	pub struct RedBlack {
		...
	}

	impl RedBlack {
		pub fun new() -> RedBlack {
			...
		}
	}
	```

# Basic types

## Strings

### Iterating over the code points of a string

```rust
let text: &'static str = "árvíztűrő tükörfúrógép";
let str_arr: Vec<char> = text.chars().collect();

for i in 0..str_arr.len() {
    println!("{}", str_arr[i])
}
```

## Function pointers

The following example shows a simple state machine realized with a vector of
function pointers and an enumeration that is used to store the current state and
index into the vector.

```rust
#[derive(Copy, Clone)]
enum State {
    Normal,
    CommentStart,
    InComment,
    EndStart,
}

struct StateData {
	state: State,
}

impl StateData {
	fn new() -> StateData {
		StateData {
			state: State::Normal,
		}
	}
}

fn normal(s: &mut StateData, c: char) {
	if c == '/' {
		s.state = State::CommentStart;
	} else {
		print!("{}", c);
	}
}

fn comment_start(s: &mut StateData, c: char) {
	if c == '*' {
		s.state = State::InComment;
	} else {
		print!("/{}", c);
	}
}

fn in_comment(s: &mut StateData, c: char) {
	if c == '*' {
		s.state = State::EndStart;
	}
}

fn end_start(s: &mut StateData, c: char) {
	if c == '/' {
		s.state = State::Normal;
	} else if c != '*' {
		s.state = State::InComment;
	}
}

fn remove_comments(code: &str) {
	let states:Vec<fn(&mut StateData, char)> = vec![
		normal,
		comment_start,
		in_comment,
		end_start,
	];

	let mut s = StateData::new();

	for c in code.chars() {
		states[s.state as usize](&mut s, c);
	}
}

fn main() {
	remove_comments(
        r#"
		/* Hello world */
		fn thisIsAFunction(s: &str) {
			println!(s);
			/*
			 * Multiline comment
			 *
			 * Try to catch me if you can
			 */
		}
	"#,
    );
}
```

# Modelling concepts

## Static and dynamic dispatch

The following example shows how static dispatch is achieved using templates
(and with the `impl` shorthand) and how dynamic dispatch works with the `dyn`
keyword. The latter involves using trait objects, having values on the heap and
working with fat pointers (that contain the address of the variable and that of
the virtual method table which points to the methods implementing the given
trait for the type).

```rust
trait Driveable {
    fn drive(&self);
}

struct Sedan {}

impl Driveable for Sedan {
    fn drive(&self) {
        println!("sedan driving")
    }
}

struct SUV {}

impl Driveable for SUV {
    fn drive(&self) {
        println!("suv driving")
    }
}

// this becomes a template function and does not
// use vtable nor heap
fn drive_vehicle(v: impl Driveable) {
    v.drive();
}

// this is equivalent to the one above
fn drive_vehicle_exp<D: Driveable>(v: D) {
    v.drive();
}

// yet another way of writing the same thing
fn drive_vehicle_where<D>(v: D)
where
    D: Driveable,
{
    v.drive();  
}

fn main() {
    let vehicles: Vec<Box<dyn Driveable>> = vec![
        Box::new(Sedan {}),
        Box::new(SUV {}),
        Box::new(SUV {}),
        Box::new(Sedan {}),
    ];

    for v in vehicles {
        v.drive();
    }

    drive_vehicle(Sedan{});
    drive_vehicle_exp(Sedan{});
    drive_vehicle_where(Sedan{});
}
```

## Effective error handling

It is customary and very comfortable to use the `?` operator to pass errors up
the call stack in functions. The catch is that this only works if all the call
that have this operator return the exact same error type.

A possible solution to this is to not return `Error` but a boxed, `dyn Error`
which will allow a polymorphic handling of the errors:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    ...
}
```

The problem with this solution is that it involves a heap allocation and
dynamic dispatch. A more idiomatic and efficient way to handle multiple error
types in Rust is to define a custom enum that represents all possible errors in
your function and implement the std::error::Error trait for it. This allows you
to still use the ? operator without incurring the performance hit of dynamic
allocation. For example:

```rust
use std::fmt;

#[derive(Debug)]
enum MyError {
    Io(std::io::Error),
    Parse(std::num::ParseIntError),
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            MyError::Io(err) => write!(f, "IO error: {}", err),
            MyError::Parse(err) => write!(f, "Parse error: {}", err),
        }
    }
}

impl std::error::Error for MyError {}

fn my_function() -> Result<(), MyError> {
    let data = std::fs::read_to_string("file.txt").map_err(MyError::Io)?;
    let number: i32 = data.trim().parse().map_err(MyError::Parse)?;
    println!("Parsed number: {}", number);
    Ok(())
}
```

By using a custom error type, you avoid heap allocation, maintain type safety,
and improve performance while still leveraging Rust's powerful error-handling
mechanisms.

# GUI

## Gtk

### Cross-compiling Gtk

#### Using the mingw toolchain locally

Theoretically, the following should work (it doesn't work for me 
at the time of writing -- 06-01-2022, neither on Arch nor on Manjaro).

Install the required mingw libraries:

-	mingw-w64-gcc
-	mingw-w64-gtk{3,4}
-	mingw-w64-cairo
-	mingw-w64-freetype2
-	mingw-w64-gdk-pixbuf2
-	mingw-w64-gettext
-	mingw-w64-graphene
-	mingw-w64-harfbuzz
-	mingw-w64-pango

Issue:

```bash
PKG_CONFIG_PATH=/usr/x86_64-w64-mingw32/lib/pkgconfig \
PKG_CONFIG_ALLOW_CROSS=1 \
cargo build --release --target "x86_64-pc-windows-gnu"
```

#### Using a Docker image

Many recommend using a Fedora-based Docker for doing essentially the same as
above. Some of these solutions produced the same error I got locally, but [this
one][2] actually worked.

Clone the repository (preferably with `--recursive` as it contains a Windows 10
theme as a submodule) and `cd` into it. Start the docker daemon (by issuing
`systemctl docker start`).

Issue:

```bash
docker build . -t gtkcross
```

Then `cd` into the root of the Gtk-based Rust project that you want to compile
and issue:

```bash
docker create -v $(pwd):/home/rust/src --name rust_gtk gtkcross:latest
```

Finally, build by issuing:

```bash
docker start -ai rust_gtk
```

>	**Note**
>
>	1.	The first step only needs to be done once. The second step only needs to
>		be done once per project. The third step shall be done every time a new
>		build is needed.
>
>	2.	The names `gtkcross` and `rust_gtk` can be freely changed.
>
>	3.	If you want to use the Windows 10 theme, add the following after `docker
>		create`:
>
>		`-e WIN_THEME=true`

The Windows executable will be packaged into `package.zip` together with all the
DLLs that it depends on. It's not the most beautiful solution but it works.

Also note, that the `target` directory will be owned by `root`.

[1]: https://archlinux.org/packages/community/x86_64/mingw-w64-gcc/
[2]: https://github.com/etrombly/rust-crosscompile
