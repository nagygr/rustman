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
need to edit `~/.cargo/config`:

```toml
[target.x86_64-pc-windows-gnu]
linker = "/usr/bin/x86_64-w64-mingw32-gcc"
ar = "/usr/bin/x86_64-w64-mingw32-ar"
```

>	**Note**
>
>	These paths are valid for Arch Linux and Manjaro. On other systems these
>	tools might be located elsewhere.

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

## Adding dependencies

Dependencies are added to `Cargo.toml`. The `cargo build` command automatically
downloads and links them to the project.

The simplest syntax is:

```toml
[dependencies]
time = "0.1.12"
```

where the string contains the *semver* of the project. In some cases, more
details are needed, e.g.:

```toml
[dependencies]
gio = { version = "0.9.1", features = ["v2_44"] }
gtk = { version = "0.9.2", features = ["v3_16"] }
```

[1]: https://archlinux.org/packages/community/x86_64/mingw-w64-gcc/
