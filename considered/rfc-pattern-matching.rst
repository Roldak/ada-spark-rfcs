- Feature Name: pattern_matching
- Start Date: 2020-07-15
- RFC PR: `#50 <https://github.com/AdaCore/ada-spark-rfcs/pull/50>`_
- RFC Issue: (leave this empty)

Summary
=======

We extend case statements and expressions to support `patterns` inside choices
of alternatives. A pattern is an expression that represents a blueprint for a
value of the matched type. It might contain wildcards, which can match any
subexpression of the corresponding type.

Patterns will encompass the current abilities of Ada's case statements and
expressions. As opposed to Ada's current case functionality, the user will be
able to pattern match on a value of any type, be it elementary or composite.

We also provide the possibility of binding values matched by a subpattern to a
new name as part of the matching. This name can then be used in the statements
or expressions of the corresponding alternative.

Motivation
==========

Pattern matching as we describe in this document is a feature that exists in
many programming languages already. It is slowly gaining traction in more
programming languages because it is a very expressive way to match against
values that can have different heterogeneous shapes, in a concise and complete
way.

We believe that it is a very expressive tool for the modern programming world,
as well as a great fit for Ada, which already has pretty expressive ways of
constructing data with heterogeneous structure, via discriminated types and
tagged types.

Pattern matching is in-line with the Ada philosophy because it reduces the
surface where errors can happen in complex type/value/structure matching code,
making the code simpler to read and understand, as well as giving the compiler
more tools to help the programmer.

Guide-level explanation
=======================

Pattern matching can be used to select the execution of a number of
alternatives, depending on the form of an expression of any elementary or
composite type. It takes the form of a disjunction of cases, corresponding to
different patterns.

Matching scalar types
---------------------

For scalar types, a pattern can be either a static literal, a range, a subtype,
or a wildcard, represented with the ``<>`` box notation. We say that an expression
matches a pattern if it is included in the set of values represented by the
pattern, with the wildcard matching any values of the type. For example, we can
write a match on a discrete type as follows:

.. code-block:: ada

  function Increase (X : Integer) return Integer is
  begin
    case X is
      when Negative     => return 0;
      when 0 .. 100     => return 2 * X;
      when Natural'Last => return Natural'Last;
      when <>           => return X + 1;
    end case;
  end Increase;

A few other things of note:

- All choices in a pattern must be static (literals, bounds of ranges, and
  subtypes).

- Patterns that lead to a similar treatment can be grouped together using the
  ``|`` connector.

- It is possible to match several values at once, by grouping them together in
  an aggregate like notation. All values do not need to have the same type.


.. note::

   ``others`` is still a valid wildcard at the top level of a match, when
   standing alone in a when branch, for backward compatibility reasons but
   cannot substitute to the ``<>`` wildcard in general.

Here is another example:

.. code-block:: ada

  type Sign is (Neg, Zero, Pos);

  function Multiply (S1, S2 : Sign) return Sign is
    (case (S1, S2) is
       when (Neg, Neg) | (Pos, Pos) => return Pos,
       when (Zero, <>) | (<>, Zero) => return Zero,
       when (Neg, Pos) | (Pos, Neg) => return Neg);


.. note:: The "matching several values" feature is here because it's a
    relatively cheap way of working around a particular issue caused by the
    lack of tuples in Ada. It is however our hope that we can add tuples to Ada
    and remove this syntax sugar in the future.

The function ``Multiply`` returns the sign of the result of a multiplication,
depending on the sign of the operands. The connector ``|`` is used here to
group together toplevel patterns, but it can also be used inside a pattern.

Matching composite types
------------------------

For composite types, patterns take a form that mimics aggregates, with
component values that are themselves patterns. It is possible to use
qualification to provide the type of a pattern. In this case, a test is first
executed to check if the selecting expression is in the type, then the
pattern is processed assuming that the selecting expression has the type of the
qualification. Here is an example of code matching an object of an
unconstrained array type:

