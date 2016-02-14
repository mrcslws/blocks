---
layout: post
title: "HTM time series: Two charts, one scale"
date:   2016-02-13 12:00:00
---

<script src="//d3js.org/d3.v3.min.js" charset="utf-8"></script>
<style type="text/css">
      text {
      font: 10px sans-serif;
      }

      .axis path {
      fill: none;
      stroke: none;
      shape-rendering: crispEdges;
      }

      .axis line {
      stroke: none;
      shape-rendering: crispEdges;
      }

      .y.axis line {
      stroke: black;
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
      }
</style>

<div style="position:absolute; left:0; right:0;">
<div id="putItHere" style="width:900px;margin:auto;position:relative;"></div>
</div>
<div style="height:400px;"></div>

This combines the two [previous](/blocks/2016/01/16/htm-column-time-series.html) [plots](/blocks/2016/01/31/htm-segment-time-series.html), giving them a shared x scale. It also redesigns the zoom/navigation experience. And it's significantly faster.

In the [first post](/blocks/2016/01/16/htm-column-time-series.html), the main perf feature is that it samples the data. In the [second post](/blocks/2016/01/31/htm-segment-time-series.html) I moved away from `<rect/>`s and toward `<path/>`s, making draws faster. In this post, I draw the path in data-space and scale/translate (a.k.a. "stretch") it into pixel-space. I use stretching for fast zooming, periodically redrawing the underlying path. "drawing" and "stretching" are two different things. Stretching is fast.

## The code

### Fetch the data

This part hasn't changed. Look at the Python sections in these posts:

- [Columns](/blocks/2016/01/16/htm-column-time-series.html)
- [Segments](/blocks/2016/01/31/htm-segment-time-series.html)

### Draw the data

