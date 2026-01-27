This project is divided into three categories: The frontend (`app/`),
the machinery behind it (`lib/`) and some tests (`test/`).

# app

The app consists of an interactive mode (only running in `js-web` due to
its HTML rendering) and a terminal UI (working only in `js` for now).

We use a stateless model-view architecture inspired by reactive
frameworks such as React or Elm. Its hidden internals are defined in
`lib/ui/{view,model}`. There is a global model (`app/model`) that
handles generic operation for both backends. The only interface the
backend models have to implement is the `NetModel` (`app/net`).

Specifically, each app has a `view` function that defines its frontend.
This function can dispatch effects then handled by the generic model and
then potentially forwarded to the backend models.

`app/model` handles things such as compiling/rendering the input,
storing the current configuration/source and forwarding other requests
to the respective backend models.

`app/interactive/` is the main backend. Its HTML view is defined
declaratively. Internally, it uses tagging machinery that tags each HTML
tag uniquely, such that partial redraws can happen. The rendering in the
model converts from the `NetStream` to geometric objects drawable on a
canvas (more below). The rest of the code should be fairly
self-explanatory.

`app/cli/` is an experimental terminal interface (TUI) for NodeJS. It
translates only from the lambda calculus and does not feature a
graphical interpretation of the net. Otherwise it supports similar
features as the interactive implementation and shares some code with it
(including the common `NetModel` interface).

# lib

The library is responsible for multiple aspects of our program. For one,
we have two languages (interaction calculus (IC) and lambda calculus
(LC)) that have to be parsed, checked, compiled and reduced. The final
product is always the `Net` structure, which abstracts aways over
irrelevant things like named variables.

We also have a lot of code responsible for the rendering of this `Net`
structure. For that we use separate types (triangle, arrow) that only
have properties actually required for rendering them.

The library also defines the abstract implementations of the frontend
model/view infrastructure.

# Net Pipeline

The concrete different net representations:

    /// lambda calculus term
    /// (lib/language/lc/term)
    type Term { Sym(...); Abs(...); App(...) }

    /// IC constructor with named ports+polarities
    /// this is the intermediate representation as you would parse it from IC
    /// (lib/language/ic/term)
    type Constructor(...)

    /// IC constructor with unique, numeric ports, annotated with position vectors
    /// this is the intermediate representation used for force-directed layouting
    /// (lib/net/net)
    type Agent(...)

    /// as the Agents have unique ports, we have a separate Wire structure that connects them together
    /// (lib/net/net)
    type Wire(...)

    /// the final representation used for rendering the net to a canvas
    /// contains color, size, position, etc.
    type Triangle(...)
    type Arrow(...)

As we generally stream all the constructs via effects between stages, we
use type aliases for `WireStream = emit[Wire]`,
`AgentStream = emit[Agent]`, `NetStream = { WireStream, AgentStream }`,
as well as `Draw = { emit[Arrow], emit[Triangle] }`. We do not use
interfaces for such streams, as we also have to reify them between
stages because of the stateless UI model. Also, streams can otherwise
not be replayed if they use side effects (like parsing).

Roughly, a full pipeline from LC could look like this (slightly modified
signatures for simplicity):

    language/lc/parser::parse!() / { read[Char], emit[Term] } // parse to term
    -->
    language/lc/netter::net! / { read[Term], emit[Constructor] } // translate to IC constructors
    -->
    language/ic/ruler::redexes { / emit[Constructor] } / emit[Redex] // find redexes between IC constructors
    -->
    language/ic/ruler::step { / emit[Constructor], emit[Redex] } / { emit[Constructor], emit[Redex] } // apply one rule to some redex
    -->
    ... // as many reduction steps as wanted
    -->
    net/net::net! / { read[Constructor], NetStream } // convert constructors to positioned nodes with explicit edges
    -->
    net/net::layout { / NetStream } / NetStream // single-step force-directed layout
    -->
    ... // as many layout steps as wanted
    -->
    [frontend] { / NetStream } / Draw // convert to canvas elements with color, frontend-specified sizes etc.
    -->
    ui/canvas::draw { / Draw }: Canvas // apply geometries to internal canvas structure
    -->
    ui/canvas::applyTo(canvas, node) // apply canvas to HTML node
    -->
    ... // repeat from reduction/layout step