.. code-block:: ada

  type Int_Array is array (Positive range <>) of Integer;
  subtype Arr_1_10 is Int_Array (1 .. 10);

  Arr : Int_Array := ...;

  case Arr is
    --  Match all arrays of length 3 containing elements 1, 2, and 3
    when (1, 2, 3)                                => null

    --  Match arrays ranging from 1 to 8 whose first two elements are 4
    when (1 | 2 => 4, 3 .. 8 => <>)               => null

    --  Match arrays ranging from 1 to 10 which do not contain zero
    when Arr_1_10'(others => Positive | Negative) => null;

    --  Match other arrays ranging from 1 to 10
    when Arr_1_10                                 => null;

    --  Match every other cases. Equivalent to `when others`
    when <>                                       => null;
  end case;

Note that, since the type ``Int_Array`` is unconstrained, all composite
patterns should be constrained. To use unconstrained patterns, like ``(others
=> 12)``, it is possible to qualify the pattern to a constrained type.

.. note:: We could allow unconstrained patterns too, it remains to be seen
    whether it notably complicates implementation.

Unlike for regular aggregates, whether associations are explicit or not makes a
difference for pattern matching. For a value to match an array pattern which
uses named associations, both the bounds and the values should agree.  On the
other hand, if the composite pattern is positional, the values only are
relevant.

String literals are considered to be positional, so the literal ``"foo"`` will
match all strings equal to ``"foo"``, whether they start at index ``1`` or not.

Records
^^^^^^^

A similar syntax can be used to match records, including discriminated records.
Here is an example:

.. code-block:: ada

 type Opt (Has_Value : Boolean) is record
    case Has_Value is
       when True =>
          Val : Int;
       when others => null;
    end case;
 end record;

 subtype None is Opt (Has_Value => False);

 I : Opt := ...;

 case I is
    when None | (Has_Value => True, Val => 0) => return 0;
    when (Has_Value => True, Val => Negative) => return -1;
    when (Has_Value => True, Val => Positive) => return 1;
 end case;


The case statement returns the sign of an optional value. If no values are
present, ``0`` is returned. The subtype ``None`` is introduced to act as a short
form for the pattern ``(Has_Value => False)``.

.. note:: Pattern matching is seen as particularly useful in the context of
    discriminated records, because it allows safe and complete handling of
    every case, in a fashion that is very close to what is done with sum types
    in functional languages. It is seen as a strictly better way of accessing
    fields whose existence depends on a discriminant, because it cannot fail at
    runtime.

Pattern matching can also be used on tagged types: It is possible to match on
an object of a classwide type. Matching different shapes can be done either
using a subtype pattern, or a qualified composite pattern.

.. note:: Usually, subtypes used as patterns, as well as in qualified
   expressions, should be compatible with the type of the selecting expression.
   However, if the selecting expression is tagged, it is possible to use any
   (possibly classwide) type from the hierarchy, as long as they are
   convertible.

Note that, as derivation trees can always be extended, a default case should
necessarily be used when matching an object of a classwide type. Here is an
example:

.. code-block:: ada

 type Shape is tagged record
    X, Y : Integer;
 end record;

 type Line is new Shape with record
    X2, Y2 : Integer;
 end record;

 type Circle is new Shape with record
    Radius : Natural;
 end record;

 S : Shape'Class := ...;

 case S is
    when Circle'Class'(Radius => 0, others => <>) => Put_Line ("point");
    when Circle'Class                             => Put_Line ("circle");
    when Line'Class                               => Put_Line ("line");
    when <>                                       => Put_Line ("other shape");
 end case;

Note that, unlike regular aggregates, composite patterns can be used for
classwide types. They can contain associations for components which are present
in the root type of the hierarchy. Since potential subsequent derivations might
add components, these patterns should always contain a default case
``others => <>``.