Use [d3.js](http://d3js.org/) to draw the CSV data.

**CSS**

{% highlight css %}

text {
    font: 10px sans-serif;
}

.axis path {
    fill: none;
    stroke: none;
    shape-rendering: crispEdges;
}

.axis line {
    stroke: none;
    shape-rendering: crispEdges;
}

.y.axis line {
    stroke: black;
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

{% endhighlight %}

**JavaScript**

Here's a code drop.

{% highlight javascript %}

//
// SHARED STATE
//
var chartWidth = 600,
    chartLeft = 40,
    x = d3.scale.linear()
        .range([0, chartWidth]),
    onxscalechanged = [], // callbacks
    onZoomScaleExtentChanged = [], // callbacks
    timestepCount,
    zoom = d3.behavior.zoom()
        .on('zoom', function() {
            // Enforce a translateExtent
            if (x(0) > x.range()[0]) {
                zoom.translate([0, zoom.translate()[1]]);
            }
            else if (x(timestepCount) < x.range()[1]) {
                var xDomain = x.domain(),
                    domainWidth = xDomain[1] - xDomain[0],
                    leftMostInDataSpace = timestepCount - domainWidth;
                zoom.translate([-(leftMostInDataSpace * zoom.scale()),
                                zoom.translate()[1]]);
            }
            onxscalechanged.forEach(function(f) { f(true); });
        }),
    container = d3.select('#putItHere').append('div')
        .style('position', 'relative'),
    charts = container.append('div')
        .style('position', 'absolute')
        .style('left', chartLeft + 'px'),
    xSamplesDomain = [null, null],
    xSamples = [];

function onTimestepCountKnown(count) {
    if (!timestepCount || count > timestepCount) {
        timestepCount = count;
        // Set the domain to [0, count], but do this charade because the zoom's
        // scale is encapsulated and we can't change it without changing the
        // domain. A zoom scale of 1 means that the data space and pixel space
        // are equal.
        var scale = chartWidth / count;
        x.domain([0,chartWidth]);
        zoom.x(x);
        zoom.scale(scale);
        zoom.scaleExtent([chartWidth / count, Math.max(40, scale)]);
        onZoomScaleExtentChanged.forEach(function(f) { f(); });
    }
}

function xResample() {
    var extent = x.domain();
    if (xSamplesDomain[0] == extent[0] && xSamplesDomain[1] == extent[1]) {
        // No need to resample.
        return;
    }

    var xSamplesNew;
    if (extent[1] - extent[0] > chartWidth) {
        var bucketWidth = (extent[1] - extent[0]) / chartWidth,
            iPrevious = 0;
        xSamplesNew = d3.range(extent[0], extent[1], bucketWidth)
            .slice(0, chartWidth) // Floating point math can cause an extra.
            .map(function(x) {
                var data = {x0: x, x1: Math.min(x + bucketWidth, extent[1])};
                while (iPrevious < xSamples.length &&
                       xSamples[iPrevious].x < data.x0) {
                    iPrevious++;
                }

                if (iPrevious < xSamples.length &&
                    xSamples[iPrevious].x < data.x1) {
                    // When zooming / panning, the behavior is less
                    // jarring if we reuse samples rather than
                    // grabbing a new random sample.
                    data.x = xSamples[iPrevious].x;
                }
                else {
                    // Choose randomly from the interval.
                    // Otherwise with repeating patterns we'll have aliasing.
                    data.x = Math.random() * (data.x1 - data.x0) + data.x0;
                    data.x = Math.round(data.x);
                    if (data.x < data.x0) {
                        data.x++;
                    }
                    else if (data.x >= data.x1) {
                        data.x--;
                    }
                }

                return data;
            });
    }
    else {
        // No sampling needed.
        xSamplesNew = d3.range(Math.floor(extent[0]), extent[1])
            .map(function(x) { return {x0: x, x: x, x1: x + 1 };});
    }
    xSamples = xSamplesNew;
    xSamplesDomain = [extent[0], extent[1]];
}

//
// ZOOM WIDGET
//
(function() {
    function zoomTowardCenter(scale, drawImmediately) {
        var extent = x.domain(),
            center = ((extent[1] - extent[0]) / 2) + extent[0];
        zoom.scale(scale);
        var timestepsPerPixel = 1/scale,
            nTimesteps = chartWidth * timestepsPerPixel,
            newLeftmost = center - (nTimesteps/2);
        zoom.translate([-newLeftmost * scale, 0]);
        onxscalechanged.forEach(function(f) { f(false); });
    }
    var zoomer = container.append('svg')
            .attr('width', 22)
            .attr('height', 102)
            .style('position', 'absolute')
            .style('top', '20px')
            .style('left', 0)
            .append('g')
            .attr('transform', 'translate(1,1)'),
        grooveHeight = 60,
        knobWidth = 20,
        knobHeight = 4,
        groove = zoomer.append('g')
            .attr('transform', 'translate(0, 20)'),
        grooveY = d3.scale.log()
            .domain([1, 5]) // default while waiting for csv
            .range([grooveHeight - knobHeight, 0]);
    onZoomScaleExtentChanged.push(function() {
        grooveY.domain(zoom.scaleExtent());
        placeKnob();
    });
    groove.append('rect')
        .attr('x', 8)
        .attr('y', 0)
        .attr('width', 3)
        .attr('height', 60)
        .attr('stroke', 'lightgray')
        .attr('fill', 'none');
    groove.append('rect')
        .attr('class', 'clickable')
        .attr('width', 20)
        .attr('height', 60)
        .attr('stroke', 'none')
        .attr('fill', 'transparent')
        .on('click', function () {
            var y = d3.event.clientY - d3.event.target.getBoundingClientRect().top;
            zoomTowardCenter(grooveY.invert(y), true);
        });

    [{text: '+',
      translateY: 0,
      onclick: function() {
          var y = Math.max(0, grooveY(zoom.scale()) - 5);
          zoomTowardCenter(grooveY.invert(y), true);
      }},
     {text: '-',
      translateY: grooveHeight + 20,
      onclick: function() {
          var y = Math.min(grooveHeight - knobHeight, grooveY(zoom.scale()) + 5);
          zoomTowardCenter(grooveY.invert(y), true);
      }}]
        .forEach(function(spec) {
            var button = zoomer.append('g')
                    .attr('transform', 'translate(0,' +
                          spec.translateY + ')');
            button.append('text')
                .attr('class', 'noselect')
                .attr('x', 10)
                .attr('y', 10)
                .attr('dy', '.26em')
                .attr('text-anchor', 'middle')
                .style('font', '15px sans-serif')
                .style('font-weight', 'bold')
                .style('fill', 'gray')
                .text(spec.text);
            button.append('rect')
                .attr('width', 20)
                .attr('height', 20)
                .attr('stroke-width', 1)
                .attr('stroke', 'gray')
                .attr('fill', 'transparent')
                .attr('class', 'clickable')
                .on('click', spec.onclick);
        });

    var knob = groove.append('g')
            .attr('class', 'draggable')
            .attr('transform', function(d) {
                return 'translate(0,' + grooveY(d) + ')';
            }),
        knobProgress = knob.append('rect')
            .attr('height', knobHeight)
            .attr('fill', 'black')
            .attr('stroke', 'none'),
        knobTitle = knob.append('title');
    knob.append('rect')
        .attr('width', knobWidth)
        .attr('height', knobHeight)
        .attr('fill', 'transparent')
        .attr('stroke', 'gray')
        .call(d3.behavior.drag()
              .on('dragstart', function() {
                  zoomer.classed('dragging', true);
              })
              .on('drag', function() {
                  var y = d3.event.sourceEvent.clientY -
                          groove.node().getBoundingClientRect().top;
                  y = Math.max(0, y);
                  y = Math.min(grooveHeight - knobHeight, y);
                  zoomTowardCenter(grooveY.invert(y));
              })
              .on('dragend', function() {
                  zoomer.classed('dragging', false);
              }));

    function placeKnob() {
        var scale = zoom.scale(),
            sampleRate = Math.min(scale, 1);
        knob.attr('transform', 'translate(0,' + grooveY(scale) + ')');
        knobProgress.attr('width', knobWidth * sampleRate);
        knobTitle.text(sampleRate == 1 ?
                       "Displaying every timestep in this interval." :
                       ("Due to limited pixels, only " +
                        Math.round(sampleRate*100) +
                        "% of timesteps in this interval are shown."));
    }

    onxscalechanged.push(placeKnob);
})();

//
// SHARED CHART CODE
//
function stackedTimeSeries() {
    var colorScale,
        x,
        y,
        xExtent;

    function stretch(selection) {
        selection.each(function(data) {
            var pixelsPerTimestep =
                    (x.range()[1] - x.range()[0]) /
                    (x.domain()[1] - x.domain()[0]);
            d3.select(this).selectAll('.stretchMe')
                .attr('transform', 'scale(' + pixelsPerTimestep +
                      ',1)translate(' + (-x.domain()[0]) + ',0)');
        });
    }

    var chart = function (selection) {
        selection.each(function(data) {
            var stretchMe = d3.select(this).selectAll('.stretchMe')
                    .data([data]);
            stretchMe.enter()
                .append('g')
                .attr('class', 'stretchMe');

            var layer = stretchMe.selectAll('.layer')
                    .data(function (d) {return d;});
            layer.enter()
                .append('path')
                .attr('class', 'layer')
                .attr('shape-rendering', 'crispEdges')
                .attr('fill', function(d, i) { return colorScale(i); })
                .attr('stroke', 'none');

            layer.attr('d', function (ds) {
                return ds.map(function(d) {
                    var x0 = d.x0,
                        x1 = d.x1,
                        y0 = y(d.y0 + d.y),
                        y1 = y(d.y0);
                    return ['M', x0, y0,
                            'L', x1, y0,
                            'L', x1, y1,
                            'L', x0, y1,
                            'Z'].join(' ');
                }).join(' ');
            });

            layer.exit().remove();
        }).call(stretch);
    };

    chart.stretch = stretch;

    chart.colorScale = function(_) {
        if (!arguments.length) return colorScale;
        colorScale = _;
        return chart;
    };

    chart.x = function(_) {
        if (!arguments.length) return x;
        x = _;
        return chart;
    };

    chart.y = function(_) {
        if (!arguments.length) return y;
        y = _;
        return chart;
    };

    return chart;
}

//
// COLUMN STATES PLOT
//
(function() {
    var margin = {top: 0, right: 300, bottom: 4, left: 0},
        height = 150 - margin.top - margin.bottom;

    var svg = charts.append('svg')
            .attr('width', chartWidth + margin.left + margin.right)
            .attr('height', height + margin.top + margin.bottom);

    var defs = svg.append('defs');
    defs.append('clipPath')
        .attr('id', 'clip1')
        .append('rect')
        .attr('width', chartWidth)
        .attr('height', height);

    d3.csv('/stuff/hotgym.column_states.2016-01-16.1505.csv')
        .row(function(d) {
            var intKeys = ['n-unpredicted-active-columns',
                           'n-predicted-active-columns',
                           'n-predicted-inactive-columns'];
            intKeys.forEach(function(k) {
                d[k] = parseInt(d[k]);
            });
            return d;
        })
        .get(function(error, timesteps) {
            onTimestepCountKnown(timesteps.length);
            var stackOrder = [{key: 'n-unpredicted-active-columns',
                               color: 'hsl(0,100%,50%)',
                               activeText: 'active',
                               predictedText: 'not predicted'},
                              {key: 'n-predicted-active-columns',
                               color: 'hsl(270,100%,40%)',
                               activeText: 'active',
                               predictedText: 'predicted'},
                              {key: 'n-predicted-inactive-columns',
                               color: 'hsla(210,100%,50%,0.5)',
                               activeText: 'not active',
                               predictedText: 'predicted'}],
                yStackMax = d3.max(timesteps, function(d) {
                    return stackOrder.reduce(function(sum, o) {
                        return sum + d[o.key];
                    }, 0);
                }),
                chartNode = svg.append('g')
                    .attr('class', 'chart')
                    .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')'),
                stackColorScale = d3.scale.ordinal()
                    .domain(stackOrder.map(function(d) { return d.key; }))
                    .range(stackOrder.map(function(d) { return d.color; })),
                help = chartNode.append('g')
                    .attr('transform', 'translate(' + (chartWidth+50) + ',40)');

            // Title and legend
            help.append('text')
                .attr('class', 'noselect')
                .attr('text-anchor', 'right')
                .attr('x', 5)
                .attr('y', 0)
                .text('active and predicted columns, stacked by:');

            var legend = help.append('g')
                    .attr('class', 'noselect')
                    .attr('transform', 'translate(0, 20)'),
                unitWidth = 80,
                color = legend.selectAll('g')
                    .data(stackOrder);
            stackOrder.forEach(function(o, i) {
                var g = legend.append('g')
                        .attr('transform', 'translate(' + i * unitWidth + ',0)');
                g.append('rect')
                    .attr('width', unitWidth)
                    .attr('height', 4)
                    .attr('fill', o.color);
                g.append('text')
                    .attr('x', unitWidth/2)
                    .attr('dy', '-0.24em')
                    .attr('text-anchor', 'middle')
                    .text(o.activeText);
                g.append('text')
                    .attr('x', unitWidth/2)
                    .attr('y', 14)
                    .attr('text-anchor', 'middle')
                    .text(o.predictedText);
            });

            // Chart
            var y = d3.scale.linear()
                    .domain([0, yStackMax])
                    .range([height, 0]),
                drawTimeout = null,
                chart = stackedTimeSeries()
                    .colorScale(stackColorScale)
                    .x(x)
                    .y(y),
                chartNodeInner = chartNode.append('g')
                    .style('clip-path', 'url(#clip1)');

            chartNode.append('rect')
                .attr('fill-opacity', 0)
                .attr('x', 0)
                .attr('y', 0)
                .attr('width', chartWidth)
                .attr('height', height)
                .call(zoom);

            function draw () {
                if (drawTimeout) {
                    clearTimeout(drawTimeout);
                    drawTimeout = null;
                }

                xResample();

                var layers = [];
                stackOrder.forEach(function(o) {
                    var layer = xSamples.map(function(data) {
                        var y = timesteps[data.x][o.key] || 0;
                        return {x: data.x, x0: data.x0, x1: data.x1, y: y};
                    });
                    layers.push(layer);
                });
                d3.layout.stack()(layers); // inserts y0 values
                chartNodeInner.datum(layers)
                    .call(chart);
            }

            onxscalechanged.push(function(maybeCoalesce) {
                chartNodeInner.call(chart.stretch);
                if (!drawTimeout) {
                    drawTimeout = setTimeout(draw, maybeCoalesce ? 250 : 1000/60);
                }
            });

            // y axis
            chartNode.append('g')
                .attr('transform', 'translate(' + chartWidth + ', 0)')
                .attr('class', 'y axis')
                .call(d3.svg.axis()
                      .scale(y)
                      .ticks(4)
                      .tickPadding(2)
                      .tickSize(4)
                      .outerTickSize(0)
                      .orient('right'));

            // Initial change
            draw();
            onxscalechanged.forEach(function (f) { f(); });
        });
})();

//
// SEGMENTS PLOT
//
(function() {
    var margin = {top: 20, right: 300, bottom: 20, left: 0},
        height = 200 - margin.top - margin.bottom;

    var svg = charts.append('svg')
            .attr('width', chartWidth + margin.left + margin.right)
            .attr('height', height + margin.top + margin.bottom);

    var defs = svg.append('defs');
    defs.append('clipPath')
        .attr('id', 'clip2')
        .append('rect')
        .attr('width', chartWidth)
        .attr('height', height);

    d3.text('/stuff/hotgym.segments.2016-01-31.1400.csv', 'text/csv', function(error, contents) {
        var rows = d3.csv.parseRows(contents),
            htmData = rows[0],
            timesteps = rows.splice(1),
            activationThreshold = parseInt(htmData[0]),
            maxTotalSegments = 0,
            maxConnectedSynapsesInSegment = 0;

        onTimestepCountKnown(timesteps.length);

        timesteps.forEach(function(row) {
            var totalSegments = 0;
            for (var i = 0; i < row.length; i++) {
                row[i] = parseInt(row[i]);
                totalSegments += row[i];
                if (i > maxConnectedSynapsesInSegment && row[i] > 0) {
                    maxConnectedSynapsesInSegment = i;
                }
            }
            if (totalSegments > maxTotalSegments) {
                maxTotalSegments = totalSegments;
            }
        });

        var chartNode = svg.append('g')
                .attr('class', 'chart')
                .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')'),
            stackOrder = d3.range(maxConnectedSynapsesInSegment + 1).reverse(),
            colorExtent = ['whitesmoke', 'black'],
            connectedSynapsesColorScale = d3.scale.linear()
                .domain([0, maxConnectedSynapsesInSegment])
                .range(colorExtent),
            stackColorScale = d3.scale.linear()
                .domain([maxConnectedSynapsesInSegment, 0])
                .range(colorExtent),
            help = chartNode.append('g')
                .attr('transform', 'translate(' + (chartWidth+50) + ',40)');

        // Title and legend
        help.append('text')
            .attr('class', 'noselect')
            .attr('text-anchor', 'right')
            .attr('x', 2)
            .attr('y', 0)
            .text('distal segments, stacked by:');

        var legend = help.append('g')
                .attr('class', 'noselect')
                .attr('transform', 'translate(0, 20)'),
            domainWidth = maxConnectedSynapsesInSegment + 1,
            legendWidth = 200,
            unitWidth = legendWidth / domainWidth,
            rect = legend.selectAll('rect')
                .data(d3.range(maxConnectedSynapsesInSegment + 1));
        rect.enter()
            .append('rect')
            .attr('x', function(d, i) { return i * unitWidth; })
            .attr('width', unitWidth)
            .attr('height', 4);
        rect.attr('fill', function(d, i) { return connectedSynapsesColorScale(d); });
        legend.append('text')
            .attr('x', legendWidth / 2)
            .attr('y', -4)
            .attr('text-anchor', 'middle')
            .text('number of connected synapses on segment');
        legend.append('text')
            .attr('y', 14)
            .attr('text-anchor', 'middle')
            .text(0);
        legend.append('text')
            .attr('x', legendWidth)
            .attr('text-anchor', 'middle')
            .attr('y', 14)
            .text(maxConnectedSynapsesInSegment);

        // Chart
        var y = d3.scale.linear()
                .domain([0, maxTotalSegments])
                .range([height, 0]),
            drawTimeout = null,
            chart = stackedTimeSeries()
                .colorScale(stackColorScale)
                .x(x)
                .y(y),
            chartNodeInner = chartNode.append('g')
                .style('clip-path', 'url(#clip2)');

        chartNode.append('rect')
            .attr('fill-opacity', 0)
            .attr('x', 0)
            .attr('y', 0)
            .attr('width', chartWidth)
            .attr('height', height)
            .call(zoom);

        function draw () {
            if (drawTimeout) {
                clearTimeout(drawTimeout);
                drawTimeout = null;
            }

            xResample();

            var layers = [];
            stackOrder.forEach(function(key) {
                var layer = xSamples.map(function(data) {
                    var y = timesteps[data.x][key] || 0;
                    return {x: data.x, x0: data.x0, x1: data.x1, y: y};
                });
                layers.push(layer);
            });
            d3.layout.stack()(layers); // inserts y0 values
            chartNodeInner.datum(layers)
                .call(chart);
        }

        onxscalechanged.push(function(maybeCoalesce) {
            chartNodeInner.call(chart.stretch);
            if (!drawTimeout) {
                drawTimeout = setTimeout(draw, maybeCoalesce ? 250 : 1000/60);
            }
        });

        // Axes
        var yLabels = chartNode.append('g')
                .attr('transform', 'translate(' + (chartWidth + 2) + ', 0)');

        onxscalechanged.push(function () {
            var finalRow = timesteps[Math.floor(x.domain()[1] - 1)],
                spikable = finalRow.slice(activationThreshold)
                    .reduce(function(sum, v) { return sum + v; }, 0),
                total = finalRow.reduce(function(sum, v) { return sum + v; }, 0);
            var groups = yLabels.selectAll('g')
                    .data([[spikable, 'spikable', '0.34em'],
                           [total, 'total', '0.34em']]);
            var enteringGroups = groups.enter()
                    .append('g');
            enteringGroups.append('line')
                .attr('x1', 0)
                .attr('x2', 4)
                .attr('y1', 0)
                .attr('y2', 0)
                .attr('stroke', 'black')
                .attr('stroke-width', 1);
            enteringGroups.append('text')
                .attr('class', 'noselect number')
                .attr('x', 6)
                .attr('dy', function(d, i) { return d[2]; });
            enteringGroups.append('text')
                .attr('class', 'noselect')
                .attr('x', 6)
                .attr('y', 15)
                .text(function(d) { return d[1]; });
            groups.attr('transform', function(d, i) {
                return 'translate(0,' + y(d[0]) + ')';
            })
                .select('.number')
                .text(function(d, i) { return d[0]; });
        });

        // Initial change
        draw();
        onxscalechanged.forEach(function (f) { f(); });
    });
})();

