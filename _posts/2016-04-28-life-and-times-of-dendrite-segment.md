---
layout: post
title: "The Life and Times of a Dendrite Segment"
date:   2016-04-28 12:00:00
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


I like seeing [HTM](http://numenta.org/) run from multiple angles.

Here's a new angle: **segment lifetimes**.

<div style="position:absolute; left:0; right:0;">
<div id="experiment3" style="width:950px;margin:auto;position:relative;"></div>
</div>
<div style="height:725px;"></div>
<script>
insertCharts3(document.getElementById('experiment3'),
      "/stuff/2016.04.28.experiment3_column_states.csv",
      "/stuff/2016.04.28.experiment3_segment_lifetimes.json"
      );
</script>

_______

<br/>Quick explanation:

- The gray rectangles show dendrite segment addition and removal.
- The colors show segment activity.
- It's interactive, and there's a learning curve.
  - Use your mouse wheel on the top chart to zoom in on a timestep.
  - Use your mouse wheel on the bottom chart to zoom in on a segment.
  - Do both! Try zooming in on different colorful areas.

Other details:

- This is a random sample of a large number of segments. The sorted list of
  segments is split into a small number of equal-sized buckets, and one segment
  is chosen randomly from each. So it's not totally random.
- If you zoom way in and then out, the random sample is different every time.

_______

<div style="height:16px;"></div>



In
[lots of time series visualizations](http://mrcslws.com/blocks/2016/02/15/htm-stacks-of-time-series.html),
the y dimension shows counts. This is a useful aggregate, but it obscures some
of what's going on. When you flatten a description of an HTM into counts, you
can't track an individual column or cell or segment across time.

Another approach is to use the y dimension to show
identity. [Sanity](https://github.com/nupic-community/sanity)'s HTM region
inspector does this with HTM columns, so you can follow a single column by
moving left and right. It tells stories about individual HTM columns.

This visualization tells stories about individual HTM segments.

<div style="margin-top:20px"></div>

## An example: Is white noise a neuralyzer?

Let's run an experiment:

1. Feed an HTM some data with sequencial patterns ([hotgym](https://github.com/numenta/nupic/blob/master/src/nupic/datafiles/extra/hotgym/rec-center-hourly.csv)).
2. Feed it random garbage.
3. Feed it the first data again.

What happens?

We should run this experiment twice. The
[Python temporal memory](https://github.com/numenta/nupic/blob/master/src/nupic/research/temporal_memory.py)
doesn't currently implement segment cleanup -- it will add dendrite segments
forever, never clearing out old ones. The
[C++ temporal memory](https://github.com/mrcslws/nupic.core/blob/columnSegmentWalk/src/nupic/algorithms/TemporalMemory.cpp)
implements segment cleanup, so it's worth testing how this impacts results.

<div style="margin-top:20px"></div>

### The base algorithm: temporal_memory.py

<div style="margin-top:20px"></div>

<div style="position:absolute; left:0; right:0;">
<div id="experiment1" style="width:950px;margin:auto;position:relative;"></div>
</div>
<div style="height:725px;"></div>
<script>
insertCharts3(document.getElementById('experiment1'),
      "/stuff/2016.04.28.experiment1_column_states.csv",
      "/stuff/2016.04.28.experiment1_segment_lifetimes.json"
      );
</script>

Using the top chart, you can see where each of the following happened:

- It received 1000 steps of orderly data.
- Then it received 1000 steps of totally unpredictable data.
- Finally it received 1000 steps of the same orderly data.

Using the bottom chart, you can see:

- It grew 4500 segments which tended to be useful.
- Then it grew 73000 segments which tended to never be used.
- Finally it grew 500 useful segments, but mostly relied on the original 4500.
  - (Zoom and pan to the bottom of the chart to see these 500.)

The white noise filled the temporal memory with junk, but its early knowledge
was preserved.

<div style="margin-top:20px"></div>

### Now add segment cleanup: TemporalMemory.cpp

<div style="margin-top:20px"></div>

<div style="position:absolute; left:0; right:0;">
<div id="experiment2" style="width:950px;margin:auto;position:relative;"></div>
</div>
<div style="height:725px;"></div>
<script>
insertCharts3(document.getElementById('experiment2'),
      "/stuff/2016.04.28.experiment2_column_states.csv",
      "/stuff/2016.04.28.experiment2_segment_lifetimes.json"
      );
</script>

This Temporal Memory uses the following segment cleanup rules:

- When a segment's last synapse is removed, the segment is removed.
- When a cell exceeds its maximum allowed number of segments, it removes its
  least recently active segment. The segment's birth timestep is used as its
  initial "last active" timestep.

It's good to see that this Temporal Memory cleans up its junk segments, but
there's clearly a huge flaw here. The entire set of useful segments is destroyed
to make room for new segments. On timestep 2000, it has to relearn
everything. With these cleanup rules, white noise will make an HTM forget
everything it knows.

<div style="margin-top:20px"></div>

## More opportunities

This graphic uses a fixed sort order, showing a semi-random sample of
segments. This is really useful.

I'm excited to see what happens when we add creative sorting. You might enjoy my
[sorting demo](https://youtu.be/rEQ2XVOnhDw?t=3m20s). Imagine selecting a
timestep and saying "sort by oldest segment". It wouldn't have to do a random
sample, it could just show the top ~50. You could jump around, performing
various sortings at various timesteps.

Using a random sample gives you a picture of typical tendencies within the
population. But you can't look at it and know for sure that, for example, no
segments survived the white noise. You'd have to zoom and look at every segment
to be confident. You can't really solve this problem by showing aggregates,
either. If, say, 10 segments survived, would you be able to see them in a chart
that's showing 80000 segments? This is a job for sorting.

<div style="margin-top:20px"></div>

## Also remember

HTM dendrite segments don't necessarily represent the same thing forever. If the
segments' connections tend to change dramatically, is this graphic still useful?
Is there a way to see whether / how much this is happening? It'd be fun to
explore this area.

<div style="margin-top:20px"></div>

## Appendix: How I got this data

Here's my Jupyter notebook:

- [See it as HTML](/stuff/2016.04.28.segment-lifetimes.html)
- [Download the notebook](/stuff/2016.04.28.segment-lifetimes.ipynb)

This notebook is really cool. In it, I generated these graphics inline.

I used `temporal_memory.py` from NuPIC at
[23c6686](https://github.com/numenta/nupic/tree/23c66864703eab9b72a57a2e8bcdbd52ed8abef3). For
`TemporalMemory.cpp`, I used a private version that includes a
[few](https://github.com/numenta/nupic.core/pull/917)
[changes](https://github.com/numenta/nupic.core/pull/933) which make it faster,
more visualizable, and which fix a few bugs (without changing the existing
segment cleanup algorithm).
