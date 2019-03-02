# Popup Window API Proposal

## Problem

Vim has already got some builtin popup-widgets for certain usage, like completion menu or balloon. But users and plugin authors are still asking for more:

- 3379: [Support Floating Window](https://github.com/vim/vim/issues/3379)
- 2948: [balloon_show doesn't display the balloon](https://github.com/vim/vim/issues/2948)
- 2352: [Call to balloon_show() doesn't show the balloon in the terminal](https://github.com/vim/vim/issues/2352)
- 3811: [Feature Request: Trigger balloon via keyboard at cursor position](https://github.com/vim/vim/issues/3811)

In Vim conf 2018, Bram claimed that he has [plan for this](https://vimconf.org/2018/slides/Vim_From-hjkl-to-a-platform-for-plugins.pdf) already: 

> Popup Windows:
> 
> Use for a notification:
> - Asynchronously show window with text “build done”.
> - Remove after a few seconds.
> 
> Use for picking an item:
> - Show window where each line is an item
> - Let user pick an item
> - A bit like confirm() but much nicer

Summary, there are two types of popup window: interactive and non-interactive. 

For **non-interactive** popup windows, people want to use them to:

- Display the documentation for a function:
![](images/emacs-2.png)

- Display linting errors or warnings:
![](images/kakoune-1.jpg)

- Preview a portion of file (can be used to preview grep results in quickfix):
![](images/emacs-1.jpg)

- Display funny help for newbies:
![](images/kakoune-2.jpg)


For **interactive** popup windows, people want to use them to:

- pick an item:

![](images/ex-pick.png)

- display a nice popup menu:

![](images/ex-menu.png)

- cmd line completion:

![](images/ex-cmdline.png)

- use fuzzy finder in a popup window:

![](images/ex-fuzzy.png)

There are too many popup-related widgets for certain usage, implement one by one is nearly impossible. 

Implementing a neovim's [floating window](https://github.com/neovim/neovim/pull/6619) will take too much time (it is working in progress for almost 2-years and still not merge to master).

Can we implement the popup window in a simple and adaptive way ? Is this possible to unify all their needs and simplify api design ? 

The following parts of this article will introduce an `overlay` mechanism similar to Emacs's [text overlay](https://www.gnu.org/software/emacs/manual/html_node/elisp/Overlays.html) which is the backend of various [popup windows](https://github.com/flycheck/flycheck-popup-tip) in Emacs.


## APIs Scope

Popup windows will draw into an overlay layer, A popup will remain there after creation  until an `ease` function is called. Everything in the overlay will not interfere vim's  states, people can continue editing or using vim commands no matter there is a popup window  or not.

So, the APIs are only designed for drawing a popup window and has nothing to do with interaction. Dialogs like `yes/no` box and confirm box require user input, can be simulated with following steps:


```VimL
function! Dialog_YesNo(...)
    while not_quit
       draw/update the popup window
       get input from getchar()
    endwhile
    erase the popup window
endfunc
```

Popup window APIs is not responsible for any interactive functionalities. Instead of implementing a complex widget/event system (which is too complex), it is sane to let user to handle the interaction by `getchar()`.

There can be a `popup.vim` script contains some predefined popup windows/dialogs and will be shipped with vim itself. User can use the primitive APIs and `getchar()` to implement other complex dialogs like a popup fuzzy finder or a command line history completion box.


## Overlay Buffer

The overlay buffer is a character matrix with the same size of the screen:

```
|\     |\
| \    | \
|  \   |  \
|   \  |   \
|   |  |   |
| 1 |  | 2 |   <---- Observer
|   |  |   |
|   /  |   /
|  /   |  /
| /    | /
|/     |/

^      ^
|      |
|   Overlay Buffer
|
Ground Vim screen
 
```