Semantics
^^^^^^^^^
A value of a composite type matches a pattern if every element of the value
matches the corresponding element in the pattern (or the default `others` case
if there is none). In particular, this means that equality on composite types
is never relevant in pattern matching.

Accesses
--------

It is possible to match access objects, along with the value they designate.
A pattern for a non-null access value is represented as an aggregate with a
single component named ``all``. Here is an example:

.. code-block:: ada

 function Add (A, B : Int_Access) return Integer is
 begin
    case (A, B) is
       when ((all => <>), (all => <>)) => return A.all + B.all;
       when ((all => <>), null)        => return A.all;
       when (null, (all => <>))        => return B.all;
       when (null, null)               => return 0;
    end case;
 end Add;

Completeness & overlap checks
-----------------------------

Static checks are done at compilation to ensure that the alternatives of a
pattern matching statement or expression supply an appropriate partition of the
domain of the selecting expression.

Like for regular case statements (or expressions), if the selecting expression
is a name having a static and constrained subtype, every pattern must cover
values that are in this subtype, and all values in the subtype must be covered
by at least one alternative.

Otherwise, alternatives should cover all values that cannot statically be
excluded from the match (ie. all values of the base range for scalars, all
arrays ranging over the base range of the index type for unconstrained or
dynamically constrained arrays etc).

Additionally, if one value ``V`` can be matched by two alternatives then either
one alternative is strictly contained in the other, or there is a 3rd
alternative which is strictly contained in both and also matches ``V``.

Alternatives should be ordered so that an alternative strictly contained in
another appears before.

Alternatives contained in the same ``when`` branch are exempted of the overlap
check.

.. admonition:: design

    Do we want to forbid overlapping of scalar ranges even if they fall in the
    above category?

.. admonition:: design

   It has been considered adopting a more lax strategy akin to
   OCaml's/Haskell's/etc, but the above strategy seems to fit the Ada
   philosophy very well. Also the fact that the rule doesn't exist for
   alternatives in the same branch (via   ``|``) does make the rules expressive
   enough in our opinion.

Binding values
--------------

As part of a pattern, it is possible to give a name to a part of the selecting
expression corresponding to a subpattern of the selected alternative.  This can
be done using the keyword ``as``. Here is an example:

.. code-block:: ada

 case I is
   when (Has_Value => True, Val => <> as V : Integer) => return V;
   when (Has_Value => False) => 0;
 end case;

The name can be used to refer to the part of the selecting expression in the
statements/expression associated with the selected alternative.

A name can be associated to any subpattern as long as the pattern matches only
one value.  In particular, it is not possible to give a name to a pattern if it
is associated with the ``others`` choice in a composite pattern. For example,
the bindings below are all illegal:

.. code-block:: ada

  case Arr is
    when (1 | 2 => 4, 3 .. 8 => <> as V)       => null;
    when (1 | 2 => 5 .. 10 as V, 3 .. 8 => <>) => null;
    when Arr_1_10'(others => Positive as V)    => null;
    when <>                                    => null;
  end case;

In the most common case, when the bound pattern is a wildcard, it is possible to
write ``<V>`` instead of ``<> as V`` for short. For example, the function
``Add`` on access types can be rewritten as:

.. code-block:: ada

 function Add (A, B : Int_Access) return Integer is
 begin
    case (A, B) is
       when ((all => <X1>), (all => <X2>))              => return X1 + X2;
       when ((all => <X>), null) | (null, (all => <X>)) => return X;
       when (null, null)                                => return 0;
    end case;
 end Add;

Note that here, binding values in pattern matching brings additional safety, as
it avoids the use of dereferences.

If a binding is done in one of the members of pattern disjunction (with ``|``),
then the same name should be bound in other members of the disjunction. For
example, the second pattern in ``Add`` is ok because ``X`` is bound in both
alternatives of the disjunction.

The same name cannot be used twice in the same branch.

Reference-level explanation
===========================

