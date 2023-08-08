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

However, it is still not a perfect solution because Lua is not considered a modern language and is primarily designed for small scripts or configuration purposes. Maintaining complex projects written in Lua could be a nightmare.

I will explain this below.

## Lua is not a good language

Lua is not well-suited for complex projects due to the following reasons:

- Variable scope and function scope default to global, which make it appear like a outdated programming language from before the 1980s.
- A dereference on a non-existing key returns nil instead of an error. 
- No way to impose constraints on function arguments. 'Safe' Lua functions are a mess of type-checking code.
- Lack of object model. Can simulated by table, but it leads to some inconsistencies.
- Lack of unicode support, string is more or less an alias to bytes.
- Lack of switch/case statement, and instead relies on numerous if/else statements to simulate it.
- Indexing starts from 1, requiring additional conversion when extending it in the C languag.
- `nil` is everywhere, you always had to handle them throughout your code.
- At least 5 ways to define a module, 3 ways to define a class.
- Lacks standard library, every project has to invent its own stdlibs, which cause your code hard to reuse across projects.
- One piece of code from project1 always can't be directly reused in project2 without modification.
- More choices mean more confusion.
- Lacks proper tooling.
- Too many choices can actually be a pain, start to hinder our decision-making ability.

Despite being invented in the 1990s, Lua exudes a nostalgic 60-70s vibe.

Even with the integration of the static Lua library, code from nvim may still not be reusable in vim due to the completely different APIs underlying them. Nvim utilizes libuv and dedicated nvim_ functions, whereas vim relies on traditional interfaces.

The division already exists and will not be eliminated simply because vim integrates Lua.

## QuickJS advantages