//
// SHARED X AXIS
//
(function() {
    var xAxis = d3.svg.axis()
            .scale(x)
            .tickPadding(3)
            .tickSize(0)
            .outerTickSize(0)
            .orient('bottom')
            .tickFormat(d3.format('d')),
        marginLeft = 3,
        svg = charts.append('svg')
            .attr('width', chartWidth + 10)
            .attr('height', 30)
            .style('position', 'relative')
            .style('left', -marginLeft + 'px')
            .style('top', '-20px'),
        xAxisNode = svg.append('g')
            .attr('class', 'x axis noselect')
            .attr('transform', 'translate(' + marginLeft + ',0)')
            .append('g');
    onxscalechanged.push(function () {
        var extent = x.domain(),
            domainWidth = extent[1] - extent[0],
            pixelsPerTimestep = chartWidth / domainWidth,
            tickShift = pixelsPerTimestep / 2;
        xAxis.ticks(Math.min(domainWidth, 15));
        xAxisNode.call(xAxis);
        xAxisNode.attr('transform', 'translate(' + tickShift + ',' + '0)');
    });

    svg.append('text')
        .attr('class', 'x label noselect')
        .attr('x', marginLeft)
        .attr('y', 25)
        .text('timestep');
})();

{% endhighlight %}

<script>
//
// SHARED STATE
//
var chartWidth = 600,
    chartLeft = 40,
    x = d3.scale.linear()
        .range([0, chartWidth]),
    onxscalechanged = [], // callbacks
    onZoomScaleExtentChanged = [], // callbacks
    timestepCount,
    zoom = d3.behavior.zoom()
        .on('zoom', function() {
            // Enforce a translateExtent
            if (x(0) > x.range()[0]) {
                zoom.translate([0, zoom.translate()[1]]);
            }
            else if (x(timestepCount) < x.range()[1]) {
                var xDomain = x.domain(),
                    domainWidth = xDomain[1] - xDomain[0],
                    leftMostInDataSpace = timestepCount - domainWidth;
                zoom.translate([-(leftMostInDataSpace * zoom.scale()),
                                zoom.translate()[1]]);
            }
            onxscalechanged.forEach(function(f) { f(true); });
        }),
    container = d3.select('#putItHere').append('div')
        .style('position', 'relative'),
    charts = container.append('div')
        .style('position', 'absolute')
        .style('left', chartLeft + 'px'),
    xSamplesDomain = [null, null],
    xSamples = [];