This won't be written in the first version of the AI: we're waiting for
feedback from the prototyping phase before we write a low level version of this
AI.

.. note::
    This is the technical portion of the RFC. Explain the design in sufficient
    detail that:

    - Its interaction with other features is clear.
    - It is reasonably clear how the feature would be implemented.
    - Corner cases are dissected by example.

    The section should return to the examples given in the previous section, and
    explain more fully how the detailed proposal makes those examples work.

Rationale and alternatives
==========================

Rationale
---------

The current design is what we believe to be the best compromise to bring a
battle tested feature (pattern matching) to Ada.

We believe that pattern matching, as expressed in this document, is a natural
extension of the matching capabilities of the case statement, which is why it
is possible to subsume the existing feature set with a superset. We also
believe that is brings necessary expressivity and safety to Ada:

* It makes working with heterogeneous data safer, by providing a tool that
  ensures that you can only work on data that has been previously validated by
  the match, where it was previously easy to make mistakes, and no tools short
  of full static analyzers were able to warn you in every case.

* It encourages factorization of the shape testing logic in a way that will
  improve readability rather than hamper it, by allowing the users to focus on
  the non repetitive logic.

This is why we believe that pattern matching is worth the complexity it brings
to the language. Also, we believe that this complexity is pretty local and in
line with the benefits of the feature.

Alternatives
------------

While not strictly an alternative, something that is often compared with
pattern matching is flow sensitive (sub)type narrowing:

