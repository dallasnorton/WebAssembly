# Rust

Prerequisites :
XCode,
Rust
```
cd ~/emsdk-portable
./emsdk update
./emsdk install sdk-incoming-64bit
./emsdk activate sdk-incoming-64bit
```

To enter an Emscripten compiler environment:
```
$ source ./emsdk_env.sh
```

Install Rust

```
curl https://sh.rustup.rs -sSf | sh
```

```
source $HOME/.cargo/env
```

then we add the compile target to rust via rustup

```
rustup target add wasm32-unknown-emscripten
```

Now `emcc -v` should report something like

```
emcc (Emscripten gcc/clang-like replacement + linker emulating GNU ld) 1.37.34
clang version 5.0.0  (emscripten 1.37.34 : 1.37.34)
Target: x86_64-apple-darwin17.4.0
Thread model: posix
InstalledDir: /Users/dallasnortontemp/Code/Playground/WebAssembly/emsdk/clang/e1.37.34_64bit
INFO:root:(Emscripten: Running sanity checks)
```

Lastly

```
cargo init wasm-demo --bin
rustup override set nightly
```

# First Experiment

Insert into `src/main.rs`

```
#[derive(Debug)]
enum Direction { North, South, East, West }

fn is_north(dir: Direction) -> bool {
    match dir {
        Direction::North => true,
        _ => false,
    }
}

fn main() {
    let points = Direction::South;
    println!("{:?}", points);
    let compass = is_north(points);
    println!("{}", compass);
}
```

```
cd wasm-demo/src
cargo run
```

Will ouput 
```
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
     Running `/Users/dallasnortontemp/Code/Playground/WebAssembly/WebAssembly/rust/wasm-demo/target/debug/wasm-demo`
South
false
```

```
cargo build --target=wasm32-unknown-emscripten --release
```

Will build lots of files but the ones we are most interested are the `.wasm` and `.js` within the `target/wasm32-unknown-emscripten/release/deps/` directory.
We do get a JS file which seems counterintuitive but it's because it provides some of the glue code to fetch, initialize, and configuring it.

Now we need a webpage to import these files and use them. Create `site/index.html`

```
<html>
    <head>
        <script>
            // This is read and used by `site.js`
            var Module = {
                wasmBinaryFile: "site.wasm"
            }
        </script>
        <script src="site.js"></script>
    </head>
    <body></body>
</html>
```

Now we need to get the generated `target/` into the `site/` folder. We can use `make`.

```
SHELL := /bin/bash

all:
	cargo build --target=wasm32-unknown-emscripten --release
	mkdir -p site
	find target/wasm32-unknown-emscripten/release/deps -type f -name "*.wasm" | xargs -I {} cp {} site/site.wasm
	find target/wasm32-unknown-emscripten/release/deps -type f ! -name "*.asm.js" -name "*.js" | xargs -I {} cp {} site/site.js
```

Then run `make`

We can see the results with a simple server running `python -m SimpleHTTPServer`

You should see in the browser console:

```
trying binaryen method: native-wasm
asynchronously preparing wasm
binaryen method succeeded.
The character encoding of the HTML document was not declared. The document will render with garbled text in some browser configurations if the document contains characters from outside the US-ASCII range. The character encoding of the page must be declared in the document or in the transfer protocol.
South
false
```