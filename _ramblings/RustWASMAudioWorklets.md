---
layout: post
title: 'WebAudio DSP in Web Assembly from Rust'
toc: true
modified_date: false
order: 4
---

{:toc}


This article details an experiment running audio DSP in WASM, compiled from
Rust. The result of the exploration is a template repo for writing Rust crates
to perform DSP, compiling them to WASM, and some JS glue + utilities for calling
these WASM modules in Web Audio AudioWorkletProcessors. These processors are
packaged in a contained `npm` package.

On the Rust side, the goal was roughly to be able to define the main
AudioWorkletProcessor `process` function, so the JS side can just delegate this
to WASM (we roughly get there, with some tweaks that handle memory allocation
and pointers to buffers on the WASM linear memory).

The culmination of the experiment is [this template repo](https://github.com/JamieBeverley/wasm-audio-worklet/tree/706e547ab45a436c1d611a650ab165fbb7417799), which the below details.


## Rust -> WASM
At the time of writing (~Nov. 2024) `wasm-pack` JS bindings do not work in an
AudioWorklet context due to some JS APIs not being available there that
`wasm-pack` depends on (`fetch`, `URL`,  etc...), and there is no compatible
build target.

So instead we use `cargo` with `--target wasm32-unkown-unkown` and need to
handle JS bindings ourselves (darn!).

## Loading WASM in an AudioWorkletNode
To load the `wasm` module then, we can't rely on the `init` or `initSync` utils
we would otherwise get from `wasm-pack`. Instead, we `fetch` the `wasm` module
in our main JS thread, decoded it as an array buffer (`respons.arrayBuffer()`),
and then pass it over to the AudioWorklet context via 
`AudioWorkletNode.port.postMessage` (eg. [here](https://github.com/JamieBeverley/wasm-audio-worklet/blob/706e547ab45a436c1d611a650ab165fbb7417799/src/index.ts#L49-L51)).

```typescript
const response = await fetch(pathToWasm);
const wasmBytres = await response.arrayBuffer();
// ...
myAudioWorkletNode.port.postMessage({type: 'init-wasm', wasmBytes});
```

On the receiving side, our AudioWorkletNode needs to handle this post message
and `await WebAssembly.instantiate(message.wasmArrayBuffer)` to load the `wasm`
module (e.g. [here](https://github.com/JamieBeverley/wasm-audio-worklet/blob/706e547ab45a436c1d611a650ab165fbb7417799/public/worklets.js#L130-L139)).
The postmessage handler might look something like:

```typescript
async initWasm(data) {
    this._wasm = (await WebAssembly.instantiate(
        data.wasmBytes,
    )).instance.exports;
    // We'll get to this part later...
    this.alloc_memory();
    // Tell main JS thread the wasm module loaded
    this.port.postMessage({ type: "init-wasm-complete" });
}
```

## Handling Memory
Our WASM memory needs to size appropriately for the amount of audio channels and
parameters our AudioWorklet supports.

E.g. if we're creating a mono effects plugin we likely need (roughly):
```
+ 128 * 4-byte floats (=1 audio block) allocated for the input buffer to our `AudioWorkletProcessor`'s `process`
  function
+ 128 * 4-byte floats allocated for the output buffer
+ 128 * 4-byte floats * Number of Audio-Rate Params
+ Number of K-Rate Params * 4-bytes
```
(WebAudio block size is currently hard-configured to 128, though this may change
or be left to clients to determine based on available hardware in the future)

How I approached this:
### Allocating Memory From Rust

Created a publicly exposed `alloc` method in Rust to allocate memory (with
thanks/credit to [the-drunk-coder/wasm-loop-player](https://github.com/the-drunk-coder/wasm-loop-player/blob/master/src/lib.rs)):

```rust
#[no_mangle]
pub extern "C" fn alloc(size: usize) -> *mut f32 {

    // initialize a vec32
    let vec: Vec<f32> = vec![0.0; size];
    // convert heap-allocated array to just the pointer of the beginning of that
    // array on the heap
    Box::into_raw(
        // convert vec 32 to a heap-allocated array of f32 values
        vec.into_boxed_slice(),
    ) as *mut f32
}
```
### Coupling a WASM Memory Pointer w/ a Float32Array
I created a tiny JS class that runs in the AudioWorklet context for allocating
buffers (`Float32Array`s) that both JS and Rust can read/write to.

This is basically just a class to (intentionally) tightly couple 2 things together:
- `this.ptr` which represents a pointer to a location in the WASM module's
  linear memory model where the array buffer starts
- the `Float32Array` which contains the values (and also the length of the
  buffer)
```javascript
class WasmBuffer {
    constructor(size, wasm) {
        this.ptr = wasm.alloc(size);
        this.buffer = new Float32Array(
            wasm.memory.buffer,
            this.ptr,
            size
        );
    }
}
```

### Growing WASM Memory Using a Collection Helper Class
I created one more helper class called `WasmMemory` to behave as a collection of
`WasmBuffer`s that handles growing/shrinking the `wasm` memory's size as needed.

In brief, it exposes one main public method: `alloc`, which allows allocating a
new buffer on a wasm modules linear-memory, and re-grows the
memory as needed (see below)

Why is this needed? If we allocate a new buffer, and the size of that buffer
exceeds the prior size of the WebAssembly.Memory for our WASM module, 
_all previously allocated buffers will become read-only_. This will of course
be an issue when we need to write to these buffers.

Thus, we need to sense when memory has grown as a result of allocating and
re-allocate all previously allocated buffers.

(I know, yuck, but this should be
pretty infrequent and we can probably make educated bets about how to right-size
memory to begin with based on our DSP patch).

The full class can be found is [here](https://github.com/JamieBeverley/wasm-audio-worklet/blob/706e547ab45a436c1d611a650ab165fbb7417799/public/worklets.js#L17)
but in brief:
```typescript
class WasmMemory {
    constructor(wasm) {
        this.wasm = wasm
        this.buffers = {}; // {[k:string]: WasmBuffer}
    }

    ...
    
    alloc(name, size) {
        const beforeSize = this.getWasmBufferLength();

        // If wasm memory grows, we need to re-create old views on the memory.
        // We need to store the size of that memory before grow happens because
        // if a grow does occur, these buffers will have 0 length.
        const bufferLengths = this.getBufferLengths();

        this.buffers[name] = new WasmBuffer(size, this.wasm);
        const afterSize = this.getWasmBufferLength();
        if (beforeSize !== afterSize) {
            // When we grow all prior buffers will be transfered and readonly.
            // So we need to re-create views on the regrown wasm memory.
            // (grow sparingly!)
            const rebuildBufferNames = Object
                .keys(this.buffers)
                .filter(bufferName => name !== bufferName)

            rebuildBufferNames
                .forEach(bufferName => {
                    this.buffers[bufferName] = new WasmBuffer(
                        bufferLengths[bufferName],
                        this.wasm,
                    )
                });
        }
    }
}
```

## Invoking Rust/WASM DSP from `AudioWorklet.process(inputs, outputs, params)`

This distills to 3 things in the simplest case:
- Set the values of the `Float32Array` of the input buffer (assuming mono) to
  the values of the `inputs` parameter
    - (again, we should also have a reference to the pointer that `alloc` (from
    Rust) gave us, referencing the start location of this Float32Array)
- Call the `wasm` module's DSP function, passing the pointer to the input
  `Float32Array`, and the pointer to our output `Float32Array`
    - (this should probably set some values in the output array)
- Set the values of the `output` parameter to those of the output `Float32Array`


In pseudo code (or concrete implementation [here](https://github.com/JamieBeverley/wasm-audio-worklet/blob/706e547ab45a436c1d611a650ab165fbb7417799/public/worklets.js#L191))
```typescript
process(inputs, outputs, params){
    // (assuming this.inBuffer and this.outBuffer are WasmBuffer instances from
    // above)
    this.inBuffer.buffer.set(inputs[0][0])
    this.wasm.process(this.inBuffer.ptr, this.outBuffer.ptr)
    outputs[0][0].set(this.outBuffer.buffer)
    return true
}
```

(this speaks to the motivation for tightly-coupling a `Float32Array` with the
wasm pointer in the `WasmBuffer` class above - anywhere we're passing a buffer
of values to `wasm` we need both the `Float32Array` and a pointer to where it
starts in the WASM linear memory model)

## Structuring the Repo

### `npm` package
I wanted the root of this repo to be an installable `npm` package that exports
a AudioWorkletNode and handles the messy bits regarding fetching public `.wasm`
files and passing them to the AudioWorklet context.

There are at least 2 static assets we need to include as part of this `npm`
package:
- all `.wasm` files
- our `worklets.js` file containing the AudioWorkletProcesor sub-class(es)

I keep both of these in a folder called `public` and then configure `vite` to
ensure these are copied over. `vite.config.js`:
```javascript
export default defineConfig({
    ...
    build: {
        ...
        copyPublicDir: true,
        assetsDir: "public",
    },
});
```

The details of the public exports of this npm package can be seen in full [here](https://github.com/JamieBeverley/wasm-audio-worklet/blob/706e547ab45a436c1d611a650ab165fbb7417799/src/index.ts)
but the crux of it is basically that a new AudioWorkletNode that runs DSP in
wasm should only require specifying a few things, which I've abstracted in a
(terribly-java-named) class called `WorkletModuleFactory`

For example:
```typescript
const BitCrusher = new WorkletModuleFactory(
    // The path to `public/worklets.js` containing AudioWorkletProcessor JS 
    // subclass
    new URL('./worklets.js', import.meta.url).href,
    // The name of the processor (ie. the same as where you call
    // registerProcessor('some name',...) in your worklet.js file
    "BitCrush",
    // The path to the wasm module (compiled from rust)
    new URL('./bit_crusher.wasm', import.meta.url).href,
    // A timeout, how long to wait for the wasm module to load before erroring
    // out.
    5000,
    ...
)
```

### The Rust Crate(s)

I chose to structure this as 1 crate (and thus 1 wasm) per
`AudioWorkletProcessor`.

`./crates/cargo.toml` aggregates a few sub-crates eg:
```toml
[workspace]
members = [
    "bit_crusher", "wave_shaper", "common",
]
```
Each sub member imports common utilities from the `common` crate (Eg. such as
the `alloc` function which is always needed from JS)

eg: `./crates/bit_crusher/cargo.toml:
```
[package]
name = "bit_crusher"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
common = { path = "../common" }
```
Building & installing these `.wasm` files then just involves:
```sh
# build
cd crates && cargo build --target wasm32-unknown-unknown --release
# install
cp crates/target/wasm32-unknown-unknown/release/*.wasm public/
```

## Attempts at Profiling

Part of the motivation for moving the bulk of the work done in
`AudioWorkletProcessor.process` to `wasm` was to see if we'd achieve
performance gains by doing so.

Profiling this was a bit tricky, a few things were considered/attempted (
although to be honest, this wasn't terribly well conceived).

### Precise Profiling
Ideally we could author the equivalent DSP algorithm in JS and Rust/wasm and
use the `Performance` API to determine which implementation was faster in an
`AudioWorkletProcessor` context.

Unfortunately the `Performance` API is not available in the
AudioWorkletProcessor.

`new Date()` APIs are also not fine-grained enough for comparing performance of
processing an audio block. If our audio sample rate is 41kHz, and our dsp is
_just_ keeping up with this (without audio drop outs), we're processing
`41000/128=~320` audio blocks per second. So each block takes less than or equal
to 1/320s = ~3.125ms. In practice, these are often sub 1ms but the `Date` API
only goes up to the millisecond (at the very least, profiling in `ms` would be
pretty inaccurate).

### Stress-test Profiling
Instead of precisely profiling the speed of processing 1 audio block, we can
instead see how the browser performs if we try to run a ton of
`AudioWorkletProcessor`s at once and compare whether JS or WASM implementations
differ.

I attempted this with a DSP/`process` method that simply plays back a loaded
sound buffer (e.g. a drum loop `.wav`).

After attempting to run about 1000 instances of the same `wasm` processor
concurrently I started seeing errors:

```
WebAssembly.Memory failed to reserve a large virtual memory region. This may be due to low configured virtual memory limits on this system.
```

This isn't all that suprising: for each of the 1000 nodes, I was allocating:
- 2 buffers the size of an audio block for input/output
- the full audio buffer we're playing back (~4MB in my case)

Empirically, it seems the JS implementation started to perform better than my
wasm implementation for simple audio file playback at ~900 concurrent nodes
(wasm started to have audio dropouts, JS implementation seemed more stable).

This isn't all that interesting or surprising; in the `wasm` implementation we
have extra overhead copying `inputs` to a `Float32Array` that is accessible to
our `wasm` instance, and then copying the `wasm`'s `outputs` back over to
JS-land.

Alas this wasn't a terribly well-conceived test and my interest in going deeper
dropped off.

### Startup Time
Fetching the `.wasm` module, initializing memory, and instantiating
a WebAssembly module clearly incurs some overhead that we wouldn't have if just
working in JS.

I didn't profile this because its pretty minimal and manageable. In my case
anyway, my `.wasm` files were pretty small so startup has no perceivable delay.

We should take care to cache `.wasm` bytes (the `ArrayBuffer`s we decode after
fetching a `.wasm` file) so we don't re-fetch and re-decode each time we
instantiate a new Node.

This is trivial to do by keeping a reference to each `Promise` we create for
each `.wasm` we try to load and decode (see [here](https://github.com/JamieBeverley/wasm-audio-worklet/blob/706e547ab45a436c1d611a650ab165fbb7417799/src/index.ts#L44)).
