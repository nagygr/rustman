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
the call stack in functions. The catch is that this only works if all the calls
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
your function and implement the `std::error::Error` trait for it. This allows you
to still use the `?` operator without incurring the performance hit of dynamic
allocation.

*For example:*

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

>   **Note**
>
>   `map_err` is a method on `Result` and it maps a `Result<T, E>` to
>   `Result<T, F>` by applying a function to a contained `Err` value, leaving
>   an `Ok` value untouched.

By using a custom error type, you avoid heap allocation, maintain type safety,
and improve performance while still leveraging Rust's powerful error-handling
mechanisms.

## I/O-bound and CPU-intensive parallelism

I/O-bound parallelism is best handled by an `async` runtime (e.g. [Tokio][3]).
The problem is that a blocking operation can ruin the performance of the entire
runtime. This is why Tokio has a way to spawn a blocking thread which is
handles differently. But that's only a single thread, so CPU-intensive
parallelism needs to be implemented on top of that, either by hand, or using a
crate like [Rayon][4].

The following example shows a use-case where multiple files are read in
parallel using Tokio, then the contents are processed also in parallel but with
Rayon (contents are split to words, stop words are removed and a word frequency
count is performed). In the end, the results are written back to a file.

```rust
use rayon::prelude::*;
use regex::Regex;
use std::collections::{HashMap, HashSet};
use std::path::PathBuf;
use std::sync::Arc;
use tokio::fs::File;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::sync::mpsc;

// --- Constants ---
const STOPWORDS_FILE: &str = "stopwords.txt";
const OUTPUT_FILE: &str = "processed_words.txt";

// --- Helper Function: Load Stopwords ---
async fn load_stopwords(filepath: &str) -> Result<HashSet<String>, Box<dyn std::error::Error>> {
    let mut file = File::open(filepath).await?;
    let mut contents = String::new();
    file.read_to_string(&mut contents).await?;
    let stopwords: HashSet<String> = contents.lines().map(|s| s.to_lowercase()).collect();
    Ok(stopwords)
}

// --- Text Processing Functions (CPU-intensive, to be run by Rayon) ---

// Splits text into words, converts to lowercase, and filters out non-alphabetic characters.
fn split_to_words(text: &str) -> Vec<String> {
    let re = Regex::new(r"\b\w+\b").unwrap(); // Matches word boundaries
    re.find_iter(text)
        .map(|m| m.as_str().to_lowercase())
        .collect()
}

// Removes stopwords from a list of words.
fn remove_stopwords(words: Vec<String>, stopwords: &HashSet<String>) -> Vec<String> {
    words
        .into_iter()
        .filter(|word| !stopwords.contains(word))
        .collect()
}

// CPU-intensive word frequency calculation operation.
fn perform_word_frequency_calculation(words_list: Vec<Vec<String>>) -> String {
    if words_list.is_empty() {
        return "No words to compare.".to_string();
    }

    // Combine all words from all files into a single list for this simple demo
    let all_words: Vec<String> = words_list.into_iter().flatten().collect();

    // Create a word frequency map
    let mut word_counts: HashMap<String, usize> = HashMap::new();
    for word in &all_words {
        *word_counts.entry(word.clone()).or_insert(0) += 1;
    }

    // A very simplistic "word_frequency" output: just return the most frequent words.
    let mut sorted_words: Vec<(&String, &usize)> = word_counts.iter().collect();
    sorted_words.sort_by(|a, b| b.1.cmp(a.1));

    let mut result = String::new();
    result.push_str("Most frequent words across all files:\n");
    for (word, count) in sorted_words.iter().take(20) {
        // Top 20 most frequent words
        result.push_str(&format!("  {}: {}\n", word, count));
    }
    result
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let args: Vec<String> = std::env::args().collect();
    if args.len() < 2 {
        eprintln!("Usage: {} <file1> <file2> ...", args[0]);
        return Ok(());
    }

    let input_files: Vec<PathBuf> = args[1..].iter().map(PathBuf::from).collect();

    // --- Load Stopwords ---
    println!("Loading stopwords...");
    let stopwords = Arc::new(load_stopwords(STOPWORDS_FILE).await?);
    println!("Stopwords loaded.");

    // --- Channels for communication between Tokio (IO) and Rayon (CPU) ---
    // Channel for sending file contents to Rayon
    let (tx_content, mut rx_content) = mpsc::channel::<(usize, String)>(100); // (index, content)
    // Channel for sending processed words back to Tokio
    let (tx_processed, mut rx_processed) = mpsc::channel::<Vec<String>>(100);

    let num_files = input_files.len();
    let stopwords_clone_for_rayon = stopwords.clone();

    // --- Spawn Tokio tasks for reading files concurrently ---
    let read_handles: Vec<_> = input_files
        .into_iter()
        .enumerate()
        .map(|(i, path)| {
            let tx_content = tx_content.clone();
            tokio::spawn(async move {
                println!("Reading file: {:?}", path);
                match File::open(&path).await {
                    Ok(mut file) => {
                        let mut contents = String::new();
                        if let Ok(_) = file.read_to_string(&mut contents).await {
                            println!("Finished reading file: {:?}", path);
                            if let Err(e) = tx_content.send((i, contents)).await {
                                eprintln!("Error sending file content: {}", e);
                            }
                        } else {
                            eprintln!("Error reading content from file: {:?}", path);
                        }
                    }
                    Err(e) => {
                        eprintln!("Error opening file {:?}: {}", path, e);
                    }
                }
            })
        })
        .collect();

    // Drop the original tx_content to signal the Rayon task when all senders are dropped
    drop(tx_content);

    // --- Spawn a Rayon task for CPU-intensive operations ---
    // The closure passed to spawn_blocking is synchronous, but it uses a new
    // current_thread Tokio runtime to execute an async block inside.
    let rayon_handle = tokio::task::spawn_blocking(move || {
        tokio::runtime::Builder::new_current_thread()
            .enable_all()
            .build()
            .unwrap()
            .block_on(async move {
                // This async block runs on the blocking thread
                let mut file_contents: Vec<Option<String>> = vec![None; num_files];
                let mut received_count = 0;

                // Receive file contents from the Tokio reader tasks
                // This `recv().await` awaits on the internal current_thread runtime
                while let Some((index, content)) = rx_content.recv().await {
                    file_contents[index] = Some(content);
                    received_count += 1;
                    if received_count == num_files {
                        break;
                    }
                }

                // Filter out None values and collect the actual contents
                let valid_contents: Vec<String> =
                    file_contents.into_iter().filter_map(|x| x).collect();

                // Process each file's content in parallel using Rayon
                let processed_words_per_file: Vec<Vec<String>> = valid_contents
                    .par_iter()
                    .map(|content| {
                        let words = split_to_words(content);
                        remove_stopwords(words, &stopwords_clone_for_rayon)
                    })
                    .collect();

                // Perform the "word frequency calculation" on all processed words
                let word_frequency_result =
                    perform_word_frequency_calculation(processed_words_per_file.clone());

                // Send the processed words (or a combined representation) back to Tokio
                // This `send().await` awaits on the internal current_thread runtime
                let all_processed_words: Vec<String> =
                    processed_words_per_file.into_iter().flatten().collect();
                let unique_words: Vec<String> = all_processed_words
                    .into_iter()
                    .collect::<HashSet<String>>()
                    .into_iter()
                    .collect();

                if let Err(e) = tx_processed.send(unique_words).await {
                    eprintln!("Error sending processed words back to Tokio: {}", e);
                }
                word_frequency_result // This String is the return value of the async block
            }) // block_on returns the result of the async block
    });

    // --- Wait for all read operations to complete (optional, but good for error handling) ---
    for handle in read_handles {
        handle.await?;
    }

    // --- Receive processed data from Rayon and write to output file ---
    let mut output_file = File::create(OUTPUT_FILE).await?;
    println!("Awaiting processed words from Rayon...");

    let mut all_unique_words_to_write = Vec::new();
    while let Some(words) = rx_processed.recv().await {
        all_unique_words_to_write.extend(words);
    }
    all_unique_words_to_write.sort(); // Sort for consistent output

    println!("Writing processed words to '{}'...", OUTPUT_FILE);
    for word in all_unique_words_to_write {
        output_file.write_all(word.as_bytes()).await?;
        output_file.write_all(b"\n").await?;
    }
    println!("Finished writing processed words.");

    // Await the Rayon task to get its return value (the word_frequency result)
    let word_frequency_output = rayon_handle
        .await
        .expect("Rayon task failed to complete or panicked");
    println!("\n--- Word Frequency Results ---");
    println!("{}", word_frequency_output);

    println!("\nText processing completed successfully!");

    Ok(())
}
```

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
[3]: https://crates.io/crates/tokio
[4]: https://crates.io/crates/rayon
