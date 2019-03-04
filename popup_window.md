# Popup Window API Proposal

## Problem

Vim has already got some builtin popup-widgets for certain usage, like completion menu or balloon. But users and plugin authors are still asking for more:

- 3379: [Support Floating Window](https://github.com/vim/vim/issues/3379)
- 2948: [balloon_show doesn't display the balloon](https://github.com/vim/vim/issues/2948)
- 2352: [Call to balloon_show() doesn't show the balloon in the terminal](https://github.com/vim/vim/issues/2352)
- 3811: [Feature Request: Trigger balloon via keyboard at cursor position](https://github.com/vim/vim/issues/3811)
- (YCM): [Implement argument completion/hints](https://github.com/Valloric/YouCompleteMe/issues/234)
- [RFC: Signature help/argument hints in a second popup menu](https://groups.google.com/forum/#!searchin/vim_dev/argument$20hints%7Csort:date/vim_dev/YntCTlqdVyc/Me8uwfBpAgAJ)

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

Summary, there are two types of popup window: **interactive** and **non-interactive**. 

For **non-interactive** popup windows, people want to use them to:

- Display the documentation for a function:

![emacs-2](https://user-images.githubusercontent.com/3035071/53684507-46213880-3d49-11e9-822e-44f6520c00f9.png)

- Display linting errors or warnings:

![kakoune-1](https://user-images.githubusercontent.com/3035071/53684508-4f120a00-3d49-11e9-8bff-2e68aba12917.jpg)

- Preview a portion of file (can be used to preview grep results in quickfix):

![emacs-1](https://user-images.githubusercontent.com/3035071/53684510-56d1ae80-3d49-11e9-9b6e-f3735784c258.jpg)

- Display funny help for newbies (like paperclip assistent in office 2000):

![kakoune-2](https://user-images.githubusercontent.com/3035071/53684518-5cc78f80-3d49-11e9-8cac-7883d3052ae6.jpg)


For **interactive** popup windows, people want to use them to:

- pick an item:

![ex-pick](https://user-images.githubusercontent.com/3035071/53684524-65b86100-3d49-11e9-9593-d133d00e324b.png)


- display a nice popup menu:

![ex-menu](https://user-images.githubusercontent.com/3035071/53684526-6b15ab80-3d49-11e9-96ae-bc3b5e1f52c9.png)


- cmd line completion:

![ex-cmdline](https://user-images.githubusercontent.com/3035071/53684527-723cb980-3d49-11e9-9b6b-1a6ca7f1a19c.png)

- use fuzzy finder in a popup window:

![ex-fuzzy](https://user-images.githubusercontent.com/3035071/53684528-7963c780-3d49-11e9-8c69-5d8d7b1b7cc2.png)


There are too many popup-related widgets for certain usage, designing one by one is nearly impossible. 

Implementing a neovim's [floating window](https://github.com/neovim/neovim/pull/6619) will take too much time (it is working in progress for almost 2-years and still not merge to master).

Can we implement the popup window in a simple and adaptive way ? Is this possible to unify all their needs and simplify API design ? 

The following parts of this article will introduce an `overlay` mechanism similar to Emacs's [text overlay](https://www.gnu.org/software/emacs/manual/html_node/elisp/Overlays.html) which is the backend of various [popup windows](https://github.com/flycheck/flycheck-popup-tip) in Emacs.


## API Scope

Popup windows will draw into an overlay layer, A popup will remain after creation  until an `erase` function is called. Everything in the overlay will not interfere vim's  states, people can continue editing or using vim commands no matter there is a popup window  or not.

So, the APIs are only designed for drawing a popup window and has nothing to do with user input. Dialogs like `yes/no` box and confirm box require user input, can be simulated with following steps:


```VimL
function! Dialog_YesNo(...)
    while not_quit
       draw/update the popup window
       get input from getchar()
    endwhile
    erase the popup window
endfunc
```

Popup window APIs is not responsible for any interactive functionalities. Instead of implementing a complex widget/event system (which is too complex), it is sane to let user to handle the input by `getchar()`.

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
|   Overlay Buffer (M rows and N columns)
|
Ground Vim UI (M rows and N columns)
 
```

Similar to Video Buffer (0xb8000) in x86's text mode, the overlay buffer is a 2D array of characters and attributes. Change the content of the 2D array will change the text in the screen.

The overlay buffer is invisible by default and can be enabled by:

```
set guioptions+=o
```

Every existent text rendering code in both Vim & GVim needs to be updated to support this overlay buffer, once it finished, we can use it to build powerful popup windows.

There are also some basic APIs for overlay buffer:

- add a text string with position and attribute.
- erase a rectangle of text.
- command `redrawoverlay` to update the overlay (it uses double buffer to prevent flicker).

With these primitive APIs, user can draw what ever they like on the overlay buffer. 

## Overlay Panes

Overlay panes is an abstraction of the popup windows, it consists of:

- position and size
- z order (for overlapping calculation)
- background color
- border styles
- lines of text and text-properties

There can be multiple panes at the same time, the panes can be manipulated by:

```C
pane_create(int row, int col, int width, int height, ...);
pane_destroy(int pane_id);
pane_update(int pane_id, ...);
pane_move(int pane_id, int new_row, int new_col, int new_width, int new_height);
pane_show(int pane_id, bool show_hide);
```

(PS: they are provided as both C-apis and vim functions).

The life cycle of a pane is between `pane_create` and `pane_destroy`.

The (row, col) is using screen coordinate system, and there can be some functions to convert window based coordinate system to screen coordinate system. If you want to display a popup balloon right above your cursor, you can use them to calculate the position.

Finally, there is a function to render the pane list into the overlay buffer:

```C
pane_flush();
```

If you create some panes, the `overlay buffer` will not change until `pane_flush()`.


## Popup Windows

All the common popup windows are implemented in `popup.vim` script, they will use `panes` to display a popup window and `getchar()` to provide interactive functionalities.

There are some predefined popup windows:

- Popup_Message(): display a "build complete" message and hide after a few seconds.
- Popup_LinterHint(): display a message right above cursor and hide if cursor moves outside current `<cword>`.
- Popup_Menu(): display a menu (use `getchar()` to receive user input) and quit after user select an item.
- Popup_YesNo(): display a `yes/no` box and return the result after user made a selection.
- ... 
- and so on.

User or plugin authors can use the high level APIs provided by `popup.vim` or design their own popup window by utilizing lower level pane APIs or overlay APIs.

## Summary

It is complex to design an event system or NeoVim's floating window, and nearly impossible to implement every type of popup window for certain usage.

To unify and simplify the interface, this proposal suggests to provide an overlay mechanism with some primitive APIs to:

- render a popup window (pane)
- erase a popup window
- update a popup window

And let user handle input themself by `getchar()`. At last, makes it possible to enable users to create various popup windows with different styles and functionalities.

--
2019/3/2 edit: floating window got merged to master.