function onTimestepCountKnown(count) {
    if (!timestepCount || count > timestepCount) {
        timestepCount = count;
        // Set the domain to [0, count], but do this charade because the zoom's
        // scale is encapsulated and we can't change it without changing the
        // domain. A zoom scale of 1 means that the data space and pixel space
        // are equal.
        var scale = chartWidth / count;
        x.domain([0,chartWidth]);
        zoom.x(x);
        zoom.scale(scale);
        zoom.scaleExtent([chartWidth / count, Math.max(40, scale)]);
        onZoomScaleExtentChanged.forEach(function(f) { f(); });
    }
}

function xResample() {
    var extent = x.domain();
    if (xSamplesDomain[0] == extent[0] && xSamplesDomain[1] == extent[1]) {
        // No need to resample.
        return;
    }

    var xSamplesNew;
    if (extent[1] - extent[0] > chartWidth) {
        var bucketWidth = (extent[1] - extent[0]) / chartWidth,
            iPrevious = 0;
        xSamplesNew = d3.range(extent[0], extent[1], bucketWidth)
            .slice(0, chartWidth) // Floating point math can cause an extra.
            .map(function(x) {
                var data = {x0: x, x1: Math.min(x + bucketWidth, extent[1])};
                while (iPrevious < xSamples.length &&
                       xSamples[iPrevious].x < data.x0) {
                    iPrevious++;
                }

                if (iPrevious < xSamples.length &&
                    xSamples[iPrevious].x < data.x1) {
                    // When zooming / panning, the behavior is less
                    // jarring if we reuse samples rather than
                    // grabbing a new random sample.
                    data.x = xSamples[iPrevious].x;
                }
                else {
                    // Choose randomly from the interval.
                    // Otherwise with repeating patterns we'll have aliasing.
                    data.x = Math.random() * (data.x1 - data.x0) + data.x0;
                    data.x = Math.round(data.x);
                    if (data.x < data.x0) {
                        data.x++;
                    }
                    else if (data.x >= data.x1) {
                        data.x--;
                    }
                }

                return data;
            });
    }
    else {
        // No sampling needed.
        xSamplesNew = d3.range(Math.floor(extent[0]), extent[1])
            .map(function(x) { return {x0: x, x: x, x1: x + 1 };});
    }
    xSamples = xSamplesNew;
    xSamplesDomain = [extent[0], extent[1]];
}

