---
layout: post
title: "Appendix: The classic TP"
date:   2016-05-05 12:00:00
---
<script src="//d3js.org/d3.v3.min.js" charset="utf-8"></script>
<script src="/stuff/segment-stories.2016.04.28.js?1" charset="utf-8"></script>
<style type="text/css">
      text {
      font: 10px sans-serif;
      }

      .axis path {
      fill: none;
      stroke: none;
      shape-rendering: crispEdges;
      }

      .crispLayers .layer {
      shape-rendering: crispEdges;
      }

      .axis line {
      stroke: none;
      shape-rendering: crispEdges;
      }

      .showAxis path {
      stroke: black;
      }

      .y.axis line {
      stroke: none;
      }

      .noselect {
      -webkit-touch-callout: none;
      -webkit-user-select: none;
      -moz-user-select: none;
      -ms-user-select: none;
      user-select: none;
      }

      .clickable {
      cursor: pointer;
      }

      .draggable {
      cursor: -webkit-grab;
      cursor: -moz-grab;
      cursor: grab;
      }

      .dragging .draggable,
      .dragging .clickable {
      cursor: -webkit-grabbing;
      cursor: -moz-grabbing;
      cursor: grabbing;
      }
</style>

This is a quick followup to
[The Life and Times of a Dendrite Segment](/blocks/2016/04/28/life-and-times-of-dendrite-segment.html).

## The original: TP.py

