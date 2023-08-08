At present, comprehending Bram's plan to progress with vim9script in the same direction he did is a challenging task, and it is also a significant endeavor to become proficient in that aspect of Vim 9.

Therefore, I propose to integrate [QuickJS](https://bellard.org/quickjs/), a small and embeddable Javascript engine, to enable JavaScript and TypeScript in Vim.

## What about the existing if_ interfaces?

If everything I remembered is correct, the if_ interfaces have already been in existence for 9 or 10 years, but are still not very popular.

Due to the additional setup procedures required for all if_ interfaces and their limited availability on every machine, plugin authors often choose not to use them in order to attract a wider user base. Additionally, users tend to prefer pure Vimscript plugins over those that rely on +python and +lua capabilities.

Perhaps there are other reasons, but after years of experimentation, it has ultimately proven to be unsuccessful.

## Why not statically link against the Lua library?

Actually, it has already been disscussed before and got rejected by Bram:

- https://github.com/vim/vim-win32-installer/issues/182
- https://github.com/vim/vim/pull/6143

Given the reasons mentioned above, let’s revisit the discussion. This solution can align with nvim’s Lua integration, and it seems promising.

However, it is still not a perfect solution because Lua is not considered a modern language and is primarily designed for small scripts or configuration purposes. Maintaining complex projects written in Lua is a nightmare.

## Lua is not a modern language

Lua is not well-suited for complex projects due to the following reasons:

- Variable scope and function scope default to global, which make it appear like a outdated programming language from before the 1980s.
- A dereference on a non-existing key returns nil instead of an error. 
- No way to impose constraints on function arguments. 'Safe' Lua functions are a mess of type-checking code.
- Lack of object model. Can simulated by table, but it leads to some inconsistencies
- Indexing starts from 1, requiring additional conversion when extending it in the C languag.
- `nil` is everywhere, you always had to handle them throughout your code.
- At least 5 ways to define a module, 3 ways to define a class.
- Lacks standard library, every project has to invent its own stdlibs, which cause your code hard to reuse across projects.
- One piece of code from project1 always can't be directly reused in project2 without modification.
- More choices mean more confusion.
- Lacks proper tooling.
- Too many choices can actually be a pain, start to hinder our decision-making ability.

## QuickJS advantages

- [ES2020](https://tc39.github.io/ecma262/) compliance: passes nearly 100% of the ECMAScript Test Suite tests when selecting the ES2020 features.
- Portable and small in size, at only **210 KiB** of x86 code for a simple "Hello World" program.
- Consists of just a few C files with no external dependencies.
- Allows for the simultaneous use of both JavaScript and TypeScript in Vim.
- There are many programmers who are skilled in using both JavaScript and TypeScript, which will make contributing to Vim's ecosystem much more easy.
- JavaScript and TypeScript have mature development tools and abundant resources.

QuickJS is being widely used in game development nowadays. Additionally, it is also being utilized in mobile apps as an alternative to the big V8 engine.

I am not the only one endorsing JavaScript/TypeScript, there are many ongoing projects such as denops.vim and coc.nvim, which enable writing plugins in JS/TS.

The cost of integrating QuickJS into Vim is minimal, as it will only increase the binary size by approximately 210KB. However, the benefits are huge, since adopting the JavaScript/TypeScript ecosystem can greatly contribute to Vim's enhancement and success.