//
// ZOOM WIDGET
//
(function() {
    function zoomTowardCenter(scale, drawImmediately) {
        var extent = x.domain(),
            center = ((extent[1] - extent[0]) / 2) + extent[0];
        zoom.scale(scale);
        var timestepsPerPixel = 1/scale,
            nTimesteps = chartWidth * timestepsPerPixel,
            newLeftmost = center - (nTimesteps/2);
        zoom.translate([-newLeftmost * scale, 0]);
        onxscalechanged.forEach(function(f) { f(false); });
    }
    var zoomer = container.append('svg')
            .attr('width', 22)
            .attr('height', 102)
            .style('position', 'absolute')
            .style('top', '20px')
            .style('left', 0)
            .append('g')
            .attr('transform', 'translate(1,1)'),
        grooveHeight = 60,
        knobWidth = 20,
        knobHeight = 4,
        groove = zoomer.append('g')
            .attr('transform', 'translate(0, 20)'),
        grooveY = d3.scale.log()
            .domain([1, 5]) // default while waiting for csv
            .range([grooveHeight - knobHeight, 0]);
    onZoomScaleExtentChanged.push(function() {
        grooveY.domain(zoom.scaleExtent());
        placeKnob();
    });
    groove.append('rect')
        .attr('x', 8)
        .attr('y', 0)
        .attr('width', 3)
        .attr('height', 60)
        .attr('stroke', 'lightgray')
        .attr('fill', 'none');
    groove.append('rect')
        .attr('class', 'clickable')
        .attr('width', 20)
        .attr('height', 60)
        .attr('stroke', 'none')
        .attr('fill', 'transparent')
        .on('click', function () {
            var y = d3.event.clientY - d3.event.target.getBoundingClientRect().top;
            zoomTowardCenter(grooveY.invert(y), true);
        });

    [{text: '+',
      translateY: 0,
      onclick: function() {
          var y = Math.max(0, grooveY(zoom.scale()) - 5);
          zoomTowardCenter(grooveY.invert(y), true);
      }},
     {text: '-',
      translateY: grooveHeight + 20,
      onclick: function() {
          var y = Math.min(grooveHeight - knobHeight, grooveY(zoom.scale()) + 5);
          zoomTowardCenter(grooveY.invert(y), true);
      }}]
        .forEach(function(spec) {
            var button = zoomer.append('g')
                    .attr('transform', 'translate(0,' +
                          spec.translateY + ')');
            button.append('text')
                .attr('class', 'noselect')
                .attr('x', 10)
                .attr('y', 10)
                .attr('dy', '.26em')
                .attr('text-anchor', 'middle')
                .style('font', '15px sans-serif')
                .style('font-weight', 'bold')
                .style('fill', 'gray')
                .text(spec.text);
            button.append('rect')
                .attr('width', 20)
                .attr('height', 20)
                .attr('stroke-width', 1)
                .attr('stroke', 'gray')
                .attr('fill', 'transparent')
                .attr('class', 'clickable')
                .on('click', spec.onclick);
        });

    var knob = groove.append('g')
            .attr('class', 'draggable')
            .attr('transform', function(d) {
                return 'translate(0,' + grooveY(d) + ')';
            }),
        knobProgress = knob.append('rect')
            .attr('height', knobHeight)
            .attr('fill', 'black')
            .attr('stroke', 'none'),
        knobTitle = knob.append('title');
    knob.append('rect')
        .attr('width', knobWidth)
        .attr('height', knobHeight)
        .attr('fill', 'transparent')
        .attr('stroke', 'gray')
        .call(d3.behavior.drag()
              .on('dragstart', function() {
                  zoomer.classed('dragging', true);
              })
              .on('drag', function() {
                  var y = d3.event.sourceEvent.clientY -
                          groove.node().getBoundingClientRect().top;
                  y = Math.max(0, y);
                  y = Math.min(grooveHeight - knobHeight, y);
                  zoomTowardCenter(grooveY.invert(y));
              })
              .on('dragend', function() {
                  zoomer.classed('dragging', false);
              }));

    function placeKnob() {
        var scale = zoom.scale(),
            sampleRate = Math.min(scale, 1);
        knob.attr('transform', 'translate(0,' + grooveY(scale) + ')');
        knobProgress.attr('width', knobWidth * sampleRate);
        knobTitle.text(sampleRate == 1 ?
                       "Displaying every timestep in this interval." :
                       ("Due to limited pixels, only " +
                        Math.round(sampleRate*100) +
                        "% of timesteps in this interval are shown."));
    }

    onxscalechanged.push(placeKnob);
})();

