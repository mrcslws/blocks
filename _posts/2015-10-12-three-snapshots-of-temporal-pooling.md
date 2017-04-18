---
layout: post
title: "Three snapshots of Temporal Pooling"
date:   2015-10-12 12:00:00
---

_(I'm writing this for me. Feel free to enjoy it.)_

I like to build a playful understanding of just about everything I
work with. I'm trying to hold a bunch of information in my head until
it no longer feels heavy to carry.

Let's explore three points in how-TP-might-work space. Each of these
points is a snapshot of Comportex. I'm here to discuss approaches, not
results. I'll [explain](https://xkcd.com/106/) why each approach
thinks it's the right approach.

## Terminology

Let's use the most up-to-date terminology, even when discussing old
code.

- "winner cell"
  - It's selected by the cell being predictive in the previous
    timestep, the cell staying active due to temporal pooling, or as
    the best candidate in a bursting column.
- "learning"
  - A cell or column that will learn.
- "learnable"
  - A cell or bit that can be learned by a "learning" cell/column.
- "stable"
  - An active input bit that was predicted

## Episode 1 - Long-lasting activations

[Link to snapshot](https://github.com/nupic-community/comportex/tree/779374d276ae15174d16c5ff5deab501f1f699e4)

Felix elegantly
[explained](http://floybix.github.io/2014/09/26/temporal-pooling-mechanism)
how this worked. Check out the "Mechanism" section.

This approach was minimalistic. All it does is keep certain cells and
their columns active longer. This long activation means the
cells/columns participate in more learning, using the normal rules.

...and that's the approach.

**Proximal learning:** During the long activation, active columns
strengthen and weaken incoming proximal connections. _(In fact, I
think it did a bit too much weakening. When a column was selected for
its TP excitation, *and* it also had input bits, it would weaken
connections to other input bits. This may have been solved later, so
let's make sure to revisit this in a later episode.)_

**Distal learning:** On each timestep, the cell strengthens or extends
a distal segment, and the cell continues to be a candidate for other
cells' learning.

**Column activation:** Predicted input bits cause columns to get a TP
excitation. This TP excitation decays on every timestep, and can be
renewed by inputs. On every timestep, each column's excitation is the
maximum of its TP excitation and the excitation from its inputs. A
column loses its TP excitation when it is beaten.

**Cell activation:** Whenever a column is activated (or held active)
by its TP excitation, it chooses the first winner cell in the column
as the active cell.

### Why it might work

Will this approach recognize a sequence if it doesn't see it from the
beginning? Maybe, depending on the nature of the sequence. The set of
patterns in a sequence with the strongest activation effect on the
layer could create a sort of "sequence fingerprint". This is similar
to
[why the Coordinate Encoder probably works](http://mrcslws.com/gorilla/?path=hotgym.clj#probabilistic). If
that's not good enough, we could also change it so that distal
connections to lateral cells assist in activating cells. Then all that
distal learning will serve the purpose of retrieving the full sequence
SDR even when it misses the beginning.

But we might be okay without any of that. Bigger hierarchies could
solve all of our problems. Think of a hierarchy of 3 single-layer
regions. The second region might show different representations based
on its starting point in the sequence, but all of these
representations will be similar. The top region will spatial pool over
the middle region, and it will successfully name the sequence. _(It's
interesting how the spatial pooling in a higher region manages to
amplify temporal pooling.)_

## Episode 2 - Proximal segments

[Link to snapshot](https://github.com/nupic-community/comportex/tree/0ae54788dfb2b514d679d2269066f6041d8ee542)

Lots of details changed (and an entire year passed) between Episode 1
and this snapshot. At a high level, though, there was one big change.

As a column stays active, it forms new proximal synapses, and those
synapses are divided between multiple proximal segments.

**Proximal learning:** Active columns always do proximal learning. If
there's already a good segment that matches the input pattern, this
segment does the learning -- its synapses connected to active bits get
strengthened, with extra strength if the bit was predicted, and those
connected to inactive bits are weakened. Otherwise we add new
synapses, possibly to a new segment, though we only grow synapses to
active bits that were predicted (except in the lowest layer). In other
words, the column pretty much always learns to better recognize the
input pattern. (Just not in the case where there's not a matching
segment and the input pattern doesn't contain enough predicted bits.)

**Distal learning:** Essentially same as Episode 1.

**Column activation:** Mostly the same as Episode 1, with three
changes: (1) The TP excitation accumulates rather that being replaced,
up to a certain maximum. (2) The TP excitation is not lost when the
column is beaten. It just continues to decay. (3) Bursting input
causes the TP excitation to decay faster.

**Cell activation:** Essentially same as Episode 1.

### Why it might work

This addresses the worries from Episode 1. It's designed to learn each
pattern in a sequence on proximal connections. Those patterns are
split between segments, so we don't run into the problem of capturing
undesired patterns.

As before, there is no layer-wide Temporal Pooling state. In other
words, it works well with topology, less tied to the idea of discrete
regions (and layers). Parts of a layer might be stable, with predicted
input bits, while other parts might have lots of bursting.

## Episode 3 - Learn sequences. Accumulate cells.

[Link to snapshot](https://github.com/nupic-community/comportex/tree/d2435e8339c5e6b51ae0f3470ae88a77b00916df)

Two major changes:

- A focus on learning sequences in higher levels.
  - All column activations get a high TP excitation.
  - A cell only does distal learning on the timestep that the it was
    activated.
- Activation levels are now variable. Higher layers gather a union of
  recently activated cells.

To make all of this work, a new layer-wide concept is introduced:
"engaged". When a sufficient amount of the input pattern is stable,
the layer is considered engaged. So the layer is either "newly
engaged", "continuing engaged", or "not engaged". When a layer is
newly engaged, it clears every column's TP excitation and starts
afresh from the current input. As a layer stays engaged, it grows its
activation level at a constant rate, up to a certain maximum.

**Proximal learning:** Similar to Episode 2, but only learn proximally
when the layer is engaged.

**Distal learning:** A winner cell only learns in the timestep that it
becomes active. It does not continue learning as it stays active. It
does remain _learnable_ as it stays active.

**Column activation:** Whenever a column is activated, its active
cells are selected, those cells' TP excitation gets set to the max,
and this excitation is used to calculate subsequent column
activations. Whenever a layer engages, the current TP excitations are
forgotten, and the current set of active cells are treated as newly
activated (i.e. their TP excitation is set to the max). The TP
excitation decays at a constant rate. Engaged layers incrementally
grow their activation level up to a certain maximum, so a newly
activated column does not necessarily inhibit other columns.

**Cell activation:** Essentially same as Episode 2.

### Why it might work

Higher layers now behave like lower layers, just over longer periods
of time. The distal synapses no longer seem awkward like they did in
Episodes 1 and 2 -- they're aimed at sequence prediction again. This
has a good "unifying theory" feel.

The "union pooling" addition solves the problem of "The layer fixates
on the first few patterns in a sequence, freezing the representation
and failing to represent distinguishing patterns later in the
sequence."

One potential enhancement that Felix mentioned: when a layer becomes
disengaged, reset the TP excitations again and start growing the
activation level. So the unrecognized patterns get their own union
representation in the columns. Just don't do proximal learning. As a
result, this layer provides more useful information up and down the
hierarchy (feedforward and feedback).

## Some questions

I haven't experimented with any of this. Here are a few explorations
that I think might be fun / fruitful.

- Explore the lengths of sequences that columns end up representing.
  - What is the impact of the proximal segment limit?
  - What is the impact of the TP excitation decay rate?
  - Does it depend on how quickly the sequences are changing?
- Tell the story of what happens when a column's temporal pattern is
  in view for a long period of time. Something like: when it appears,
  the column activates and has a high TP excitation. Eventually this
  excitation decays to 0, but the column stays active because its
  pattern is still incoming. But it's likely to get inhibited due to
  other columns getting TP excitations. Then it reactivates again soon
  afterward, renewed. Show what actually happens. Decide whether it's
  reasonable.
