- Title: Towards a new document type for Coq.

- Driver: Emilio J. Gallego Arias (@ejgallego),
  _co-drivers proposed by Emilio_: Hugo Herbelin (@herbelin), Pierre-Marie Pédrot (@ppedrot), Clément Pit--Claudel (@cpitclaudel), Enrico Tassi (@gares), Yann Régis-Gianas (@yurug)
  (_this document is based on many discussions with them_)

- Status: Request For Comments, Partial Prototype Implementation

----

# Summary

Coq pretty printing was designed for a console-based setup, however
the situation these years has much changed and rich layout formats are
the norm, with limitations inherent to console printing still present
in Coq.

For 8.5, a first attempt to address this shortcoming came was made
with the introduction of `Richpp.richpp`. This is marked improvement
from the previous situation. However, the current `richpp` format
loses some important information in its generation, and it is still to
tied to console-based concepts, such as printing width. Once a message
is transformed into a `richpp` format, there is no way to recover the
original box-based layout.

Thus, some use case overlapping is happening between `richpp` and
`Pp.std_cmdpps`, the legacy document representation in Coq. For
instance, it is convenient to be able to concatenate two `richpp`
documents, etc... this leads to a duplication of code and primitives
and in effect, with the non-optimal situation of having two notions of
printing document inside Coq.

This CEP proposes to enrich and consolidate the current `Pp.std_cmdpps` data
type into a more general document type. In particular we propose:

- To make `Pp.std_cmdpps` serializable, allowing User Interfaces to
  render it in a way better adapted to their particular backends. This
  allows for client-side reflowing and line-breaking.
- To merge "rich" printing into the `Pp` datatype, leading to the
  deprecation of `Richpp.richpp`.
- To enrich `Pp.std_cmdpps` with Coq-specific capabilities. For instance,
  define tags to mark implicit arguments, notation boxes, making the features of
  `Ppextend` standard, etc...
- To replace the current tagging system by a canonical, serializable one.
- To deprecate console specific commands such as `Set Printing Width`,
  `Set Printing Implicits`, etc...

This proposal provides a cleanup of the code base. Current stats for
the patch is 1000 additions vs 1500 deletions, making a the codebase
500 lines smaller.

For instance, serialization of `Pp.std_cmdpps` has allowed to
implement `coqtop` as a feedback client, has removed any dependency on
console code from the core of Coq, and generally improved the
modularity of the system.

We aim to preserve byte-identical output. This means that some more
advanced features, such as conditional printers or layout-based
notation, will be implemented separately, as I don't see a way to
preserve compatibility.

**the current printing path**

Coq does:

```ocaml
Feedback.msg_* : std_cmdpps -> unit =>
  if `log_via_feedback` then
    render_to_richpp;  [kinda hacky, can't use tag content]
                       [This forces settings like printing width!]
    send_feedback
  else
    dispatch_to_logger [emacs/color/normal]
    [duplicated path for colors, different tag system from `Pp`]
```

Clients do:

- _console loggers_ => call `Pp.pp_with` to convert `std_cmdpps` to
  `Format` commands. It uses the options
- _feedback_clients_ => custom render of `Pp.richpp`, mostly removing
  useless boxing.

**proposed printing path**

Coq does:

```ocaml
Feedback.msg_* : std_cmdpps -> unit =>
  send_feedback
```

Clients render `std_cmdpps`. Coq provides standard renderers to
`out_channel`, `string`, (maybe `rich_string` == string with colors?)
and `html`. Compatibility is preserved by `string`.

Clients chose rending options such as printing width, etc...

## Use cases and motivation:

We detail some concrete use cases to illustrate the approach:

- **Client-side reflowing**: CoqIDE/jsCOQ can now easily adapt display
  width without a roundtrip. They will receive/store the message in
  `Pp.std_cmdpps` formats, and when their UI refresh function is
  called they can just use the renderer. CoqIDE will likely call the
  `rich_string` renderer, jsCoq can fully delegate to the browser
  engine!

- **Implicit arguments display on hover**: Now implicit arguments are
  rendered but with tag implicit. jsCoq can for instance map the
  implicit tag to a css `display: none` property, but show the
  arguments on hover.

- **Rich notation display**: A common request by IDEs is to
  interpreted notations in a enriched way. Coq can now deliver
  notations to the UI in a way similar to the implicits. Thus the IDE
  can selectively disable notation display or custom render notations.

- **Accurate term location**: Every box can contain a path on the term.

- **Rich document types**: We can easily support tables, paragraphs,
  emphasis on text, math, etc...

### A hard problem:

_Mapping terms to documents and viceversa_

Indeed, the ideas in `Ppannotation` is right, however, I the
implementation seems quite difficult to use, in particular because the
content of the tags must be serialized again.

SerAPI can indeed display the full annotated output, I recommend doing
it and observing what you get. Go to https://x80.org/rhino-hawk/, wait for the libraries to load, then type:

```lisp
(Control (StmAdd () "Require Import Coq.Init.Prelude. Goal forall n, n + 0 = n. intros n."))
(Control (StmObserve 4))
(Query ((pp ((pp_format PpAnn)) )) Goals)
```

My impression is that the current mix of richpp and annotations is too
disjoint to be useful (you get either a string or the full blown
`constr`, IDEs want something _in the middle_). So in a sense, what it
is below is a reification of such "PpAnn" output.

### Example of current `richpp` output vs desired output.

Current:
```lisp
Eval hnf in 2 + 2.
(Element
 (_ ()
  ((PCData " = ")
   (Element (constr.reference ()
            ((PCData "S"))))
   (PCData " (1")
   (Element (constr.notation ()
            ((PCData" +"))))
   (PCData" 2)\n : ")
   (Element (constr.reference ()
            ((PCData nat)))))))
```

Desired [simplified]:
```lisp
(hovbox
 (appbox
  (tag cons "S")
  ((notbox
     (infixbox "+"
       (tag num 1)
       (tag num 2))
     (appbox
       (tag ref "addn")
       ((tag num 1)
        (tag num 2))))))
  (tag null " : ")
  (tag type "nat"))
```

### More on notations:

Very interesting comments happened in:
https://github.com/coq/coq/pull/289

In particular, the discussion suggests adding a new experimental
`ENotation` printing-only command, that would indeed define a new notation domain.

Example:
```coq
ENotation sum := "\sum_{$1 in #2} F $1".
```
or
```coq
ENotation sum := "mathML stuff".
```

## Summary of plan:

Implementation of the plan has started, current code passes the test
suite and can be found at:

https://github.com/ejgallego/coq/tree/pp_new

Already in the tree are:

1. Remove current tagging mechanism (PR#185)
1. Make `Pp.std_cmdpps` serializable. Clean up the
   logger vs feedback distinction, fully encapsulate console access.
1. Deprecate richpp.
1. Experiments with client-side printing width. [more work needed]

Future steps:

1. Implement richer tagging system: first example, output implicit
   arguments tagged in a convenient way. (This can still be disabled
   for performance).
1. Add notation support to `Pp.std_cmdpps`. Don't remove `Ppextend`
   for the moment, but provide an experimental output and binder system.

