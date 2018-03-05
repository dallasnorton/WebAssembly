# Hello World

Prerequisites :
CMake: `brew install cmake`
XCode

Precompiled toolchain to compile C/C++ into WebAssembly
```
$ git clone https://github.com/juj/emsdk.git
$ cd emsdk
$ ./emsdk install latest
$ ./emsdk activate latest
```

To enter an Emscripten compiler environment:
```
$ source ./emsdk_env.sh --build=Release
```

Now we create a C file in a hello directory,
```
$ mkdir hello
$ cd hello
```

hello.c
```
#include <stdio.h>
int main(int argc, char ** argv) {
printf("Hello, world!\n");
}
```

We have to pass the linker flag -s WASM=1 to emcc (otherwise by default emcc will emit asm.js).

```
$ emcc hello.c -s WASM=1 -o hello.html
```

To view compiled files we can run a simple HTTP server

```
$ emrun --no_browser --port 8080 .
```

---
For more information visit [WebAssembly.org](http://webassembly.org/getting-started/developers-guide/)