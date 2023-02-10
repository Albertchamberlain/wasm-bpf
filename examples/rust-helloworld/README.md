# Example: Use Rust to write user-space program

## Create a Rust project 
```console
rustup target add wasm32-wasi
cargo new rust-helloworld
```

## Add `wasm-bpf-binding` as a dependency

```toml
wasm-bpf-binding = { path = "wasm-bpf-binding"}
```

This package provides bindings to functions that wasm-bpf exposed to guest programs.

## Add `wit-bindgen-guest-rust` as a dependency and patch it

```toml
[dependencies]
wit-bindgen-guest-rust = { git = "https://github.com/bytecodealliance/wit-bindgen", version = "0.3.0" }

[patch.crates-io]
wit-component = {git = "https://github.com/bytecodealliance/wasm-tools", version = "0.5.0", rev = "9640d187a73a516c42b532cf2a10ba5403df5946"}
wit-parser = {git = "https://github.com/bytecodealliance/wasm-tools", version = "0.5.0", rev = "9640d187a73a516c42b532cf2a10ba5403df5946"}
```

This package supports generating bindings for rust guest program with wit files. You don't have to run `wit-bindgen` manually.


## Generate `wit` filee using `btf2wit`

- Due to the restrictions on identifiers of WIT, you may encounter a lot of issues converting btf to wit.
- `wit-bindgen` generates strange import symbols for function whose name contains `-` (e.g, `wasm-bpf-load-object` will have an import name `wasm-bpf-load-object` but with name ``wasm_bpf_load_object``  exposed to the guest)

```console
gcc -g import.c -c -o import.o
pahole -J import.o
btf2wit import.o -o import.wit
```

## Put `wit` files under the `/wit` directory adjenct to `Cargo.toml`

Directory tree should be like:
```
Cargo.toml
src
| - main.rs
wit
| - host.wit
| - xxx.wit

....
```

`wit-bindgen-guest-rust` will generate bindings for each file in the `wit` directory.

## Add `#![no_main]` attribute for `main.rs` and modify the `main` function

To adapt `wasm-bpf`, the entry point for the wasm module we wrote should be a function called `__main_argc_argv` with signature `(u32,i32)->i32`.

So modify `main` function to:

```rust
#[export_name = "__main_argc_argv"]
fn main(_env_json: u32, _str_len: i32) -> i32 {

    return 0;
}
```

## Write your program to load the bpf program and attach it

Please refer to [src/main.rs](src/main.rs).

## Note

- Strings (e.g `&str`) are **NOT** zero-terminated. Be care when pass a pointer to foreign functions.
- Functions that be will be called by foreign code should have a signature `extern "C"` to ensure ABI.