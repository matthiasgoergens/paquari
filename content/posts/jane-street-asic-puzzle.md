+++
title = "Star Battle in silicon: Jane Street's ASIC puzzle"
date = 2026-09-05T12:00:00+08:00
description = "Jane Street published the mask layout of a small chip and asked what input makes its success pin go high. The chip checks an 11×11 two-star Star Battle. I extracted the netlist from the polygons and the region map from the netlist; my algebra briefly proved the puzzle unsolvable, and a floating wire corrupts one of the chip's consolation messages."
+++

*Submissions for the puzzle closed on 4 September 2026. Everything below is a
spoiler.*

In August 2026 Jane Street's hardware team published [a puzzle][blog]: a GDS
file, the format in which a chip's mask geometry is sent to a foundry, plus
an example waveform, an annotated picture of the layout, and a warm-up design
with full source. The job was to recover a netlist from the polygons, work
out what the circuit does, and find the input that makes its `success` pin
go high. The chip prints the answer when `success` goes high.

The chip is a checker for an 11×11 [Star Battle][sb] with two stars per row,
column and region. It reads the grid in as 121 serial bits, and when the grid
is right it prints `(* TWO STARS *)`, which is both the rule of the game and
an OCaml comment. Comparing notes with other solvers afterwards exposed two
things I had missed. The scripts are in a repository that I will link here
once it is public.

[blog]: https://blog.janestreet.com/can-you-reverse-engineer-an-asic/
[sb]: https://en.wikipedia.org/wiki/Star_Battle

## Hints in the supplied files

The repository provides five items, each containing a hint.
The blog says the circuit is *physically arranged to hint at its
functionality*, that one block of the design generates the output but does
not affect `success`, and that you should toggle `rst_n` (the reset, active
low) before each attempt. The physical placement is a clue, the output block
can be ignored at first, and the input protocol is stateful.

The example waveform is a VCD file (a value change dump), which is plain
text. Its `$version` field says "Leave no stone unturned!", its `$date` is
`Sat Dec 31 23:59:60 2016`, the leap second at the end of that year, and once
the 121-bit input stream ends the eight-bit output bus changes one byte per
clock: `0x54 0x52 0x59 0x20 …`, which spells `TRY AGAIN`. The output is a
byte stream that taunts wrong inputs. Each `enable` window is exactly 121
clocks long.

The input bits are talking too, though I only noticed much later. Split
either 121-bit window into eleven groups of eleven; the first seven bits of
each group, read least-significant bit first, are seven-bit ASCII and the
other four are zero. The first attempt decodes to `The night s` and the
second to `ky awaits` followed by two spaces.

## The netlist is in the polygons, and the cell library is on the web

A GDS of a synthesised design is a sea of polygons, but synthesis leaves an
enormous amount of structure intact. Every standard cell survives as a named
instance (`sky130_fd_sc_hd__and2_2` and friends) and every pin of every cell
carries a text label. There is no transistor-level work to do. The task is
to work out which pins are wired together, work out what each standard cell
computes, and simulate.

The connectivity extractor is a page of Python, using [gdstk][gdstk] to
read the GDS and [shapely][shapely] for the 2D geometry. Flatten the layout;
polygons on the same metal layer that touch are the same net; the contact
and via layers bridge adjacent metals; union-find over everything; then look
up each pin label's point to find its net. Getting it right took three
attempts.

The mirror came first. Standard-cell rows alternate orientation and GDS
marks flipped cells with a flag called `x_reflection`. I negated x. It means
reflection *about* the x-axis, which negates y. Half the pin lookups landed
in empty space until I compared one label's position with the shape
underneath it.

With the transforms fixed,
every pin found a shape, but the simulation was all zeros: every net had
fanout one, each pin in its own private component. Tracing a clock pin, I
found the routing simply stopped. A metal-1 stub ended 0.17 µm short of the
wire it obviously should reach, and gaps of exactly 0.17 µm appeared
everywhere, as if the shapes had been nibbled. The cause was that gdstk's
`Cell.flatten()` silently drops polygons: `get_polygons(depth=None)` returned
17,871 polygons for the warm-up, `flatten()` produced 16,817, and the missing
thousand were mostly the metal-1 segments that connect pins to wires.

The third problem hid inside a passing test. I had hand-written truth tables
for the cells, and the warm-up, a comparator that checks whether two numbers
sum to 496 (the third perfect number, which is presumably the joke), comes
with full source and a known netlist. It passed 52 of its 54 test vectors.
The two failures were the only two vectors whose expected output was 1. A
broken comparator outputs 0 everywhere, so a validation set with mostly-zero
answers is a poor validation set. The bug was a guess about naming: `and4bb`
inverts its *first* two inputs, `A_N` and `B_N`, not its last two. I stopped
guessing and replaced the hand-written truth tables with the functional
Verilog that the [SkyWater PDK][pdk] publishes for every cell: each model is
a handful of gate primitives, and a fifty-line interpreter turns them into
evaluation functions. From then on cell behaviour was correct by
construction.

