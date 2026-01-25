---
title: "Nette Effekte"
sub_title: "(EPE Project, Marvin Borner)"
date: ""
theme:
  path: theme.yaml
---

<!-- alignment: center -->
<!-- jump_to_middle -->

`Nette Effekte` \
(interaction net in effekt)

```
  ╭─────╮
╭─┴─╮ ╭─┴─╮
│ ζ │ │ δ │
╰┬─┬╯ ╰┬─┬╯
 │ ╰───╯ │ 
 ╰───────╯ 
```

<!-- end_slide -->

# why?

- literature often obscure => clean™ implementation
- no online reducer for pure interaction combinators
    - especially with interaction calculus input

<!-- end_slide -->

# what?

<!-- column_layout: [5,1] -->

<!-- column: 0 -->

- `interaction combinators`: graph-based model of computation
- web interface (elm-ish):
    - **parse** calculus
    - "**type-check**" etc.
    - **compile** to interaction calculus
    - **rule** stepper (by DSL)
    - interactive **force-directed** rendering

<!-- column: 1 -->

```
  ╭─────╮
╭─┴─╮ ╭─┴─╮
│ ζ │ │ δ │
╰┬─┬╯ ╰┬─┬╯
 │ ╰───╯ │ 
 ╰───────╯ 
```

<!-- end_slide -->

# effects
## model

<!-- column_layout: [4,4] -->

<!-- column: 1 -->

```scala


interface Model[C, O] {
  def render(): Unit / Redraw
  def stepReduction(): Unit / Redraw
  def stepAnimation(): Unit / Redraw
  def nextMode(): Unit / Redraw
  def source(s: String): Unit / Redraw
  def onContext(op: O): Unit / Redraw
                   // partial redrawing!

  def getMode(): Mode
  def getSource(mode: Mode): TextStream
  def getContext(): C
  def getErrors(): String
}
```

<!-- pause -->

<!-- column: 0 -->

```scala
def dispatch(msg: Event): Unit / { Model[Canvas, CanvasOp], Redraw } =
  msg match {
    ...
    case SetCanvasSize(v) =>
      do onContext(Size(v))
    case PanCanvas(v) =>
      do onContext(Pan(v))
    case ZoomCanvas(d) =>
      do onContext(Zoom(d))
  }
```

<!-- end_slide -->

# effects
## web model

```scala
...
div([
    OnDrag(box { v => PanCanvas(v) }),
    OnScroll(box { d => ZoomCanvas(d) })
  ], ["right"], [
  canvas([
    OnResize(box { v => SetCanvasSize(v) }),
    OnTick(StepAnimation()),
  ], ["inet"], do getContext())
]),
...
```

<!-- end_slide -->

# effects
## web pipeline from LC (roughly)

```scala {1|1-3|3-5|5-8|7-10|10-12|12-14|14-16|16-18}
lc::parser::parse!() / { read[Char], emit[Term] } // parse to term
-->
lc::netter::net! / { read[Term], emit[Constructor] } // translate to IC constructors
-->
ic::ruler::redexes { / emit[Constructor] } / emit[Redex] // find redexes
-->
ic::ruler::step { / emit[Constructor], emit[Redex] } / {
  emit[Constructor], emit[Redex] } // apply one rule to some redex (+repeat)
-->
net::net! / { read[Constructor], NetStream } // to positioned nodes with explicit edges
-->
net::layout { / NetStream } / NetStream // single-step force-directed layout (+repeat)
-->
[frontend] { / NetStream } / Draw // convert to canvas elements
-->
ui::canvas::draw { / Draw }: Canvas // apply geometries to internal canvas structure
-->
ui::canvas::applyTo(canvas, node) // apply canvas to HTML node
```

<!-- end_slide -->

# effects
## force-directed layout Ⅰ

```scala
interface Force {
  def origin(): Vector
  def strength(x: Double): Double
}
```

<!-- pause -->

```scala
def force(f: Vector): Vector / Force =
  do origin() + f.normalize * do strength(f.magnitude)

def repulsion(from: Vector, position: Vector): Vector / Force =
  force(position - from)

def attraction(towards: Vector, position: Vector): Vector / Force =
  force(towards - position)
```

<!-- end_slide -->

# effects
## force-directed layout Ⅱ

```scala
def centralGravitation(agent: Agent): Vector / Force =
  Vector(0.0, 0.0).attraction(agent.pos)

def coulombForce(agent: Agent) { agents: => Unit / AgentStream }: Vector / Force =
  with average
  with source[Vector] { agentPositions {agents} }
  with loop
  do emit(repulsion(do read[Vector], agent.pos))

def springForce(agent: Agent, length: Double) { net: => Unit / NetStream }: Vector / Force =
  with average
  with agent.wireForce(length) { agents{net} }
  agent.attachedWires { wires{net} }
```

<!-- end_slide -->

# effects
## catamorphism

<!-- column_layout: [4,3] -->

<!-- column: 0 -->

