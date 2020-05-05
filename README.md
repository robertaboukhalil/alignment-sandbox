# Alignment Sandbox

An interactive web tool for exploring the sequence alignment algorithms Smith-Waterman and Needleman-Wunsch, powered by WebAssembly.

![Screenshot](https://alignment.sandbox.bio/assets/img/screenshot.png)


## How to use Alignment Sandbox?

You can use the hosted version at [alignment.sandbox.bio](http://alignment.sandbox.bio).

Or, to run it locally:

```bash
npm install
npm run build
```

and open the `index.html` file in your browser.


## How it works

- To perform the sequence alignment, Alignment Sandbox runs the C tool [seq-align](https://github.com/noporpoise/seq-align) directly in the browser using WebAssembly. For details about the compilation from C to WebAssembly, see the [biowasm](https://github.com/biowasm/biowasm) project.
- Alignment Sandbox uses the [aioli](https://github.com/biowasm/aioli) library to run the WebAssembly module in a WebWorker.