With the warm-up at 54 of 54, the puzzle extracted to 728 instances of 66
cell types, 741 nets, and no unresolved pins. I then tested the extracted
netlist behaviourally by replaying the example VCD's stimulus through it
before analysing a single gate. Out came
`TRY AGAIN`, twice, byte for byte, with `success` low, exactly as recorded.

[gdstk]: https://github.com/heitzmann/gdstk
[shapely]: https://shapely.readthedocs.io/
[pdk]: https://github.com/google/skywater-pdk

## Recovering the checker logic

Ninety-two flip-flops. Twenty-one multiplexers, twelve of which share one select net and form a chain with the
serial input at its head: a 12-bit shift register, gated by something. Four
flip-flops nearby count 0 to 10 and wrap: a modulo-11 counter. And 121 clocks
per attempt is 11 × 11. The input is an 11×11 grid, shifted in row by row.

Then the expressions got big. The cone of logic feeding `success` expands
into pages of AND-OR-invert gates, some subexpressions hundreds of terms
long. I tried simplifying them by hand and produced a proof that the puzzle
was unsolvable: one check appeared to force column 10 to be all ones while
another forced it to contain exactly two ones. Assuming the puzzle was
solvable, I treated the contradiction as an algebra error and probed the
netlist with the simulator instead.

The giant expressions were all functions of just eight flip-flops, a 4-bit
position counter and a 4-bit round state machine. I forced those eight bits
to each of their 121 reachable values and evaluated the logic to produce
truth tables. The row-checking logic, three flip-flops that looked
parity-like, resisted hand analysis. Enumerating all 2,048 possible 11-bit rows through
the simulator took seconds and settled it: exactly the 55 rows with two ones
are accepted. My algebra had said odd parity; the chip wants exactly two
stars per row.

The remaining state is 22 pairs of flip-flops, each pair a small saturating
counter. Probing each pair's update condition over the 121 (round, position)
states records which grid cells it watches. Eleven pairs each watch one
column. The other eleven watch irregular blob-shaped sets of cells: 28, 21,
14, 11, and so on down to 4, and the sizes sum to exactly 121. It is a
partition. Here it is, with one letter per region:

```
AAAAABBCDDE
AAFAABCCDDE
AAFBBBBCCDE
AAFBGGGECCE
FAFBGEEEEEE
FFFBGGGEHHH
BBBBBBGEHII
BJJJGGGEHII
BJJKEEEEHII
BBJKKEEEHHH
BJJKEEEEEEE
```

Eleven regions, each required to contain exactly two stars; two per row; two
per column; and, from the 12-bit history register watching the last two rows'
worth of taps, no two stars touching, even diagonally. Those are the rules
of Star Battle. The column counters and the region counters sit in two neat
rows on the die. The output generator is the big block on the right that the
blog said to ignore.

## Solving

I solved the resulting constraint problem by backtracking over row patterns
with column and region budgets and an adjacency check. The puzzle has
exactly one solution, which the same search confirms by running to
exhaustion:

```
.......*.*.
*....*.....
.......*.*.
*.*........
....*.*....
..*.....*..
....*.....*
.*....*....
...*......*
.....*..*..
.*.*.......
```

Clock those 121 bits in, row by row with `enable` high, drop `enable`, and
keep clocking. `success` rises on the next clock and the output prints
`(* TWO STARS *)` followed by a zero byte.

## The output generator has five messages, and one of them is broken

The block I had ignored turned out to have more in it than `TRY AGAIN`. An
all-zeros input prints `EMPTY SKY`, an all-ones input prints `BIG BANG`, and
the real answer prints the OCaml comment. I found those three by enumerating
inputs. The fourth I only learned about on the day submissions closed, from
[sunaabh's write-up][sunaabh] (public since 18 August; I had not gone looking
for other people's solutions before the deadline) and [atx's
write-up][atx]: an input in which every row, column and region has two stars
but two stars touch prints `TWO NOT TOUCH`, the name under which the New York
Times publishes this game. Swapping the first two rows of the solution keeps
every count intact while making two stars adjacent, and my netlist selects
the same message, though not letter-perfect. My input enumeration had
covered the degenerate cases and missed this near miss.