```scala
def length(t: Term) =
  try t.cata
  with TermF[Int] {
    def sym(x)    = resume(1)
    def abs(x, b) = resume(1 + b)
    def app(f, a) = resume(1 + f + a)
  }
```

```scala
def show(t: Term) =
  try t.cata
  with TermF[String] {
    def sym(x)    = resume("${x}")
    def abs(x, b) = resume("λ${x}.${b}")
    def app(f, a) = resume("(${f} ${a})")
  }
```

<!-- column: 1 -->

```scala
def cata[A](t: Term): A / TermF[A] =
  t match {
    case Sym(x)    =>
      do sym(x)
    case Abs(x, b) =>
      do abs(x, b.cata)
    case App(f, a) =>
      do app(f.cata, a.cata)
  }
```

<!-- end_slide -->

# effects
## compilation

<!-- column_layout: [3,2] -->

<!-- column: 0 -->

```scala
try term.cata
with TermF[Context] {
  ...
  
  def abs(x, b) = {
    val Context(bPort, bBinds) = b
    val kPort = do port("k")
    val xPort = bBinds.bind(x)
    do emit(Constructor(Port(kPort, Pos()), [
      Port(xPort, Pos()), Port(bPort, Neg())]))
    resume(Context(kPort, bBinds.delete(x)))
  }
  
  ...
}
```

<!-- column: 1 -->

```
                  k
          ┌┄┄┄┄┄┄┄│┄┄┄┄┄┄┄┄┄┄┄┐
          ┊     ╭─┴─╮+        ┊
          ┊     │ ζ │         ┊
          ┊    +╰┬─┬╯-        ┊
⟦λx.b⟧ₖ = ┊   x╭─╯ ╰───╮b     ┊
          ┊  ┌─┴─┐  ┌┄┄┴┄┄┄┐  ┊
          ┊  │ δ ╞══╡ ⟦M⟧₆ ┊  ┊
          ┊  └───┘  └┄┄╥┄┄┄┘  ┊
          └┄┄┄┄┄┄┄┄┄┄┄┄║┄┄┄┄┄┄┘
                   fv(λx.b)
```

<!-- end_slide -->

# effects
## other

- fresh ports
- exceptions
- lexing/parsing
- net inspection
- partial redrawing (+ tagging system)

<!-- end_slide -->

# problems

- records: `Agent(...) { .name = "foo" }`
- memory leaks (browser OOM)
- long compilation times
- reification of streams with side effects (e.g. parsing)

<!-- end_slide -->

# stats

<!-- alignment: center -->

| **language**  | **loc** |
|---------------|---------|
| effekt        | 1700    |
| js-web -O0    | 9500    |
| js-web -O3    | 8300    |

<!-- end_slide -->

<!-- column_layout: [3,3] -->
<!-- column: 0 -->

```
           ₚ┌┄┄┄┄┄┄┐
   ⟦M⟧ = ◯──┤ ⟦M⟧ₚ ┊
            └┄┄┄┄┄┄┘



                 fv(M)
          ┌┄┄┄┄┄┄┄┄║┄┄┄┄┄┄┐
          ┊     ┌┄┄╨┄┄┄┐  ┊
          ┊     ┊ ⟦M⟧ₘ ┊  ┊
          ┊     └┄┄┬┄┄┄┘  ┊
          ┊        │m     ┊
          ┊      ╭─┴─╮-   ┊
⟦M N⟧ₚ =  ┊      │ ζ │    ┊ = m-(n-, p+)
          ┊     -╰┬─┬╯+   ┊
          ┊    n╭─╯ ╰─╮   ┊
          ┊  ┌┄┄┴┄┄┄┐ │   ┊
          ┊  ┊ ⟦N⟧ₙ ┊ │   ┊
          ┊  └┄┄╥┄┄┄┘ │   ┊
          └┄┄┄┄┄║┄┄┄┄┄│┄┄┄┘
              fv(N)   p
```

<!-- column: 1 -->

```



                  p
          ┌┄┄┄┄┄┄┄│┄┄┄┄┄┄┄┄┄┄┄┐
          ┊     ╭─┴─╮+        ┊
          ┊     │ ζ │         ┊
          ┊    +╰┬─┬╯-        ┊
⟦λx.M⟧ₚ = ┊   x╭─╯ ╰───╮m     ┊ = p+(x+, m-)
          ┊  ┌─┴─┐  ┌┄┄┴┄┄┄┐  ┊
          ┊  │ δ ╞══╡ ⟦M⟧ₘ ┊  ┊
          ┊  └───┘  └┄┄╥┄┄┄┘  ┊
          └┄┄┄┄┄┄┄┄┄┄┄┄║┄┄┄┄┄┄┘
                   fv(λx.M)
```

<!-- end_slide -->

```
f (Fix f) --- fmap (cata alg) ----> f a
    |                                |
    |                                |
    |                                |
    |                                |
   Fix                              alg
    |                                |
    |                                |
    |                                |
    v                                v
  Fix f   ------- cata alg ------->  a
```
