---
layout: post
title: "The column SDR that wasn't random enough"
date:   2016-03-31 12:00:00
---

<script src="//d3js.org/d3.v3.min.js" charset="utf-8"></script>

<script>
function drawActiveColumns(columns, node) {
    var scale = 0.6,
        g = d3.select(node)
            .append('svg')
            .attr('width', 1024*scale + 2)
            .attr('height', 14 + 2)
            .style('margin-bottom', '10px')
            .append('g')
            .attr('transform', 'translate(1,1)');

    g.append('rect')
        .attr('width', 1024*scale)
        .attr('height', 14)
        .attr('fill', 'none')
        .attr('stroke', 'gray');

    g.selectAll('.column')
        .data(columns)
        .call(function(column) {
            column.enter()
                .append('rect')
                .attr('class', 'column')
                .attr('width', 2)
                .attr('height', 10)
                .attr('fill', 'crimson')
                .attr('stroke', 'none')
                .attr('y', 2);

            column.exit()
                .remove();
        })
        .attr('x', function(d, i) { return d*scale; });
}
</script>

Today an untrained Spatial Pooler gave me this:

<div id="theColumns"></div>

<script>

drawActiveColumns([381, 545, 559, 569, 579, 596, 599, 612, 615, 618, 623, 624, 634,
      758, 761, 787, 788, 799, 804, 807, 808, 811, 816, 820, 825, 827,
      830, 834, 838, 843, 844, 847, 848, 854, 858, 859, 861, 878, 883, 927],
     document.getElementById('theColumns'));
</script>

And I was like, "That shouldn't happen."

<div style="height:10px;"></div>

_______

<br/>Quick explanation:

- Each red line is an active column.
- The white space contains 1024 columns.
- The exact column numbers are <span style='font-size:10px;'>[381, 545, 559, 569, 579, 596, 599, 612, 615, 618, 623, 624, 634, 758, 761, 787, 788, 799, 804, 807, 808, 811, 816, 820, 825, 827,
 830, 834, 838, 843, 844, 847, 848, 854, 858, 859, 861, 878, 883, 927]</span>

_______

<div style="height:16px;"></div>

An untrained spatial pooler should create randomly distributed SDRs. This is way
too structured.

<div style="height:10px;"></div>

## Inspect the synapses

When I saw the synapses, I was enlightened.

<div style="height:10px;"></div>

<img src='/stuff/2016-03-31-synapses.png' />

<div style="height:10px;"></div>
_______

<br/>Quick explanation:

- Every column represents an HTM column.
- Every row represents an input bit.
- A black dot indicates a synapse.

_______

<div style="height:16px;"></div>

The connections aren't random -- they are bound by topology. A column can only
connect to "nearby" input bits.

Here's the 256-bit input that I gave the spatial pooler.

<div id="theEncoding"></div>

<script>
var onbits = [ 10,  16,  50,  57,  93, 103, 105, 106, 137, 144, 145, 147, 153,
                170, 185, 199, 203, 204, 216, 235, 243];

var g = d3.select(document.getElementById('theEncoding'))
        .append('svg')
        .attr('width', 2*256 + 2)
        .attr('height', 4 + 2)
        .style('margin-bottom', '10px')
        .append('g')
        .attr('transform', 'translate(1,1)');

g.append('rect')
    .attr('width', 2*256)
    .attr('height', 4)
    .attr('fill', 'none')
    .attr('stroke', 'gray');

g.selectAll('.onbit')
    .data(onbits)
    .call(function(onbit) {
        onbit.enter()
            .append('rect')
            .attr('class', 'onbit')
            .attr('width', 2)
            .attr('height', 4)
            .attr('fill', 'blue')
            .attr('stroke', 'none');

        onbit.exit()
            .remove();
    })
    .attr('x', function(d, i) { return d*2; });
</script>

Let's overlay the inputs and active columns onto the synapse graphic.

<div style="height:10px;"></div>

<img src='/stuff/2016-03-31-synapses-overlay.png' />

Focus on the black diagonal. Wherever there are lots of blue lines crossing the
black, there are lots of red lines. Everywhere else, nothing.

Because connections are topological, and because inhibition is global, this
spatial pooler tends to only detect patterns in a subset of the input.

## The spatial pooler: Topological by default

This was news to me!

You can get rid of topology by changing `potentialRadius` from its default
(16). Here I've changed it to 256.

<div id="theColumnsFixed"></div>

<script>
drawActiveColumns([ 16,  70,  98, 155, 210, 211, 220, 225, 256, 268, 357, 378, 385,
        434, 435, 444, 446, 455, 480, 503, 517, 525, 551, 603, 633, 650,
        655, 670, 727, 740, 795, 808, 815, 862, 880, 884, 929, 944, 965, 985],
     document.getElementById('theColumnsFixed'));
</script>

<div style="height:10px;"></div>

<img src='/stuff/2016-03-31-synapses-fixed.png' />

## Should I freak out?

Maybe, but only if you use the raw `SpatialPooler`.

When you use the Spatial Pooler via the network API, it quietly
[disables topology](https://github.com/numenta/nupic/blob/3d71ef7bc9e660d398040e42d0dcb6802a9bd066/src/nupic/regions/SPRegion.py#L467). So
the CLAModel is fine. Hotgym is fine.
