# Game Of Life

Should be run before hand, to download and install the Emscripten SDK
```
git clone https://github.com/juj/emsdk.git
cd emsdk
./emsdk install sdk-incoming-64bit binaryen-master-64bit
./emsdk activate sdk-incoming-64bit binaryen-master-64bit
```

Then set up your environment for the current console:
`source ./emsdk_env.sh`
You should now be able to run the `emcc` command and it will report no files specified.

# Examine the source
First open `lyff/lyff.c` it should look something like this:
```
// Steps through one iteration of Conway's Game of Life. Returns the number of now alive cells, or
// -1 if no cells changed this iteration: i.e., stable game.
int board_step() {
  int total_alive = 0;
  int change = 0;

  // place output in A/B board
  byte *next = (byte *) &boardA;
  if (board == next) {
    next = (byte *) &boardB;
  }
  clear_board_ref(next);
...
```

You will notice some functions that will give us a simple implementation of the game.

# Build the code
In the `lyff` directory run:
```
emcc \
  -s WASM=1 -s ONLY_MY_CODE=1 -s EXPORTED_FUNCTIONS="['_board_init','_board_ref','_board_step']" \
  -o output.js *.c
  ```

  This will tell emcc(our compiler) what to compile and where to output. You should see a `output.wasm` file now.

  # HTML
Create a `index.html` file as a sibling to `lyff.c` and `output.wasm`
We are using a basic HTML outline:
```
<!DOCTYPE html>
<html>
<head>
<script>

// Add code inside the script tag.

</script>
</head>
<body>

<canvas id="canvas" style="image-rendering: pixelated; border: 2px solid blue;"></canvas>

</body>
</html>
```

Since browsers that support Web Assembly are new they also support ES6 features. We can use `async`, `await`, and `Promises`. Insert a standard Web Assembly loader inside the script tag:
```
  async function createWebAssembly(path, importObject) {
    const bytes = await window.fetch(path).then(x => x.arrayBuffer());
    return WebAssembly.instantiate(bytes, importObject);
  }
```

# The Environment
We also need to specify an `importObject`: this provides the environment Web Assembly runs in as well as any other parameters to instantiation. At a minimum, you need to provide an object like this—add it at the end your script tag:
```
  const memory = new WebAssembly.Memory({initial: 256, maximum: 256});
  const env = {
    'abortStackOverflow': _ => { throw new Error('overflow'); },
    'table': new WebAssembly.Table({initial: 0, maximum: 0, element: 'anyfunc'}),
    'tableBase': 0,
    'memory': memory,
    'memoryBase': 1024,
    'STACKTOP': 0,
    'STACK_MAX': memory.buffer.byteLength,
  };
  const importObject = {env};
```

Finally putting it all together we need to pass the `importObject` to the `createWebAssembly` function. Add this code at the end of the script tag:
```
  createWebAssembly('output.wasm', importObject).then(wa => {
    const exports = wa.instance.exports;
    console.info('got exports', exports);
    exports._board_init();  // setup lyff board

    // TODO: interact with lyff board

  }).catch(err => console.warn('err loading wasm', err));
```

Move into the `lyff` directory and start a server `python -m SimpleHTTPServer` In the developer console you should see a log of the exported methods

# Interact
At the `TODO: interact` you added on the previous page (inside the `createWebAssembly` callback), add this method and code:
```
  createWebAssembly('output.wasm', importObject).then(wa => {
    const exports = wa.instance.exports;
    console.info('got exports', exports);
    exports._board_init();  // setup lyff board

    function getBoardBuffer() {
      return new Uint8Array(memory.buffer, exports._board_ref());
    }
    function draw() {
      const buffer = getBoardBuffer();
      // TODO: render buffer
    }

    draw();

    // TODO: step through board

  }).catch(err => console.warn('err loading wasm', err));
```
Replace the `draw` method from above with this code:
```
  function draw() {
    const buffer = getBoardBuffer();

    const dim = 100;  // nb. fixed size
    canvas.width = canvas.height = dim + 2;
    canvas.style.width = canvas.style.height = `${dim*5}px`;
    const data = new ImageData(canvas.width, canvas.height);

    for (var x = 1; x <= dim; ++x) {
      for (var y = 1; y <= dim; ++y) {
        var pos = (y * (dim + 2)) + x;
        var i = (pos / 8) << 0;
        var off = 1 << (pos % 8);

        var alive = (buffer[i] & off);
        if (!alive) { continue; }

        const doff = (y * canvas.width + x) * 4;
        data.data[doff+0] = 255;
        data.data[doff+3] = 255;
      }
    }

    canvas.getContext('2d').putImageData(data, 0, 0)
  }
```
This reads the format encoded inside the buffer and renders it to a canvas.

# Step through the game
We can provide a way for the user to step through the simulation. We are going to make it so on click on the canvas it steps through the game one frame at a time. After the `draw` function is called replace `TODO: step through board` with: 
```
  canvas.onclick = ev => {
      exports._board_step();
      draw();
    };
```
Finally save and reoload. Every time you click you will call `board_step` and draw a new frame!

---
For more information visit [Google Web Assembly](https://codelabs.developers.google.com/codelabs/web-assembly-intro/index.html?index=..%2F..%2Findex#0)