.. code-block:: ada

   A : access Integer;
   if A /= null then
      --  A has type not null access Integer
      Put_Line (A.all'Image);
   end if;

   R : Optional_Integer;

   if R.Has_Value then
      --  R has type Optional_Integer (True)
      Put_Line (R.Value'Image);
   end if;

This feature could also be a good fit for Ada, at least for subtypes - it would
be weird to have the type of a value change in a branch. However, we believe
pattern matching to provide most of the benefits, especially if we, in later
revisions, take advantage of irrefutable patterns, which could allow similar
things.

.. note::
    - Why is this design the best in the space of possible designs?
    - What other designs have been considered and what is the rationale for not
      choosing them?
    - What is the impact of not doing this?
    - How does this feature meshes with the general philosophy of the languages ?

Drawbacks
=========

The complexity of the feature - and the implementation price - should obviously
be considered a drawback for any added feature. In the case of pattern
matching, the complexity is pretty big - but so is, we believe, the benefit.

The complexity is also pretty well contained to the case statement.

.. note::

   If anybody has legitimate reasons that are not variations of "Ada is too
   big" or "I don't see myself using this feature", please share!

Prior art
=========

There is a of wealth of prior art related to pattern matching, because a very
big proportion of languages now include pattern matching or something very
closely related. Worth mentioning are:

- OCaml and Haskell's pattern matching are very similarly flavored, and can be
  considered the "reference" today, as they stick very closely to the original
  pattern as expressed in ML, which is the basis for the feature set that you
  can find in many languages today. See `here for a description of Haskell's
  pattern matching <https://www.haskell.org/tutorial/patterns.html>`_.

.. note:: It is worth mentionning that pattern matching in different forms found
   itself in programming language even earlier:

   * COMIT and `SNOBOL <https://en.wikipedia.org/wiki/SNOBOL>`_ have a form of
     pattern matching, although limited to strings, and thus more akin to
     regular expressions.

   * `Refal <https://en.wikipedia.org/wiki/Refal>_` is one of the first
     languages with generalized structured pattern matching.

   * Prolog introduced a limited form of structural pattern matching in the
     logic programming context.

- Rust and Swift both have pattern matching that is very similar to the ML
  family pattern matching.

Amongst the list of languages currently considering pattern matching

- Java has introduced a limited form of pattern matching in Java 14, and is
  considering expanding it further to support full composite type matching.

- Python has a recent RFC for `pattern matching
  <https://www.python.org/dev/peps/pep-0622/>`_ that is garnering a lot of
  support from the language design team.

.. note:: expand if needed ?

.. note::
    Discuss prior art, both the good and the bad, in relation to this proposal.

    - For language, library, and compiler proposals: Does this feature exist in
      other programming languages and what experience have their community had?

    - Papers: Are there any published papers or great posts that discuss this? If
      you have some relevant papers to refer to, this can serve as a more detailed
      theoretical background.

    This section is intended to encourage you as an author to think about the
    lessons from other languages, provide readers of your RFC with a fuller
    picture.

    If there is no prior art, that is fine - your ideas are interesting to us
    whether they are brand new or if it is an adaptation from other languages.

    Note that while precedent set by other languages is some motivation, it does
    not on its own motivate an RFC.

Unresolved questions
====================

 - Which semantics should we use for binders? If we consider them as renamings,
   it would be possible to update the underlying structure through a binder.
   However, it would no longer be possible to bind parts of an object which
   might be erased (components of a variant part of a record with mutable
   discriminants in particular). We could possibly have both with a different
   syntax. For example, the constant keyword could be used to state that we want
   copy semantics, not a renaming:

.. code-block:: ada

     case A is
       when (Has_Value => True, Val => <> as constant V) => return V;
       when None                                         => return 0;
     end case;

.. note::
    - What parts of the design do you expect to resolve through the RFC process
      before this gets merged?

    - What parts of the design do you expect to resolve through the implementation
      of this feature before stabilization?

    - What related issues do you consider out of scope for this RFC that could be
      addressed in the future independently of the solution that comes out of this
      RFC?

Future possibilities
====================

This AI has been purposedly contained to the very basics of what pattern
matching can offer while still remaining useful. There are many possible forays
into making pattern matching in Ada more powerful in the future:

Conditional guards
------------------

A lot of languages offer the possibility of restricting the match of a branch
to a case where a runtime boolean predicate is satisfied:

.. code-block:: ada

   case Point is
      when (0, 0) => ...
      when (<X>, <Y>) if X < Y => ...
   end case;

This is useful and expressive, but overlap checks would have to be adapted, so
we didn't try to include it in the first version.

Custom matching for private types
---------------------------------

Ada relies very heavily on encapsulation via private types, which doesn't mesh
well with private types, which is why there is no facility for matching private
types except via a wildcard. This is why providing facilities to allow matching
of private types would be great.

There are existing functionalities in other languages such as:

- F#'s `Active patterns <https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/active-patterns>`_
- Scala's `Extractor objects <https://docs.scala-lang.org/tour/extractor-objects.html>`_

It is one of the main next goals of the working group on pattern matching to
investigate such facilities to make usage of pattern matching in Ada easier.

Irrefutable patterns
--------------------

An irrefutable pattern is a pattern that never fails to match. For example, given a simple record type:

.. code-block:: ada

   type Point is record
      X, Y : Integer;
   end record;

The pattern ``(<X>, <Y>)`` can never fail. Using irrefutable patterns might allow many interesting possibilities like:

- Destructuring assignment/object declaration

.. code-block:: ada

   (<X>, <Y>) := P;

- Destructuring in formals/parameters
- Destructuring in for loops:

.. code-block:: ada

   for (<X>, <Y>) of Point_Array loop
      ...
   end loop;

Sealed tagged hierarchies
-------------------------

Having sealed tagged hierarchies - while having a ton of other benefits for OO
in a low level language, like definite size - will make it easier to use
pattern matching, because the ``others`` clause won't be necessary anymore:

.. code-block:: ada

   type Maybe is tagged null record;
   type None is new Maybe with null record;
   type Some is new Maybe with record
      Val : T;
   end record;


   -- Without sealed classes:
    case I is
       when Some'(12) => ...
       when None => ...
       when others => ...
    end case;

   -- With sealed classes:
    case I is
       when Some'(12) => ...
       when None => ...
    end case;
