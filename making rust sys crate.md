# Making a `*-sys` crate

`*-sys` is a [naming convention](https://doc.rust-lang.org/cargo/reference/build-scripts.html#-sys-packages) for [crates](https://crates.io/) that help [Rust](https://www.rust-lang.org/) programs use C ("system") libraries, e.g. [openssl-sys](https://github.com/alexcrichton/openssl-sys), [libz-sys](https://github.com/alexcrichton/libz-sys), [lcms2-sys](https://github.com/kornelski/rust-lcms2-sys). The task of the sys crates is expose a minimal low-level C interface to Rust ([FFI](https://doc.rust-lang.org/book/ffi.html)) and to tell Cargo how to link with the library. Adding higher-level, more Rust-friendly interfaces for the libraries is left to "wrapper" crates built as a layer on top of the sys crates (e.g. a "rusty-image-app" may depend on high-level "png-rs", which depends on low-level "libpng-sys", which depends on "libz-sys").

Using C libraries in a portable way involves a bit of work: finding the library on the system or building it if it's not available, checking if it is compatible, finding C headers and converting them to Rust modules, and giving Cargo correct linking instructions. Often every step of this is tricky, because operating systems, package managers and libraries have their unique quirks that need special handling.

Fortunately, all this work can be done once in a [build script](https://doc.rust-lang.org/cargo/reference/build-scripts.html), and [published](https://doc.rust-lang.org/book/second-edition/ch14-02-publishing-to-crates-io.html) as a [`<insert library name>-sys`](https://crates.io/categories/external-ffi-bindings) Rust crate. This way other Rust programmers will be able to use the C library without having to re-invent the build script themselves.

To make a sys crate:

  0. Read [the Cargo build script documentation](https://doc.rust-lang.org/cargo/reference/build-scripts.html).
  1. Create a new Cargo crate `cargo new <library name>-sys`
  2. In `Cargo.toml` add `links = <library name>`.
     This informs Cargo that this crate links with the given C library, and Cargo will ensure that only one copy of the library is linked. Use names without any prefix/suffix (without prefix/suffix, so `florp`, not `libflorp.so`). Note that [`links`](https://doc.rust-lang.org/cargo/reference/build-scripts.html#the-links-manifest-key) is only informational and it does not actually link to anything.
  3. Create [`build.rs`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-build-field-optional) file in the root of the project (or specify `build = "<path to build.rs>"` in `Cargo.toml`).

For the `build.rs` script the general approach is:

  1. Find the library.
  2. Select static or dynamic linking.
  3. Optionally build the library from source.
  4. Expose C headers.

## Things that sys crates should _NOT_ do

Cargo build process is supposed to be entirely self-contained and work off-line.

Don't write any files outside Cargo's [dedicated output directory](https://doc.rust-lang.org/cargo/reference/environment-variables.html#environment-variables-cargo-sets-for-build-scripts) (`OUT_DIR`). Specifically, do not try to install any packages on the system. If the required library or other dependency is missing, [report an error](https://doc.rust-lang.org/beta/std/macro.panic.html) and tell the user how to install it.

Avoid downloading anything. There are packaging and deployment tools that require builds to run off-line in isolated containers. If you want to build the library from source, bundle the source in the Cargo crate.

## Finding the library

For Linux/BSD-like systems `pkg-config` is usually the best choice. There's [`pkg_config`](https://crates.io/crates/pkg-config) crate for this.

On macOS with Homebrew `pkg-config` technically works, but is problematic. Executables dynamically linked with `pkg-config`'s libraries are not redistributable and will crash on other machines that don't have Homebrew with the same version of the library installed. Because macOS users expect applications to "just work", it's recommended to use static linking by default on macOS. The exception is if you're building Homebrew formula. There, you can specify Homebrew's dependencies and link dynamically.

For Windows there's [vcpkg](https://crates.io/crates/vcpkg), but it's rarely available. You may need to search most likely directories "[the hard way](https://github.com/rust-lang-nursery/glob)" (e.g. clang-sys [searches](https://github.com/KyleMayes/clang-sys/blob/e6948cb8e7d6d67d6eb409aa788699b401ef9585/build.rs#L124-L129) `C:\Program Files\LLVM`). It's best to have a [build-from-source fallback](https://github.com/kornelski/rust-libpng-sys/blob/065b1b2a292a9bad613c2044e3357d6f46c8b77e/build.rs#L12-L14) to offer a hassle-free crate.

Finally, support [overriding library location](https://github.com/kornelski/rust-lcms2-sys/blob/bed8356d67ff21fd0476ce2045ad410272ab553a/src/build.rs#L16-L26) with `<LIBRARY_NAME>_PATH` or `<LIBRARY_NAME>_LIB_DIR` environmental variable. This is necessary in some cases, such as cross-compilation and custom builds of the library (e.g. with custom features enabled, or if the library installed in `/lib` is 6 years old).

## Select static or dynamic linking

The script must tell Cargo how to link by printing `cargo:rustc-link-lib=<name>` or `cargo:rustc-link-lib=static=<name>`. Because most users won't configure your crate (it's likely to be a dependency of a dependency of a dependency…), you should have good, safe defaults:

* Linux & BSD (except the [`musl` target](https://blog.rust-lang.org/2016/05/13/rustup.html#example-building-static-binaries-on-linux)) — use dynamic linking by default. You can expect the program to be packaged as [RPM](https://crates.io/crates/cargo-rpm)/[deb](https://crates.io/crates/cargo-deb), which will ensure these libraries are installed for you in the right places. For the `musl` target default to everything static, since it's mainly used for making entirely self-contained Linux executables.

* macOS — use static linking by default, unless you're writing a sys crate for a library shipped with macOS. Dynamic linking option may be useful for Rust programs that are later bundled in an app/framework bundle or installer package, but in Cargo that's not the default and it's complicated to set up, so it should be opt-in.

* Windows — use static linking by default, unless you're writing a sys crate for a library shipped with Windows. Dynamic linking option may be useful for applications having installers.

Note that `pkg-config` has a [`.statik()`](https://docs.rs/pkg-config/0.3.9/pkg_config/struct.Config.html#method.statik) option, but it often doesn't do anything. You may need to verify it (Linux: `ldd`, macOS: `otool -L`, Windows: `dumpbin /dependents`) and work around it.

As for the configuration itself, there are two options, both somewhat flawed:

### Cargo features

In `Cargo.toml` you can have `[features]` section with `static` and `dynamic` options.

```toml
[features]
static = []
dynamic = []
```

Don't put any of them as Cargo's `default` feature, because it's too hard to unset defaults in Cargo.

**Pro:** The features are easy to set by other crates. It's possible to configure the build entirely via `Cargo.toml`.

**Con:** Cargo features can only be set, and never unset. Once any crate anywhere makes the choice, it won't be possible to override it. Also Cargo doesn't support mutually-exclusive features, so your `build.rs` script will have to deal with both `static` and `dynamic` set at the same time.

### Environmental variables

You can check `<LIBRARY_NAME>_STATIC` environmental variable to see if the library should be linked statically.

**Pro:** Top-level project can easily override linking, even if the sys crate is a deeply nested dependency.

**Con:** Cargo doesn't help managing env vars, so the proper build will require extra scripts/tools besides Cargo.


Ideally, you can support both, with env vars taking precedence over features. This allows convenient features in simple cases, and env var as a fallback for cases where features fail.

## Build from source

If the library is unlikely to be installed on the system by default, especially when you support Windows, it's nice to automatically build it from source (and link statically).

It's a massive usability improvement for users, because they can always run `cargo build` and get something working, instead of getting errors, searching for packages, installing dependencies, setting up search paths, etc.

Downloading of the source is tricky, so it's best to avoid it. The build script would need to depend on libraries for HTTP + TLS + unarchving, which itself may be bigger and more problematic to build than just bundling sources with the sys crate. Some users require builds to work off-line. So unless the library's sources are really huge, just copy them to the Rust crate (and make sure to use a compatible [license](https://doc.rust-lang.org/cargo/reference/manifest.html#package-metadata) for your sys crate).

To avoid having duplicate copy of someone else's source code in your crate's git repository, you can add the C sources as a git submodule. Publishing to crates.io will copy it as a regular directory, so your crate's users won't even know it was a submodule.

    git submodule add https://…/example.git vendor/

During development use `cargo build -vv` to see output from the build script.

For building you have two options:

### Use the original build system of the library

You assume that the required build system (such as `make`, `autotools`, `cmake`) is installed, and just [run](https://doc.rust-lang.org/std/process/struct.Command.html) it. There are crates like [cmake](https://crates.io/crates/cmake) and [make-cmd](https://crates.io/crates/make-cmd) that help with this a *little* bit.

You may need to translate [Cargo's environmental variables](https://doc.rust-lang.org/cargo/reference/environment-variables.html) into appropriate [options for the build system](https://github.com/alexcrichton/git2-rs/blob/d90163af06e3e19f5c2c651c7e69760c357dc618/libgit2-sys/build.rs#L106-L131) to control output directories, optimization level, debug symbols, and enable `-fPIC` (Rust always wants `-fPIC`).

**Pro:** You just stick to documented way of building the library and don't peek inside.

**Con:** If the build fails, the extra indirection makes it even harder to diagnose and fix. Very likely to be painful on Windows.

### Replace the build system using [cc](https://crates.io/crates/cc).

Replacing the build system **seems** like a terrible idea inviting a lot of maintenance work. However, for a lot of libraries all the complexity of the build system exists only to handle differences between operating systems and broken compilers (which the `cc` crate handles) and searching for the library's own dependencies (which other `*-sys` crates can do for you).

In practice it often turns out that just giving the list of `.c` files to the `cc` crate, with one or two [`define()`s](https://docs.rs/cc/1.0.10/cc/struct.Build.html#method.define), is enough to build the code properly! Try running `make --dry-run VERBOSE=1` to see the files and macros needed.

If you need to [make a `config.h` file](https://github.com/kornelski/mozjpeg-sys/blob/384688f9c23e94ddeb353d414d45ede69768ec08/src/build.rs#L84-L96) for the library, don't modify the source directory. Instead, write the config header to the `OUT_DIR` and set the out dir in [include](https://docs.rs/cc/1.0.10/cc/struct.Build.html#method.include) path first.

**Pro:** The [`cc` crate](https://crates.io/crates/cc) handles integration with Cargo, even for cross-compilation. It also handles things you'd rather not do, like reading Windows Registry to find a working copy of Visual Studio.

**Con:** It's easy for small/medium and mature projects, but may be daunting for big and fast-moving projects.

### Customization

C libraries often have option of enabling/disabling features via `#define FOO_SUPPORTED`. It's a good idea to translate this into Cargo features. If the C library has some features enabled by default, set Cargo's default features the same way.

```toml
[features]
default = ["foo"]
foo = []
bar = []
```

```rust
if cfg!(feature = "foo") {
    cc.define("FOO_SUPPORTED", Some("1"));
}
if cfg!(feature = "bar") {
    cc.define("BAR_SUPPORTED", Some("1"));
}
```


## Header files

There are two places where you may need to expose C library's header (`.h`) files:

1. To C code in other `-sys` crates (optional).
2. To Rust code using your sys crate.

The first case is simpler. Make sure you have `links` in your `Cargo.toml`:

```toml
[package]
name = "foobar-sys"
links = "foobar"
```

Print `cargo:include=/path/to/include` with the directory where the library's own `.h` files are (`cargo:include` is not a special name, you can use [any `cargo:<name>=<value>`](https://doc.rust-lang.org/cargo/reference/build-scripts.html#the-links-manifest-key) to provide more information).

```rust
println!("cargo:include={}", absolute_path_to_include_dir.display());
```

If you need to make a relative path into absolute, use [`dunce::canonicalize()`](https://crates.io/crates/dunce), because `fs::canonicalize()` [is unusable](https://github.com/rust-lang/rust/issues/42869).

In your crate's documentation instruct others to read `DEP_<your links name>_INCLUDE` env variable if they need the headers (e.g. [libz](https://github.com/alexcrichton/libz-sys/blob/8eb6f9e8f45cef71f8a3fa94d849ac54a263c090/build.rs#L234) &rarr; [libpng](https://github.com/kornelski/rust-libpng-sys/blob/065b1b2a292a9bad613c2044e3357d6f46c8b77e/build.rs#L98-L99)):

```rust
cc.include(env::var_os("DEP_FOOBAR_INCLUDE").expect("Oops, DEP_FOOBAR_INCLUDE should have been set"));
```

### Bindgen

For Rust you will need to translate C headers into a Rust module with `extern "C" {}` declarations. This can be done automatically with [bindgen](https://github.com/rust-lang-nursery/rust-bindgen), but there are some things to consider.

Bindgen has option to [translate C `enum` to Rust `enum`](https://docs.rs/bindgen/0.36.0/bindgen/struct.Builder.html#method.rustified_enum). It's *nice* to have enums, but Rust has extra requirements: the enum *must* contain only valid values at all times. That's a guarantee in safe Rust. If the C side violates it, it will "poison" Rust code and cause to be badly miscompiled. If C says `enum Foo {A=1, B=2}`, but somewhere returns `(enum Foo)3`, it can't be a Rust enum.

### Stable ABI?

There's a question whether you run bindgen [once and ship that](https://github.com/kornelski/mozjpeg-sys/blob/master/src/lib.rs) file, or whether you run bindgen [every time](https://github.com/meh/rust-ffmpeg-sys/blob/9056485f7fe4a42e3a68d08761b5a7817339da23/build.rs#L907) the project is build. It depends on how incompatible different versions of the C library may be.

If the C library has a stable and portable ABI: new versions only add new functions and everything is backwards compatible, then you can pre-generate. It will be faster to build (no dependency on bindgen and clang), and you can even tweak the built file manually. Make sure it works for both 32 and 64-bit architectures (usually `#[repr(C)]` just works, but you'll need to disable generation of bindgen's layout tests, because they're architecture-specific).

If there are different, incompatible versions of the library in the wild, then you need to [use bindgen as a Rust library](https://docs.rs/bindgen) and run it from your `build.rs` to generate fresh bindings for every user. The generated file must be included somewhere in your crate's `lib.rs`. Use `include!(concat!(env!("OUT_DIR"),"/filename.rs"));`.

## Different major versions

If differences between major versions of the C library are small (e.g. only new functions added, or just a couple of struct fields changed), then you could try to automatically adapt to the version or use Cargo features to enable the new features (e.g. [mozjpeg-sys supports different ABI versions](https://github.com/kornelski/mozjpeg-sys/blob/384688f9c23e94ddeb353d414d45ede69768ec08/src/lib.rs#L115-L120), clang-sys has [features for LLVM versions](https://github.com/KyleMayes/clang-sys#supported-versions)).

If versions of the C library are totally different and incompatible, then either have separate crates (`foo1-sys` & `foo2-sys`) or at least use different [major](https://doc.rust-lang.org/cargo/reference/manifest.html#the-version-field) versions of your sys crate for different major versions of the C library, so that Cargo will know that they're not compatible.

## Cross-compilation

Rust can build executables and libraries for systems other than the one it's running on, e.g. build Linux programs on macOS, or build 32-bit libraries on a 64-bit OS.

Your `build.rs` program may be running on a different architecture than the one being compiled. This means that all your `size_of` checks and `#[cfg]`/[`cfg!()`](https://doc.rust-lang.org/std/macro.cfg.html) macros in `build.rs` *may* be for a [wrong machine](https://kazlauskas.me/entries/writing-proper-buildrs-scripts.html)! For querying target system or CPU use `CARGO_CFG_TARGET_OS`/`CARGO_CFG_TARGET_ARCH`/`CARGO_CFG_TARGET_POINTER_WIDTH` env vars instead (run `rustc --print cfg` for a full list). The only exception are `cfg(feature = "…")` checks, which are safe to use in cross-compilation.

`pkg-config` will automatically bail out when it detects cross-compilation (when env var `HOST` != `TARGET`). If you're searching for libraries on disk in other ways, also keep in mind that the host system may not be compatible with the target being built.

## Linking surprises

Write [tests](https://doc.rust-lang.org/book/second-edition/ch11-03-test-organization.html) in your sys crate's `lib.rs` to reference as many C symbols as you can. Linkers often work "lazily" and won't notice any problems with the library unless it's actually used from Rust.

In external tests (in `tests/` directory) and other crates make sure to include `extern crate <your lib>_sys;`. The C library won't be linked unless it's used via `extern crate`, even if it's set as a dependency in `Cargo.toml`!


## Documentation

Have a good README (and `readme` key in `Cargo.toml`) with clearly stated requirements and configuration options (especially env vars).

However, don't bother documenting individual FFI functions in Rust. Sys crates by definition don't change behavior of the C library and don't add anything that isn't already in the C version, so for function-specific information send users to the original C documentation (e.g. [libc](https://docs.rs/libc) intentionally doesn't document any function).

If you want to make the library easier to use, it's better to spend effort on making a second crate with a higher-level interface.

