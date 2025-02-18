# Introduction

Welcome to the Rust-GPU dev guide! This documentation is meant for documenting
how to use and develop on Rust-GPU.

## Getting started

1. Clone the repository.

    ```shell
    git clone --recurse-submodules https://github.com/EmbarkStudios.com/rust-gpu
    ```

1. Install the prerequisites using the provided setup script. From the root of the project, run:

    MacOS, Linux:

    ```shell
    sh setup.sh
    ```

    Windows:

    ```shell
    setup.bat
    ```

    The setup script installs nightly Rust (required for now, see [#78](https://github.com/EmbarkStudios/rust-gpu/issues/78) for tracking issue).

1. **optional** Install [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools#downloads) and add it to your `PATH`. You can skip this step if you just want to run examples with the defaults. See [Using installed SPIRV-Tools](#using-installed-spirv-tools) if you decide to go with this option.

1. Next, look at the [examples](examples) folder. There are two projects here: [examples/example-shader](examples/example-shader) and [examples/example-runner](examples/example-runner). The example-shader project is a "GPU crate", one that will be compiled to a SPIR-V module. The example-runner project is a normal, CPU crate that uses vulkan to consume the example-shader SPIR-V module to display a shader.

    Run the example:

    ```shell
    cargo run --bin example-runner
    ```

    This will build `rustc_codegen_spirv`, the compiler, then use that compiler to build `example-shader` into a SPIR-V module, then finally, build a vulkan sample app (taken from [ash's examples](https://github.com/MaikKlein/ash/blob/master/examples/src/bin/triangle.rs)) using the built SPIR-V module to display the shader in a window.

    All of this is orchestrated by the [spirv-builder](spirv-builder) crate, which is used in example-runner's `build.rs` file. Please look at that file, as well as both example projects in general, to see how to set up your own shaders!

Be aware that this project is in a very early phase - if the above doesn't work, please [file an issue](https://github.com/EmbarkStudios/rust-gpu/issues)!

## Getting started, for power users who don't want to use spirv-builder

If you would like to build the compiler, `rustc_codegen_spirv` is the relevant folder. Install the prerequisites, as above, then, `cd rustc_codegen_spirv && cargo build`. This produces an .so file, located at `./target/debug/librustc_codegen_spirv.so` (or `.dll`/`.dylib` depending on your platform).

This file is a dynamically loaded backend for rustc - you may tell rustc to use it as a backend through the `-Z codegen-backend=...` flag. To pass this to rustc through cargo, set the environment variable `RUSTFLAGS="-Z codegen-backend=$PATH_TO_FILE"`.

Then, when building a GPU crate, we need to configure some flags when we call cargo. First, we need to build libcore ourselves - we obviously have no SPIR-V libcore installed on our system! Use the flag `-Z build-std=core`. Then, we need to tell rustc to generate SPIR-V instead of x86 code: `--target spirv-unknown-unknown`.

Overall, building your own SPIR-V crate looks like:

```shell
export RUSTFLAGS="-Zcodegen-backend=$THIS_REPO/target/debug/librustc_codegen_spirv.so"
cargo build -Z build-std=core --target spirv-unknown-unknown --release
```

(with an appropriate path for `$THIS_REPO`, and replacing `export` with `set` if you're on windows as well as the proper dll name)

This will produce a `target/spirv-unknown-unknown/release/crate_name.spv` file.

To create a GPU crate, look at the [examples/example-shader](examples/example-shader) crate. In short, reference the `spirv-std` crate, and use intrinsics defined there to create your shader.

This is all a little convoluted, hence the [spirv-builder](spirv-builder) crate handles a lot of this.

## Using installed SPIRV-Tools

By default, all of the crates and examples in this repo will compile the `spirv-tools-sys` crate, including a lot of C++ code from [SPIRV-Tools](https://github.com/EmbarkStudios/SPIRV-Tools). If you don't want to build the C++ code because you already have [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools#downloads) installed, or just don't want to spend more time compiling, you can build/run the crate with the `use-installed-tools` feature.

```shell
cargo run \
    --manifest-path examples/example-runner/Cargo.toml \
    --features use-installed-tools \
    --no-default-features
```

You should see `warning: use-installed-tools feature on, skipping compilation of C++ code` during the compilation, but otherwise the build will function just the same as if you compiled the C++ code, with the exception that it will fail if you don't have SPIRV-Tools installed correctly.