//
// SHARED CHART CODE
//
function stackedTimeSeries() {
    var colorScale,
        x,
        y,
        xExtent;

    function stretch(selection) {
        selection.each(function(data) {
            var pixelsPerTimestep =
                    (x.range()[1] - x.range()[0]) /
                    (x.domain()[1] - x.domain()[0]);
            d3.select(this).selectAll('.stretchMe')
                .attr('transform', 'scale(' + pixelsPerTimestep +
                      ',1)translate(' + (-x.domain()[0]) + ',0)');
        });
    }

    var chart = function (selection) {
        selection.each(function(data) {
            var stretchMe = d3.select(this).selectAll('.stretchMe')
                    .data([data]);
            stretchMe.enter()
                .append('g')
                .attr('class', 'stretchMe');

            var layer = stretchMe.selectAll('.layer')
                    .data(function (d) {return d;});
            layer.enter()
                .append('path')
                .attr('class', 'layer')
                .attr('shape-rendering', 'crispEdges')
                .attr('fill', function(d, i) { return colorScale(i); })
                .attr('stroke', 'none');

            layer.attr('d', function (ds) {
                return ds.map(function(d) {
                    var x0 = d.x0,
                        x1 = d.x1,
                        y0 = y(d.y0 + d.y),
                        y1 = y(d.y0);
                    return ['M', x0, y0,
                            'L', x1, y0,
                            'L', x1, y1,
                            'L', x0, y1,
                            'Z'].join(' ');
                }).join(' ');
            });

            layer.exit().remove();
        }).call(stretch);
    };

    chart.stretch = stretch;

    chart.colorScale = function(_) {
        if (!arguments.length) return colorScale;
        colorScale = _;
        return chart;
    };

    chart.x = function(_) {
        if (!arguments.length) return x;
        x = _;
        return chart;
    };

    chart.y = function(_) {
        if (!arguments.length) return y;
        y = _;
        return chart;
    };

    return chart;
}

