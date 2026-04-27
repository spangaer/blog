# CXX RS shared library

## Setting the scene

This covers building a Rust shared library with a clean C++ ABI using https://cxx.rs.

It follows up on
https://github.com/dtolnay/cxx/issues/1153#issuecomment-2521433004
which refers to [this comment][cxx-issue-880-comment], suggesting the following CMake line:

```cmake
add_executable(my-examples-cpp src/main.cpp ${RUST_TARGET}/cxxbridge/my-cxx/src/lib.rs.cc)
```

That approach, compiling bridge-generated files on the consumer side, turned out to be
nonsensical. Here's why:

_cxx.rs_ C symbols are unstable because they embed the cxx version:

```
my$crate$does$something$cxxbridge1$192$...
```

The `192` segment is the _cxx.rs_ minor version, so it will change with every dependency update.
To avoid that instability, the ABI must be moved to C++ symbols.

In addition, cxx.rs's _runtime_ (`cxx.cc`) is compiled as part of the cxx.rs crate build.
Parts of it handle `CxxString` interaction.
If your ABI exposes Rust types like `::rust::String`, cross-language compatibility functions like
`explicit operator std::string_view() const` are also implemented by `cxx.cc` and should likely
be exported by your shared library.

A [pending PR](https://github.com/dtolnay/cxx/pull/1025) intends to deliver this. Until
it lands, if you don't want a custom build, an alternate strategy is required.

## Goals

Our goal is to export all C++ symbols backed by our Rust-built library.
This includes symbols from the `cxx.cc` runtime, generated bridge `.cc` files, and hand-written
`.cc` files.

## All OSes - `build.rs`

The following `build.rs` snippet is shared across platforms:

```rust
let out_dir = env::var("OUT_DIR").unwrap();
let cxx_lib_name = format!("{}-bridge", env::var("CARGO_PKG_NAME").unwrap());

// setting the bridges generates the .h and .cc files;
// suppress default cargo metadata so we can link with +whole-archive
cxx_build::bridges(bridges)
    .cargo_metadata(false)
    .std("c++20")
    .compile(&cxx_lib_name); // compile .cc files

// force the linker to include all C++ symbols from the static archive,
// even if nothing in the Rust side references them
println!("cargo::rustc-link-search=native={}", out_dir);
println!(
    "cargo::rustc-link-lib=static:+whole-archive={}",
    cxx_lib_name
);
```

## Linux - The easy part

GNU linker version scripts come to the rescue here:

`build.rs`

```rust
// GNU ld (Linux, MinGW): version script handles both upstream and bridge symbols via glob
// patterns targeting my::crate::* and rust::cxxbridge1::*, be it in mangled form.
let exports = MANIFEST_DIR.join("exports/exports-lin.map");
println!(
    "cargo::rustc-link-arg=-Wl,--version-script={}",
    exports.display()
);
```

`exports/exports-lin.map`

```
{
  global:
    #extern "C++" {
    #  rust::cxxbridge1::*;
    #  my::crate::*;
    #};
    # mangled symbols because the extern "C++" glob doesn't match all
    # (e.g. conversion operators like String::operator std::string)
    *rust10cxxbridge1*;
    *my5crate*;
  local: *;
};
```

Normally the linker supports exporting unmangled symbols via globbing,
but that seems to have bugs when it comes to C++ operators, which we get past by
working in the _Itanium ABI_ mangled domain.

All object archives get processed by the linker at link time, so it can easily select what it
needs. A bit sad that the other platforms can't do the same.

## Windows and macOS

For Windows and macOS, this requires extra effort. We'll scan through a compiled object
archive and list all symbols (that match a certain infix), so we can use that list for
export.

For cross-platform consistency, I developed bot-assisted bash scripts for both OSes.

### The scripts

#### `extract-symbols-{{os}}`

The gist of it is:

1. Expect two arguments: a grep pattern and a static library path.
2. For Windows, a section to locate a suitable `dumpbin`.
3. List candidate symbols from the archive using platform tooling
   (`nm -g` on macOS, `dumpbin //LINKERMEMBER` on Windows).
4. Normalize each output line down to just the mangled symbol name.
5. Keep only symbols matching the provided pattern.
6. Sort in stable ASCII order and deduplicate (`LC_ALL=C sort -u`).
7. Output the final symbol list, one symbol per line, ready to feed into export definitions.

#### `gen-cxx-exports-{{os}}`

The gist of it is:

1. Search the build tree for the platform-specific cxxbridge library
  (`libcxxbridge1.a` on macOS, `cxxbridge1.lib` on Windows).
2. Call the matching extractor script with the platform-specific symbol pattern.
   - win: `@cxxbridge1@rust@@`
   - mac: `rust10cxxbridge1`
3. Write all symbol lines to file `cxx-exports-{{os}}.txt`

The script is invoked with the following steps:

```bash
# clean cxx.rs build artifacts
./exports/cxx-clean
# remove stale exports before rebuild
rm -f ./exports/cxx-exports-{{os}}.txt
# build to generate symbols
cargo build
# extract symbols from lib
./exports/gen-cxx-exports-{{os}}
# rebuild with exports to verify linking
cargo build
```

The output files are generated _once_ (whenever cxx.rs is updated) and committed to source control.
Recurring builds can produce variant _cxxbridge1_ libraries in your target directory, which
could lead to unpredictable, thus unwanted, results.

The scripts above only _list_ the `cxx.cc` symbols. The `build.rs` below ties listing and
exporting together.

### `build.rs`

> **Note:** the MSVCRT support from [this comment][cxx-issue-880-comment] comes along here too.

```rust
// The code below handles build setup variations per OS/compiler.
// One thing shared between OSes is the need for exporting the cxx.rs C++ symbols.
// While the goal is universal, the way to do so is highly dependent on the toolset.

// Note: libc and libstdc++ (or respective OS variants) are dynamically linked on all supported
// platforms.

if env::var("TARGET").is_ok_and(|s| s.contains("windows-msvc")) {
    // MSVC compiler suite
    if env::var("CFLAGS").is_ok_and(|s| s.contains("/MDd")) {
        // debug runtime flag is set

        // don't link the default CRT
        println!("cargo::rustc-link-arg=/nodefaultlib:msvcrt");
        // link the debug CRT instead
        println!("cargo::rustc-link-arg=/defaultlib:msvcrtd");
    }

    // MSVC link.exe (Windows): export all cxx C++ symbols

    let emit_export = |symbol: &str| {
        // Use /EXPORT: (additive, unlike /DEF which overrides);
        println!("cargo::rustc-link-arg=/EXPORT:{}", symbol);
    };

    // upstream cxxbridge1 symbols generated by gen-cxx-exports-win
    let exports_file = MANIFEST_DIR.join("exports/cxx-exports-win.txt");
    println!("cargo::rerun-if-changed={}", exports_file.display());
    if exports_file.exists() {
        let contents = std::fs::read_to_string(&exports_file).unwrap();
        for line in contents.lines() {
            let line = line.trim();
            if !line.is_empty() && !line.starts_with('#') {
                emit_export(line);
            }
        }
    }

    // bridge lib symbols extracted at build time
    let bridge_lib = Path::new(&out_dir).join(format!("{}.lib", cxx_lib_name));
    let script = MANIFEST_DIR.join("exports/extract-symbols-win");
    for symbol in extract_symbols(&script, "crate@my", &bridge_lib) {
        emit_export(&symbol);
    }
} else if env::var("TARGET").is_ok_and(|s| s.contains("apple")) {
    // macOS ld

    // set install name to @rpath so the dylib is relocatable
    let lib_name = format!(
        "lib{}.dylib",
        env::var("CARGO_PKG_NAME").unwrap().replace('-', "_")
    );
    println!(
        "cargo::rustc-link-arg=-Wl,-install_name,@rpath/{}",
        lib_name
    );

    // export all cxx C++ symbols
    let emit_export = |symbol: &str| {
        // Use -exported_symbol: (additive, unlike -exported_symbols_list which overrides);
        println!("cargo::rustc-link-arg=-Wl,-exported_symbol,{}", symbol);
    };

    // upstream cxxbridge1 symbols generated by gen-cxx-exports-mac
    let exports_file = MANIFEST_DIR.join("exports/cxx-exports-mac.txt");
    println!("cargo::rerun-if-changed={}", exports_file.display());
    if exports_file.exists() {
        let contents = std::fs::read_to_string(&exports_file).unwrap();
        for line in contents.lines() {
            let line = line.trim();
            if !line.is_empty() && !line.starts_with('#') {
                emit_export(line);
            }
        }
    }

    // bridge lib symbols extracted at build time
    let bridge_lib = Path::new(&out_dir).join(format!("lib{}.a", cxx_lib_name));
    let script = MANIFEST_DIR.join("exports/extract-symbols-mac");
    for symbol in extract_symbols(&script, "my5crate", &bridge_lib) {
        emit_export(&symbol);
    }
} else {
    // GNU ld
}

...

/// extract mangled symbols from a static library using a platform-specific bash script
fn extract_symbols(script: &Path, pattern: &str, lib_path: &Path) -> Vec<String> {
      let mut cmd = if cfg!(windows) {
        let mut c = Command::new(r"C:\Program Files\Git\bin\bash.exe");
        c.arg(script);
        c
    } else {
        Command::new(script)
    };
  ...
}
```

In short:
1. Read `cxx.cc` symbols from the stored file.
2. Read your own `.cc` symbols from the binary in `OUT_DIR` using the same export script.
3. Emit all discovered symbols as OS-specific linker arguments.

# Wrap-up

It isn't pretty, but it works.

> _Don't let perfect get in the way of better!_
> (Or simply working in this case)

## Search terms

- cxx.rs shared library export C++ symbols
- cxx.rs dylib cdylib export cxxbridge symbols
- Rust cxx bridge shared library Windows macOS Linux
- cxx.rs cxxbridge1 symbol version unstable ABI
- cargo build.rs export symbols whole-archive
- GNU ld version script cxx.rs mangled symbols
- MSVC /EXPORT cxx.rs bridge symbols build.rs
- macOS -exported_symbol cxx Rust dylib
- cxx.cc runtime export CxxString rust::String
- cxx.rs pending PR 1025 workaround shared library

[cxx-issue-880-comment]: https://github.com/dtolnay/cxx/issues/880#issuecomment-2521375384