Here are segment lifetimes in the classic version of Temporal Memory,
[TP.py](https://github.com/numenta/nupic/blob/bf38147eb2215b5243d123e411133a8e0ff6d3c9/src/nupic/research/TP.py).

Here I performed the same experiment:

1. Feed an HTM some data with sequencial patterns
   ([hotgym](https://github.com/numenta/nupic/blob/master/src/nupic/datafiles/extra/hotgym/rec-center-hourly.csv)).
2. Feed it random garbage.
3. Feed it the first data again.

<div style="position:absolute; left:0; right:0;">
<div id="experiment1" style="width:950px;margin:auto;position:relative;"></div>
</div>
<div style="height:725px;"></div>
<script>
insertCharts3(document.getElementById('experiment1'),
      "/stuff/2016.05.03.experiment1_column_states.csv",
      "/stuff/2016.05.03.experiment1_segment_lifetimes.json"
      );
</script>

_______


<br/>Notes:

- This doesn't show segment matches (light green). `TP.py` doesn't compute
  these, it only computes matching segments for active columns (dark green).
- For those of you familiar with `TP.py`: I've disabled backtracking, sequence
  length limits, "pay attention mode", and global decay. So it's similar to pure
  Temporal Memory, but it still has differences, e.g. "start cells" and
  implementing "winner cells" differently, not actually bursting unpredicted
  columns but instead choosing a single cell immediately. Since we're never
  activating all cells in a column, segments are less likely to be matching, so
  we're more likely to add new segments. This is one reason there's a higher
  segment learning rate, but there may be other reasons I'm not thinking of.

_______

<div style="height:16px;"></div>

Success! It keeps itself tidy, and it doesn't discard its useful segments when
it sees white noise. Looking at it from this angle, this implementation of
segment cleanup is superior to that of the new `TemporalMemory.py` /
`TemporalMemory.cpp`.

## The segment cleanup rules

- When a segment's synapse count falls below the activation threshold, the
  segment is removed.
  - ...but with global decay disabled, there's no learning rule that would cause
    this to happen. Incorrect predictions are not punished.
- When a cell exceeds its maximum allowed number of segments, it removes the
  segment with the lowest
  [duty cycle](https://en.wikipedia.org/wiki/Duty_cycle).

So this all depends on how we calculate the duty cycle.

Every time a segment is created or activated, it is considered active for this
timestep. A segment's duty cycle is, roughly, the fraction of timesteps it is
active.

TP.py uses this formula:

$$
D_{t} = (1 - \alpha(t)) D_{t-1} + \alpha(t) v_{t}
$$

Notation:

- $$ D_{t} $$ is the duty cycle at time $$ t $$.
- $$ \alpha $$, "alpha", is a multiplicative factor. It's implemented as a
  lookup table for constants, based on the current timestep (scroll down).
- $$ v_{t} $$ is whether the segment is active at time $$ t $$. It's 0 if no, 1
  if yes.

Let's make sense of the formula.

One way to calculate duty cycle would be:

$$
D_{t} = \frac{\text{number of activations}}{t}
$$

All timesteps would be created equal. All segment activations have equal impact
on the duty cycle.

You're still reading, so you're smart enough to know that the incremental
version of this formula is:

$$
\begin{align}
D_{t} & = \frac{(t-1) D_{t-1} + v_t}{t} \\
\\
    & = \frac{t-1}{t} D_{t-1} + \frac{1}{t} v_t \\
\\
    & = (1 - \frac{1}{t}) D_{t-1} + \frac{1}{t} v_t
\end{align}
$$

This formula makes intuitive sense. For example, if $$ t $$ is 1000, this rule
says:

> "The duty cycle is $$ \frac{999}{1000} $$ of the previous duty cycle, plus $$
> \frac{1}{1000} $$ of the segment's current state."

So $$ \alpha $$ corresponds to $$ \frac{1}{t} $$. It's interesting to see how
they differ.

Here are the $$ \alpha $$ values that TP.py uses, compared to $$ \frac{1}{t} $$.

<style>
td {
  padding: 20px 30px;
  border: 1px solid black;
}
</style>


| $$ t $$ range  | $$ \frac{1}{t} $$ range  | $$ \alpha $$ |
|:---------------:|:---------------:|:-------------:|
| $$ 0 : 100 $$      | $$ \infty : 1\% $$ | n/a
| $$ 100 : 320 $$    | $$ 1\% : 0.3125\% $$  | $$ 0.32\% $$
| $$ 320 : 1000 $$  | $$ 0.3125\% : 0.1\% $$ | $$ 0.1\% $$
| $$ 1000 : 3200 $$  | $$ 0.1\% : 0.03125\% $$ | $$ 0.032\% $$
| $$ 3200 : 10000 $$  | $$ 0.03125\% : 0.01\% $$ | $$ 0.01\% $$
| $$ 10000 : 32000 $$  | $$ 0.01\% : 0.003125\% $$ | $$ 0.0032\% $$

<br />

and so on.

This table shows that most of the time $$ \alpha > \frac{1}{t} $$. For example,
at timestep 1000, $$ \frac{1}{\alpha} \approx 3200 $$, so this rule basically says:

> "The duty cycle is $$ \sim \frac{3199}{3200} $$ of the previous duty cycle,
> plus $$ \sim \frac{1}{3200} $$ of the segment's current state."

In other words, it starts with a duty cycle that was established in 1000
timesteps, but it gives this previous value the weight of 3200 timesteps. So
this segment's duty cycle will take a disproportionately long time to
change. Hopefully this Temporal Memory had a happy childhood.


## How I got this data

Here's my Jupyter notebook:

- [See it as HTML](/stuff/2016.05.02.old-tp-segment-lifetimes.html)
- [Download the notebook](/stuff/2016.05.02.old-tp-segment-lifetimes.ipynb)

I used a
[private throwaway version of TP.py](https://github.com/mrcslws/nupic/blob/blogpost.2016.05.05/src/nupic/research/TP.py)
that added segment event hooks.

Oh, in case you don't know:

- `TP.py` / `TP10X2.py` are the production implementations of Temporal
  Memory. They combine Temporal Memory with some nonbiological features. TP10X2
  is mostly written in C++ and produces identical results to TP.py.
- `TemporalMemory.py` / `TemporalMemory.cpp` are recent pure implementations of
  Temporal Memory.

Yes, this post lied a little bit about the $$ \alpha $$ value at step 1000. This
one made a better story.

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