//
// COLUMN STATES PLOT
//
(function() {
    var margin = {top: 0, right: 300, bottom: 4, left: 0},
        height = 150 - margin.top - margin.bottom;

    var svg = charts.append('svg')
            .attr('width', chartWidth + margin.left + margin.right)
            .attr('height', height + margin.top + margin.bottom);

    var defs = svg.append('defs');
    defs.append('clipPath')
        .attr('id', 'clip1')
        .append('rect')
        .attr('width', chartWidth)
        .attr('height', height);

    d3.csv('/stuff/hotgym.column_states.2016-01-16.1505.csv')
        .row(function(d) {
            var intKeys = ['n-unpredicted-active-columns',
                           'n-predicted-active-columns',
                           'n-predicted-inactive-columns'];
            intKeys.forEach(function(k) {
                d[k] = parseInt(d[k]);
            });
            return d;
        })
        .get(function(error, timesteps) {
            onTimestepCountKnown(timesteps.length);
            var stackOrder = [{key: 'n-unpredicted-active-columns',
                               color: 'hsl(0,100%,50%)',
                               activeText: 'active',
                               predictedText: 'not predicted'},
                              {key: 'n-predicted-active-columns',
                               color: 'hsl(270,100%,40%)',
                               activeText: 'active',
                               predictedText: 'predicted'},
                              {key: 'n-predicted-inactive-columns',
                               color: 'hsla(210,100%,50%,0.5)',
                               activeText: 'not active',
                               predictedText: 'predicted'}],
                yStackMax = d3.max(timesteps, function(d) {
                    return stackOrder.reduce(function(sum, o) {
                        return sum + d[o.key];
                    }, 0);
                }),
                chartNode = svg.append('g')
                    .attr('class', 'chart')
                    .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')'),
                stackColorScale = d3.scale.ordinal()
                    .domain(stackOrder.map(function(d) { return d.key; }))
                    .range(stackOrder.map(function(d) { return d.color; })),
                help = chartNode.append('g')
                    .attr('transform', 'translate(' + (chartWidth+50) + ',40)');

            // Title and legend
            help.append('text')
                .attr('class', 'noselect')
                .attr('text-anchor', 'right')
                .attr('x', 5)
                .attr('y', 0)
                .text('active and predicted columns, stacked by:');

            var legend = help.append('g')
                    .attr('class', 'noselect')
                    .attr('transform', 'translate(0, 20)'),
                unitWidth = 80,
                color = legend.selectAll('g')
                    .data(stackOrder);
            stackOrder.forEach(function(o, i) {
                var g = legend.append('g')
                        .attr('transform', 'translate(' + i * unitWidth + ',0)');
                g.append('rect')
                    .attr('width', unitWidth)
                    .attr('height', 4)
                    .attr('fill', o.color);
                g.append('text')
                    .attr('x', unitWidth/2)
                    .attr('dy', '-0.24em')
                    .attr('text-anchor', 'middle')
                    .text(o.activeText);
                g.append('text')
                    .attr('x', unitWidth/2)
                    .attr('y', 14)
                    .attr('text-anchor', 'middle')
                    .text(o.predictedText);
            });

            // Chart
            var y = d3.scale.linear()
                    .domain([0, yStackMax])
                    .range([height, 0]),
                drawTimeout = null,
                chart = stackedTimeSeries()
                    .colorScale(stackColorScale)
                    .x(x)
                    .y(y),
                chartNodeInner = chartNode.append('g')
                    .style('clip-path', 'url(#clip1)');

            chartNode.append('rect')
                .attr('fill-opacity', 0)
                .attr('x', 0)
                .attr('y', 0)
                .attr('width', chartWidth)
                .attr('height', height)
                .call(zoom);

            function draw () {
                if (drawTimeout) {
                    clearTimeout(drawTimeout);
                    drawTimeout = null;
                }

                xResample();

                var layers = [];
                stackOrder.forEach(function(o) {
                    var layer = xSamples.map(function(data) {
                        var y = timesteps[data.x][o.key] || 0;
                        return {x: data.x, x0: data.x0, x1: data.x1, y: y};
                    });
                    layers.push(layer);
                });
                d3.layout.stack()(layers); // inserts y0 values
                chartNodeInner.datum(layers)
                    .call(chart);
            }

            onxscalechanged.push(function(maybeCoalesce) {
                chartNodeInner.call(chart.stretch);
                if (!drawTimeout) {
                    drawTimeout = setTimeout(draw, maybeCoalesce ? 250 : 1000/60);
                }
            });

            // y axis
            chartNode.append('g')
                .attr('transform', 'translate(' + chartWidth + ', 0)')
                .attr('class', 'y axis')
                .call(d3.svg.axis()
                      .scale(y)
                      .ticks(4)
                      .tickPadding(2)
                      .tickSize(4)
                      .outerTickSize(0)
                      .orient('right'));

            // Initial change
            draw();
            onxscalechanged.forEach(function (f) { f(); });
        });
})();

