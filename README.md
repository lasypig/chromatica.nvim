# chromatica.nvim

Chromatica is an asynchronous syntax highlight engine for Neovim. It is
a python3 remote plugin. Currently, chromatica focuses on providing
semantic accurate syntax highlighting for C-family languages (using
libclang).

The project is in alpha state, but it is fairly stable and usable now.

## Features

* Asynchronous parsing and highlighting provides fast and responsive highlight
  as you update your code.
* Semantic-accurate highlighting for C-family languages.

### Example

<img src="https://raw.githubusercontent.com/arakashic/chromatica.nvim/master/figures/comparison_c.png" width="100%"/>

## Prerequites

* [Neovim][3] 0.1.6
* [Python3][4] and [Neovim python client][5]
* [libclang][6] (prefers 3.9.0, though an older version may work as
    well)

### Known Incompatibility

* Python2 (sorry, Python3 only)
* Some syntax plugins (depending on the loading order, third-party
    syntax plugins may overwrite/mess up Chromatica's highlight)

## Installation

### Install Prerequites

Install neovim python client and latest clang
```bash
pip3 install neovim
brew install llvm --HEAD --with-clang
```
Note the install configuration only include the required option
`--with-clang`.  Chromatica should work just fine if you have more
options turned on when building LLVM.

### Install Chromatica

Use a plugin manager (for example, Neobundle)

```vim
NeoBundle 'arakashic/chromatica.nvim'
```

Or manually check out the repo and put the directory to your vim runtime
path.


## Essential Settings

Like many other clang-based plugins, a path to your libclang is needed.
Chromatica will default to `/usr/lib/libclang.so`, but you can specify a
different one by setting

```vim
let g:chromatica#libclang_path='/usr/local/opt/llvm/lib'
```

The path can be set to either the path of the libclang.dylib/libclang.so
file, or the directory that contains it.

If you want Chromatica to be automatically loaded at startup, you will
need to set

```vim
let g:chromatica#enable_at_startup=1
```

Alternatively, you can manually enable and disable Chromatica by calling,
respectively, `:ChromaticaStart` and `:ChromaticaStop`.



## Compilation Flags

Chromatica already has flags for simple codes. To provide the most
accurate highlighting for complex projects, chromatica needs to know the
correct compilation flags such as include search path and macro
definitions. There are two ways to do that in chromatica.

1. A compilation database `compile_commands.json`.

    This is usually generated by CMake. If the project does not use
    CMake, you can generate it using [Bear][7].

2. A `.clang` file that has the compilation flags.

   The `.clang` file allows you to manually set the flags. For example:

   ```
   flags=-I/home/arakshic/.local/include -DNDEBUG
   flang=-I/../src
   ```

   Each `flags` option can have one or more compiler arguments. A
   `.clang` file can have multiple `flags` options. They will be
   concatenated in the order of their appearance.

When chromatica initializes, it search the current directory and the
ancestor directories for these two files. If both file are present,
chromatica will combine the flags in them.

For convenience, you can also set the
`g:chromatica#dotclangfile_search_path` option to the directory that you
put the `.clang` file or the compilation database. It overrides the
default search directory.

## Highlight Feature Level

Chromatica provides different feature levels. Each level enables a
different set of highlight. This is controlled by
`g:chromatica#highlight_feature_level`. 

The default level is 0, which provides basic semantic highlight with
default vim syntax.

A more advanced level is 1, which gets more detailed highlight from
libclang with a customized syntax file.

## Responsive Mode

By default, chromatica only updates highlight when returned to normal
mode after changing the buffer. This is quite awkward since you may have
changed thousands lines of codes, but the highlight will only be updated
when you finish those changes and return to normal mode.

Chromatica provides a responsive mode that reparses and updates the
hightlight as soon as you change the code (in insert mode). To use the
responsive mode, simply set

```vim
let g:chromatica#responsive_mode=1
```

in your vimrc.

Note that the responsive mode comes at the cost of frequent reparsing
the buffer. Even when the highlight is done asynchronously, frequent
reparsing can still cause performance (editor responsiveness) problem if
you C++ code is super complex (Yes, I haven't experienced this problem
with C code). Chromatica uses pre-compiled header to speed up the
repasing and throttles the number of reparse requests per seconds to
avoid reparse flooding. You can increase `g:chromatica#delay_ms` if you
still experiencing performance issues.

## Troubleshooting and Customization

When a token is not highlighted or not highlighted correctly, the first
thing to check if whether Chromatica has the correct compilation
arguments. Because Chromatica uses the clang compiler parser, it is very
important to get all the compilation arguments right. For example, if
the compiler cannot find some of the header file, it may lead to some
tokens does not get highlighted. The command `ChromaticaShowInfo` will
print the basic information for the current buffer including the
location of `.clang`, compilation database, compilation arguments, etc.

Chromatica has a debug log. It can be enabled by executing the
`ChromaticaEnableLog` command (for one time use) or set the
`g:chromatica#debug_log` option. It will generate a `chromatica.log`
file in the current directory.

Chromatica also provides a AST dump feature that is useful for the users
who want to customize the highlight settings. Simply executing the
`ChromaticaDbgAST` will generate a `AST_out.log` file in the current
directory. It contains the parsed tokes in the visible part of the
buffer. The file is color-coded using terminal colors. You might need to
manually enable the parsing of color code in your pager or reader. I
would simply do a `less -R` on it.

For the following sample code

```c++
#include <iostream>

int main(int argc, const char* argv[])
{
    return 0;
}
```

The `AST_out.log` is

```
include chromaticaInclusionDirective [1, 2, 7] PREPROC IDENTIFIER INCLUSION_DIRECTIVE 
iostream None [1, 11, 8] IDENTIFIER INVALID_FILE 
int chromaticaType [3, 1, 3] KEYWORD FUNCTION_DECL FUNCTIONPROTO INT NONE 
main chromaticaFunctionDecl [3, 5, 4] IDENTIFIER FUNCTION_DECL FUNCTIONPROTO INT NONE 
int chromaticaType [3, 10, 3] KEYWORD PARM_DECL INT NONE 
argc chromaticaParmDecl [3, 14, 4] IDENTIFIER PARM_DECL INT NONE 
const chromaticaStorageClass [3, 20, 5] KEYWORD PARM_DECL INCOMPLETEARRAY NONE 
char chromaticaType [3, 26, 4] KEYWORD PARM_DECL INCOMPLETEARRAY NONE 
argv chromaticaParmDecl [3, 32, 4] IDENTIFIER PARM_DECL INCOMPLETEARRAY NONE 
return chromaticaStatement [5, 5, 6] KEYWORD RETURN_STMT NONE 
0 Number [5, 12, 1] LITERAL INTEGER_LITERAL INT NONE 
```

Each line represents one token. Following the token's spelling, there is
the name of syntax group. This syntax group is what you need to set
customized highlight. If a token does not match any syntax group, it
will be shown as `None`.  Then, there is the position of the token in
`[line, start column, length]` format. The rest fields are the raw info
of the token which are useful for debugging when some token is not
correctly highlighted.

## Acknowledgement

This project is largely inspired by [deoplete][1] and [color_coded][2].

[1]: https://github.com/Shougo/deoplete.nvim
[2]: https://github.com/jeaye/color_coded
[3]: https://neovim.io
[4]: https://www.python.org/download/releases/3.0
[5]: https://github.com/neovim/python-client
[6]: http://clang.llvm.org
[7]: https://github.com/rizsotto/Bear
