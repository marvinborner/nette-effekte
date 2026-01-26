# Nette Effekte

> interactive interaction net compiler and reducer

      ╭─────╮
    ╭─┴─╮ ╭─┴─╮
    │ ζ │ │ δ │  
    ╰┬─┬╯ ╰┬─┬╯
     │ ╰───╯ │
     ╰───────╯

-   [THEORY](THEORY.md)
-   [ARCHITECTURE](ARCHITECTURE.md)

> no ai was used for any part of this project

# installation

## manual

-   Install a current Effekt version and its JS dependencies as per [the
    instructions](https://effekt-lang.org/docs)
    -   You can ignore LLVM, as we do not support it for now

## nix

> the nix setup is untested except for running CI

``` bash
nix develop # open shell with installed dependencies
nix run     # run main entry
nix build   # build project
```

# run

## interactive web

``` bash
effekt --backend js-web app/main.effekt
```

If building has finished successfully, Effekt will give you a link you
can open in a browser (recommended: modern desktop firefox).

You can enter terms in the left panel (see [THEORY](THEORY.md) for the
syntax). Multiple lines will be interpreted as multiple constructors or
multiple lambda terms. You can then render the input by clicking the
*render* button. Pan/zoom the produced graph by dragging/clicking your
mouse. Step the reducer by pressing *step*. Change the source language
by pressing the *mode* button.

## cli

``` bash
effekt --backend js app/main.effekt
```

The CLI only supports lambda terms (see [THEORY](THEORY.md) for the
syntax). Enter a term and press enter to compile. Step reduction with an
exclamation mark `!`. Exit with `Ctrl+c`.

## tests

``` bash
effekt --backend js test/main.effekt
```

# examples

## lambda calculus

Remember the [THEORY](THEORY.md): for now this only supports a very
small subset of the lambda calculus (we do not even label duplicators)!
Though at least the linear lambda calculus is supported.

-   `x => x` (identity)
-   `x => y => x` (`k`, notice the erasure via nilary duplicator)
-   `x => x x => x` (parsed as two separate identities)
-   `(x => (x x) x => x)` (duplication of identity, reduces to identity)
-   `s => z => (s (s (s (s z))))` (church 4, nesting of duplicators)
    -   (effekt might crash with `b_k_0 is not defined` here)

## interaction calculus

# things to note / bugs

-   sometimes you have to resize the window first in order for the
    canvas size to be interpreted correctly (otherwise the agents can
    appear stretched/squeezed)
-   the parser emits successfully parsed parts of the term eagerly, so
    if there's some garbage after correctly parsed code, it will not
    complain
-   if you leave the site open for too long, your memory will eventually
    run out
