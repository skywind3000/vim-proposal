# Introducing VSCode's workspace mechanism

## Problem

https://github.com/vim/vim/issues/3573#issuecomment-433705924

> It would be great to have a standardized APIs for project-related information (e.g. file lists, root directory, VCS type, a way to set project-specific buffer-local options / variables etc.). Much of this can be implemented in vim plugins but it would be preferable if vim provided a standard way to do this.

## Project Information

Vim can introduce a way to provide following information:

- root directory
- file list
- buffer-local options

Plugins can have a place to store project-specific configurations:

- ctags/cscope parameters
- linter parameters
- include folders (c/c++)
- CFLAG / CXXFLAG / LDFLAGS (c/c++)
- actions: command to run, debug etc.

## Goals

- A way to locate project root (use root markers).
- A place for plugins to read project local configurations (like `.vscode` folder in vscode).
- A standard that plugins can follow even if running in old vims.
- Very simple to implement.

## Solution

Use something like vscode's `.vscode` folder:

```
project1
├── include
├── src
│   ├── module1
│   └── module2
└── .vscode                  <- located in the project root directory
    ├── settings.json        <- project-specific settings
    └── tasks.json           <- how to "debug" this project or "build" this project
```

The `.vscode` folder in vscode is used to store project local settings and it locates in the project root directory. Every file inside `project1` will use the settings defined in `.vscode/settings.json`. And the `.vscode` folder can be used to locate project root for current file.

Similar to `.vscode`, this solution will start by defining a new `.vimprj` folder (the name is configurable).

### Root marker

Introduce a new option `rootmarker` for locating project root of current file, it defaults to `.vimprj` and can be set:

```VimL
set rootmarker=.vimprj
set rootmarker=.vscode
set rootmarker=.git
set rootmarker=.project
```

By default, there will be an `.vimprj` folder (can be renamed by `set rootmarker=xxx`) in your project root directory. Like vscode use a `.vscode` folder in every project root to store workspace settings. Plugins can put their project related configuration file in the `.vimprj` folder.

### Locating project root

The project root of current file is the nearest parent directory with a folder named `.vimprj` in it.

A buffer-specific variable `v:projectroot` is used to store the project root directory of current buffer, it is initialized when you open a file as:

```VimL
let v:projectroot = fnamemodify(find(&rootmarker, '.;'), ':h')
```

If `.vimprj` can not be found in the parent directories, an empty string will be used. For files already opened by vim, `v:projectroot` can be updated when you close the file and open again.

**Plugins can use the same algorithm to calculate project root directory even if running in old vims.**

### Modifiers

```
%:j    - same as v:projectroot, project root directory of current buffer
%:i    - file path relative to current project root
```

These modifiers can be used in command line or passing to `expand(xxx)`.

Use modifiers to cd to the project root:

    :cd %:j

Open nerdtree in the project root:

    :NERDTree %:j

Open netrw in the project root:
  
    :Explore %:j


### Buffer local settings

Like `.vscode/settings.json` in vscode, if there is a `settings.json` file located in `.vimprj` folder, it will be loaded when you open a file in this project:

```json
{
    "shiftwidth": 2,
    "tabstop": 2,
    "expandtab": false
}
```

Each key/value pair in `settings.json` will be initialized as a buffer local setting. 

**Alternative way** for this is to create a `local.vim` in the `.vimprj` folder and it will be sourced in `sandbox` for every file in the project.

This can be done by a vimscript shipped with vim (like matchit.vim).


### Old vim compatibility

Plugins running in old vims can calculate project root themself:

```VimL
if !exists('g:rootmarker')
    let g:rootmarker = exists('+rootmarker')? &rootmarker : ".vimprj"
endif
if !exists('v:projectroot')
    let b:_project_root = fnamemodify(find(g:rootmarker, '.;'), ':h')
else
    let b:_project_root = v:projectroot
endif
```

With this standard, even if running in old vims, plugin can get the project root directory of current file.

### File list

There can be a `.vimprj/filelist` file to describe the filelist in this project:

```
inc/interface.h
src/hello.c
src/world.c
src/module/xp.c
```

This file is optional, if it doesn't exist, all files inside root directory will be considered as project files.

If a plugin need generate ctags database for current project or need grep symbol in current project, the filelist can be used for that.