The imperfection is the interesting part. One net in the extracted design
has no driver. It feeds only pin `A1` of two cells in the output
block, an `a31oi` and an `a311o` sitting next to each other near x = 178 µm,
y = 92 µm, and no output pin of any cell reaches it. Every other net in the
design has exactly one driver, so this is not an extraction artefact.
[jestoph][jestoph], a solver posting under that name, found the same wire by
visual inspection (his post describes a wire connected only to two input
pins, with a neighbouring connection that lands on nothing) and reported it
to Jane Street, who confirmed it as a bug.

The net reaches only two output bits, and forcing it
to either value leaves `success` and the other four messages unchanged. But
the message it does touch cannot be printed correctly with the net held at
either value. At 0 the chip emits `TWO"NOT TOUCH` (byte four is `0x22`
rather than a space); at 1 it emits `TWO NOT TOUCJ` (the last byte is `0x4A`
rather than `0x48`). Printing it correctly needs the net high during the
fourth byte and low during the thirteenth, so the intended driver was
presumably a timing signal whose route never reached the pin. On fabricated
silicon the input floats, and unless it happens to change value between
those two bytes the message comes out misprinted one way or the other. My
simulator treats an undriven net as 0, which is why the first spelling is
the one I observed. The solvers who used formal tools reported a clean
message, which is what I would expect if their tools treat an undriven wire
as a free variable at every time step.

[sunaabh]: https://sunaabh.com/systems/2026/08/18/jspuzzle.html
[atx]: https://atx.name/electronics/asic-re/
[jestoph]: https://jestoph.com/2026/09/04/jane-street-challenge.html

## How other people did it

The public write-ups I know of took four routes to the same answer.

[jestoph][jestoph] built a circuit simulator, a hardware description
language, a test harness and half a waveform viewer before reading any
documentation, then solved the input backwards with [Z3][z3], an SMT solver,
one wire at a time. [sunaabh][sunaabh] used the Python API of
[KLayout][klayout], a layout viewer, to extract the netlist, emitted Verilog,
and asked the SAT solver built into [Yosys][yosys], an open-source synthesis
tool, for an input that makes `success` high 121 cycles after `enable`. The
solver posting as [atx][atx] did the extraction with gdstk and shapely as I
did, then handed the whole problem to [SymbiYosys][sby], a formal
verification front end for Yosys, with a single `cover(success)` statement,
which asks the tool to find any input sequence that reaches that state; a
solution came out in under two minutes. [Karel Peeters][karel] used
KLayout's `LayoutToNetlist` extractor, unrolled the circuit into about
100,000 boolean variables and equations over 122 time steps, and solved them with Z3 in
under a second.

The solver route finds the answer
without the region map; atx notes that recovering the extra constraints
"would now require actually reverse engineering the circuit subcomponents",
which is what the counter-pair probing above does. The probing route, in my
hands, missed `TWO NOT TOUCH`, which the solver route found by enumerating
inputs.

[z3]: https://github.com/Z3Prover/z3
[klayout]: https://www.klayout.de/
[yosys]: https://yosyshq.net/yosys/
[sby]: https://symbiyosys.readthedocs.io/
[karel]: https://gist.github.com/KarelPeeters/dba417c2690cf0505ac9079ca1c609ab

## Easter eggs

- **Morse code below the die.** Two cell types, `INTERNAL_3` and
  `INTERNAL_7`, each contain a single rectangle and sit in a row fifty
  micrometres below the nominal die, which is why the GDS bounding box
  extends below y = 0. The narrow and wide bars, with gaps in the ratio
  1:3:7, encode Morse: `PER ARENAM AD ASTRA`, "through the sand to the stars", a silicon
  variant of *per aspera ad astra*.
- **The logo.** Jane Street's concentric-circle mark is drawn in a metal
  layer, in both the puzzle and the warm-up. A full-chip render hides it among
  the routing; one layer viewed alone shows it.
- **The waveform.** A hint in the version field, a leap second in the date,
  and `The night sky awaits` in the input bits.
- **The messages.** `EMPTY SKY`, `BIG BANG`, `TWO NOT TOUCH`, `TRY AGAIN`,
  `(* TWO STARS *)`: an astronomy progression that ends in an OCaml comment.
- **496.** The warm-up's constant is the third perfect number. That one is
  an interpretation rather than a labelled payload.

Building the netlist took a few days. I validated every tool on the warm-up
before pointing it at the puzzle, including test vectors whose right answer
is 1, and replayed the recorded waveform through the extracted netlist
before analysing a single gate.

I have since built another puzzle chip, [grand-finale][gf]. It has a serial
input and reports its verdict on an eight-bit bus, and it comes with a
sealed envelope.

[gf]: https://github.com/matthiasgoergens/grand-finale
