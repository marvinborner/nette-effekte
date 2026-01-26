# Theory

> this file is best read with a good monospace font in a terminal

This project is all about interaction combinators. Interaction
combinators are a subset of interaction nets, a graphical model of
computation. Specifically, we have two variadic agents; the constructor
and duplicator. *Agents* (like nodes in a graph) are connected by
*wires* (edges). The final graph is called a *net*. In our case, these
wires are directed (*polarized* - another subset of interaction
combinators).

Specifically, agents consist of *ports* where one end of a wire is
connected to. Each port can only be connected to a single port with a
single wire (linearity). Ports are either positive (+) or negative (-).
Only positive ports can be connected by wire to negative ports and
vice-versa.

There exist two port variants: *Principal ports* and *auxiliary ports*.
Each agent has exactly one principal port and 0-n auxiliary ports. When
the principal ports of two agents are connected by wire, they can
*interact*. Interaction is the only kind of net reduction. Through
interaction, both interacting agents get removed, while the other ends
of the wires of both agent's auxiliary ports can be rewired arbitrarily
to each other or to new agents constructed through the interaction. The
interaction is therefore entirely local, as an interaction can never
interfere with another interaction.

We write constructors `c` and duplicators `d` using the following
grammar:

    p ::= + | -
    l ::= a, b, abc, ...
    c ::= lp(lp, ..., lp)
    d ::= lp[lp, ..., lp]

For example:

    a+(b+, b-)
    a-[c+, c-]

Here, the principal ports `a` of both agents are connected to each
other, while the auxiliary ports are connected per agent (`b+ -> b-` and
`c+ -> c-`).

In interaction nets, the interaction *rule* that specifies the precise
graph rewrite can be chosen arbitrarily by the interpreter (interaction
system). Due to the restricted set of agents, interaction combinators
can interact in one of two ways: Commutation (interaction between
duplicator and constructor), or annihilation (interaction between
constructor and constructor).

The interaction between two duplicators depends on the desired semantics
-- for the semantics of the subset of the lambda calculus typable in the
Elementary Affine Logic (EAL), duplicators must have labels that specify
whether two interaction duplicators annihilate or commute. To support
the semantics of the full lambda calculus, you have to additionally
insert control agents into the net ("bookkeeping", "oracle")\*. In this
experimental project, we always annihilate two connected duplicators.

*Commutation* of two n-ary agents (ζ = constructor, δ = duplicator):

     │⋯│               │         │
    ╭┴─┴╮            ╭─┴─╮     ╭─┴─╮
    │ ζ │            │ δ │  ⋯  │ δ │
    ╰─┬─╯            ╰┬─┬╯     ╰┬─┬╯
      │               │ ╰───╭───╯ │
      │    ───────►   │     │     │
      │               │ ╭───╯───╮ │
    ╭─┴─╮            ╭┴─┴╮     ╭┴─┴╮
    │ δ │            │ ζ │  ⋯  │ ζ │
    ╰┬─┬╯            ╰─┬─╯     ╰─┬─╯
     │⋯│               │         │

*Annihilation* of two n-ary agents:

     │⋯│              │ ⋯ │
    ╭┴─┴╮             │   │
    │ ζ │             │   │
    ╰─┬─╯             ╰─╭─╯
      │                 │
      │    ───────►     │
      │                 │
    ╭─┴─╮             ╭─╯─╮
    │ ζ │             │   │
    ╰┬─┬╯             │   │
     │⋯│              │ ⋯ │

Notably it can very well happen that a duplicator interacts with a
constructor that has different polarities for some of its auxiliary
ports. In this case, the resulting duplicators (at the top right) turn
into a variant with different polarizations at their principal ports. A
common example is the dual of the duplicator (called superposition):
`x+[x1-, x2-, x3-]` (sup) vs `x-[x1+, x2+, x3+]` (dup).

\*(as interaction combinators are Turing complete: the full lambda
calculus could also be fully supported without control nodes -- in fact
there is a lot of literature on that --, but you lose part of the reason
why you would typically want to translate to interaction nets in the
first place; this being increased sharing even under abstractions,
incremental superpositions, "optimal reduction")

## Lambda Calculus

The lambda calculus syntax we parse and support is strictly the
following:

    x ::= a, b, abc, ...
    e ::= x | x => e | (e .. e)

For example $(\lambda x.x x) \lambda y.y$ would be written as
`(x => (x x) y => y)` in our syntax.

We choose the following translation:

                   ₚ┌┄┄┄┄┄┄┐
    ⟦M⟧     =    ◯──┤ ⟦M⟧ₚ ┊
                    └┄┄┄┄┄┄┘


                       p
               ┌┄┄┄┄┄┄┄│┄┄┄┄┄┄┄┄┄┄┄┐
               ┊     ╭─┴─╮+        ┊
               ┊     │ ζ │         ┊
               ┊    +╰┬─┬╯-        ┊
    ⟦λx.M⟧ₚ =  ┊   x╭─╯ ╰───╮m     ┊  = p+(x+, m-)
               ┊  ┌─┴─┐  ┌┄┄┴┄┄┄┐  ┊
               ┊  │ δ ╞══╡ ⟦M⟧ₘ ┊  ┊
               ┊  └───┘  └┄┄╥┄┄┄┘  ┊
               └┄┄┄┄┄┄┄┄┄┄┄┄║┄┄┄┄┄┄┘
                        fv(λx.M)
                                           ↑
                                     perfectly dual!
                                           ↓
                        fv(M)
                 ┌┄┄┄┄┄┄┄┄║┄┄┄┄┄┄┐
                 ┊     ┌┄┄╨┄┄┄┐  ┊
                 ┊     ┊ ⟦M⟧ₘ ┊  ┊
                 ┊     └┄┄┬┄┄┄┘  ┊
                 ┊        │m     ┊
                 ┊      ╭─┴─╮-   ┊
    ⟦M N⟧ₚ  =    ┊      │ ζ │    ┊    = m-(n-, p+)
                 ┊     -╰┬─┬╯+   ┊
                 ┊    n╭─╯ ╰─╮   ┊
                 ┊  ┌┄┄┴┄┄┄┐ │   ┊
                 ┊  ┊ ⟦N⟧ₙ ┊ │   ┊
                 ┊  └┄┄╥┄┄┄┘ │   ┊
                 └┄┄┄┄┄║┄┄┄┄┄│┄┄┄┘
                     fv(N)   p

Beware of edge cases in the abstraction translation: If the variable is
bound once (linearly), no duplicator is required. If it is not bound at
all, the duplicator is nilary and becomes an eraser.

## Resources

Drawings were done by me (for another project).

We refer to an excellent collection of interaction net resources:
https://github.com/marvinborner/interaction-net-resources

There's no actually novel theory in this project.