- QuickJS is also small in size, requiring only 210 KB of extra binary size to run a "hello world" program..
- [ES2020](https://tc39.github.io/ecma262/) compliance: passes nearly 100% of the ECMAScript Test Suite tests when selecting the ES2020 features.
- Consists of just a few C files with no external dependencies.
- The JS-based technology stack has a wealth of previous achievements for you to utilize, making it a better ecosystem.
- Less typing: `{ }` vs `begin/end`.
- Simple tasks can be written in JavaScript, while complex tasks can be written in TypeScript, the most modern and elegant scripting language.
- There are many programmers who are skilled in using both JavaScript and TypeScript, which will make contributing to Vim's ecosystem much more easy.
- Among numerous JS virtual machine implementations, QuickJS has relatively good performance, second only to V8.
- Lua has only got over 10k projects on GitHub, while the number for JavaScript and TypeScript stands at 380k and 130k, respectively.
- Plenty of development tools and resources.

QuickJS is being widely used in game development nowadays. Additionally, it is also being utilized in mobile apps as an alternative to the big V8 engine.

The cost of integrating QuickJS into Vim is minimal, as it will only increase the binary size by approximately 210KB. However, the benefits are huge, since adopting the JavaScript/TypeScript ecosystem can greatly contribute to Vim's enhancement and success.

## Side-by-side comparison

Declaring a class in TypeScript is very straightforward:

![](https://github.com/skywind3000/vim-proposal/blob/master/images/class-ts.png?raw=true)


Creating an instance is also clear and easy:

```TypeScript
var p = new Person("skywind", 18, 1800)
console.log(p.toString())
```

Isn’t this how a program should be written?

In Lua, the concept of "less is more" means that it doesn’t provide a built-in class construct. Instead, it suggests using tables and the setmetatable function to simulate classes:

```lua
Person = {}
Person.__index = Person

function Person:create(name, age, salary)
	local obj = {}
	setmetatable(obj, Person)
	obj.name = name
	obj.age = age
	obj.salary = salary
	return obj
end

function Person:toString()
	return self.name .. ' (' .. self.age .. ') (' .. self.salary .. ')'
end

local p = Person:create('skywind', 18, 1800)
print(p:toString())
```

Honestly, tell me which kind of code you would prefer to write? Lua’s approach to class definition can indeed appear messy at first glance, and it becomes even more challenging when dealing with inheritance. The use of the `":"` symbol in Lua is also unconventional compared to mainstream languages.

Writing this type of program can easily become a tangled mess when it grows larger. It’s prone to mistakes, and you may not even realize that you forgot to write a line like `Person.__index = Person`, which could lead to unpredictable outcomes. Many new technologies aim to simplify existing ones, but often end up adding more complexity. In other languages, defining a basic class is a simple task, but in Lua, it becomes quite frustrating.

In some projects, to simplify this matter, they have implemented a function called `"class"`` that allows you to define a class like this:

```Lua
Account = class(function(acc,balance)
              acc.balance = balance
           end)

function Account:withdraw(amount)
   self.balance = self.balance - amount
end

-- can create an Account using call notation!
acc = Account(1000)
acc:withdraw(100)
```

Does it look good? Not really, but it helps you handle the tedious tasks like setmetatable and inheritance, avoiding any omissions. However, the `"class"` implementation in this project is not compatible with classes implemented in other projects. They cannot inherit from each other, and even the instantiation process can have multiple approaches:

```lua
p = Person:create("project1", 18, 1800)
p = Person:new("project2", 18, 1800)
p = Person("project3", 18, 1800)
```

The instantiation functions in Project 1 and Project 2 have different names. Project 3 has its own implementation of a class, where you directly call the class name. The classes implemented in the first project cannot be called using the approach used in the second project.

Without a standard, collaboration becomes difficult. Even the external editor/IDE finds it challenging to understand what kind of class you have defined or what members it contains. As a result, you don’t receive much assistance and support during development.

After writing code in this way for a while, you may find yourself questioning why you are wasting so much time on the seemingly unnecessary task of ensuring compatibility in class definitions. Why can’t we have a unified approach like in TypeScript/JavaScript, where all projects use the new keyword for instantiation and the class keyword for declaration?

At this point, Lua gurus would tell you, "This is called the **Less is more** principle, and you don’t understand it." Doesn’t that make you feel like smashing your keyboard?

It is precisely because of Lua’s shortcomings that language fragmentation has occurred. Each project has to come up with its own set of fundamental things, making it difficult for code to be reused between projects and knowledge to accumulate. Most Lua code is difficult to exist independently from the project it belongs to, resulting in a lack of knowledge accumulation. The consequence is that, although Lua is 21 years older than TypeScript, the number of open source projects using Lua is less than 1/10 of those using TypeScript.

Comparing with TypeScript mentioned earlier, who is more beautiful and who is uglier? Who is better and who is worse? It is clear at a glance.

## Error indentification

TypeScript has a well-designed compiler and linter that can help you identify errors in the editor during the compilation or editing phase.

With Lua, you have to run the code to find out where the error occurred. This is Lua’s so-called "Less is more" approach, where it doesn’t tell you where the mistake is and instead lets you step on the landmine during runtime. In the end, you spend more time paying the price for its shortcomings.

Therefore, Lua is not suitable for writing large programs. When the program becomes large, Lua code tends to become fragmented and loses maintainability.

Lua has a confusing rhetoric, which goes like, "Lua’s positioning has always been small and refined, not like Python/Java." If you truly only use Lua to write small code snippets or configurations of one or two hundred lines, I have no objections. 

However, when it comes to complex plugin development, where Lua is often used to write 10k+ of lines of new plugins/modules, it is no longer appropriate to use the "small and refined positioning" as an excuse for its syntactic shortcomings in the face of complexity and maintainability.

## Existing attempt in Vim

I am not the only one who endorse JavaScript/TypeScript; there are several famous projects that allow writing plugins in JS/TS:

- [CoC](https://github.com/neoclide/coc.nvim): Nodejs extension host for vim & neovim, load extensions like VSCode and host language servers. There are [323 CoC packages](https://www.npmjs.com/search?q=keywords%3Acoc.nvim) written in JavaScript or TypeScript.
- [denops.vim](https://github.com/vim-denops/denops.vim): created by vim-jp, allows developers to write plugins in TypeScript/Deno. 

It can be seen from this that many programmers hope to use TS/JS to extend Vim. We can't simply ignore that there are 323 npm packages powered by CoC.

These two projects have already proven the feasibility of using JavaScript/TypeScript in Vim.

## Summary

Introducing TypeScript/JavaScript to Vim does not mean replacing vim9script. Just provide more possibilities. It could also be a significant opportunity to rejuvenate and enhance Vim’s greatness once again.




