At present, comprehending Bram's plan to progress with vim9script in the same direction he did is a challenging task, and it is also a significant endeavor to become proficient in that aspect of Vim 9.

Therefore, I propose to integrate [QuickJS](https://bellard.org/quickjs/), a small and embeddable Javascript engine, to enable JavaScript and TypeScript in Vim.

## Advantages

- [ES2020](https://tc39.github.io/ecma262/) compliance: passes nearly 100% of the ECMAScript Test Suite tests when selecting the ES2020 features.
- Portable and small in size, at only **210 KiB** of x86 code for a simple "Hello World" program.
- Consists of just a few C files with no external dependencies.
- Allows for the simultaneous use of both JavaScript and TypeScript in Vim.
- There are many programmers who are skilled in using both JavaScript and TypeScript, which will make contributing to Vim's ecosystem much more easy.
- JavaScript and TypeScript have mature development tools and abundant resources.


QuickJS is being widely used in game development nowadays. Additionally, it is also being utilized in mobile apps as an alternative to the big V8 engine.

I am not the only one endorsing JavaScript/TypeScript, there are many ongoing projects such as denops.vim and coc.nvim, which enable writing plugins in JS/TS.

The cost of integrating QuickJS into Vim is minimal, as it will only increase the binary size by approximately 210KB. However, the benefits are huge, since adopting the JavaScript/TypeScript ecosystem can greatly contribute to Vim's enhancement and success.