//
// SEGMENTS PLOT
//
(function() {
    var margin = {top: 20, right: 300, bottom: 20, left: 0},
        height = 200 - margin.top - margin.bottom;

    var svg = charts.append('svg')
            .attr('width', chartWidth + margin.left + margin.right)
            .attr('height', height + margin.top + margin.bottom);

    var defs = svg.append('defs');
    defs.append('clipPath')
        .attr('id', 'clip2')
        .append('rect')
        .attr('width', chartWidth)
        .attr('height', height);

    d3.text('/stuff/hotgym.segments.2016-01-31.1400.csv', 'text/csv', function(error, contents) {
        var rows = d3.csv.parseRows(contents),
            htmData = rows[0],
            timesteps = rows.splice(1),
            activationThreshold = parseInt(htmData[0]),
            maxTotalSegments = 0,
            maxConnectedSynapsesInSegment = 0;

        onTimestepCountKnown(timesteps.length);

        timesteps.forEach(function(row) {
            var totalSegments = 0;
            for (var i = 0; i < row.length; i++) {
                row[i] = parseInt(row[i]);
                totalSegments += row[i];
                if (i > maxConnectedSynapsesInSegment && row[i] > 0) {
                    maxConnectedSynapsesInSegment = i;
                }
            }
            if (totalSegments > maxTotalSegments) {
                maxTotalSegments = totalSegments;
            }
        });

        var chartNode = svg.append('g')
                .attr('class', 'chart')
                .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')'),
            stackOrder = d3.range(maxConnectedSynapsesInSegment + 1).reverse(),
            colorExtent = ['whitesmoke', 'black'],
            connectedSynapsesColorScale = d3.scale.linear()
                .domain([0, maxConnectedSynapsesInSegment])
                .range(colorExtent),
            stackColorScale = d3.scale.linear()
                .domain([maxConnectedSynapsesInSegment, 0])
                .range(colorExtent),
            help = chartNode.append('g')
                .attr('transform', 'translate(' + (chartWidth+50) + ',40)');

        // Title and legend
        help.append('text')
            .attr('class', 'noselect')
            .attr('text-anchor', 'right')
            .attr('x', 2)
            .attr('y', 0)
            .text('distal segments, stacked by:');

        var legend = help.append('g')
                .attr('class', 'noselect')
                .attr('transform', 'translate(0, 20)'),
            domainWidth = maxConnectedSynapsesInSegment + 1,
            legendWidth = 200,
            unitWidth = legendWidth / domainWidth,
            rect = legend.selectAll('rect')
                .data(d3.range(maxConnectedSynapsesInSegment + 1));
        rect.enter()
            .append('rect')
            .attr('x', function(d, i) { return i * unitWidth; })
            .attr('width', unitWidth)
            .attr('height', 4);
        rect.attr('fill', function(d, i) { return connectedSynapsesColorScale(d); });
        legend.append('text')
            .attr('x', legendWidth / 2)
            .attr('y', -4)
            .attr('text-anchor', 'middle')
            .text('number of connected synapses on segment');
        legend.append('text')
            .attr('y', 14)
            .attr('text-anchor', 'middle')
            .text(0);
        legend.append('text')
            .attr('x', legendWidth)
            .attr('text-anchor', 'middle')
            .attr('y', 14)
            .text(maxConnectedSynapsesInSegment);

        // Chart
        var y = d3.scale.linear()
                .domain([0, maxTotalSegments])
                .range([height, 0]),
            drawTimeout = null,
            chart = stackedTimeSeries()
                .colorScale(stackColorScale)
                .x(x)
                .y(y),
            chartNodeInner = chartNode.append('g')
                .style('clip-path', 'url(#clip2)');

        chartNode.append('rect')
            .attr('fill-opacity', 0)
            .attr('x', 0)
            .attr('y', 0)
            .attr('width', chartWidth)
            .attr('height', height)
            .call(zoom);

        function draw () {
            if (drawTimeout) {
                clearTimeout(drawTimeout);
                drawTimeout = null;
            }

            xResample();

            var layers = [];
            stackOrder.forEach(function(key) {
                var layer = xSamples.map(function(data) {
                    var y = timesteps[data.x][key] || 0;
                    return {x: data.x, x0: data.x0, x1: data.x1, y: y};
                });
                layers.push(layer);
            });
            d3.layout.stack()(layers); // inserts y0 values
            chartNodeInner.datum(layers)
                .call(chart);
        }

        onxscalechanged.push(function(maybeCoalesce) {
            chartNodeInner.call(chart.stretch);
            if (!drawTimeout) {
                drawTimeout = setTimeout(draw, maybeCoalesce ? 250 : 1000/60);
            }
        });

        // Axes
        var yLabels = chartNode.append('g')
                .attr('transform', 'translate(' + (chartWidth + 2) + ', 0)');

        onxscalechanged.push(function () {
            var finalRow = timesteps[Math.floor(x.domain()[1] - 1)],
                spikable = finalRow.slice(activationThreshold)
                    .reduce(function(sum, v) { return sum + v; }, 0),
                total = finalRow.reduce(function(sum, v) { return sum + v; }, 0);
            var groups = yLabels.selectAll('g')
                    .data([[spikable, 'spikable', '0.34em'],
                           [total, 'total', '0.34em']]);
            var enteringGroups = groups.enter()
                    .append('g');
            enteringGroups.append('line')
                .attr('x1', 0)
                .attr('x2', 4)
                .attr('y1', 0)
                .attr('y2', 0)
                .attr('stroke', 'black')
                .attr('stroke-width', 1);
            enteringGroups.append('text')
                .attr('class', 'noselect number')
                .attr('x', 6)
                .attr('dy', function(d, i) { return d[2]; });
            enteringGroups.append('text')
                .attr('class', 'noselect')
                .attr('x', 6)
                .attr('y', 15)
                .text(function(d) { return d[1]; });
            groups.attr('transform', function(d, i) {
                return 'translate(0,' + y(d[0]) + ')';
            })
                .select('.number')
                .text(function(d, i) { return d[0]; });
        });

        // Initial change
        draw();
        onxscalechanged.forEach(function (f) { f(); });
    });
})();

//
// SHARED X AXIS
//
(function() {
    var xAxis = d3.svg.axis()
            .scale(x)
            .tickPadding(3)
            .tickSize(0)
            .outerTickSize(0)
            .orient('bottom')
            .tickFormat(d3.format('d')),
        marginLeft = 3,
        svg = charts.append('svg')
            .attr('width', chartWidth + 10)
            .attr('height', 30)
            .style('position', 'relative')
            .style('left', -marginLeft + 'px')
            .style('top', '-20px'),
        xAxisNode = svg.append('g')
            .attr('class', 'x axis noselect')
            .attr('transform', 'translate(' + marginLeft + ',0)')
            .append('g');
    onxscalechanged.push(function () {
        var extent = x.domain(),
            domainWidth = extent[1] - extent[0],
            pixelsPerTimestep = chartWidth / domainWidth,
            tickShift = pixelsPerTimestep / 2;
        xAxis.ticks(Math.min(domainWidth, 15));
        xAxisNode.call(xAxis);
        xAxisNode.attr('transform', 'translate(' + tickShift + ',' + '0)');
    });

    svg.append('text')
        .attr('class', 'x label noselect')
        .attr('x', marginLeft)
        .attr('y', 25)
        .text('timestep');
})();

</script>
