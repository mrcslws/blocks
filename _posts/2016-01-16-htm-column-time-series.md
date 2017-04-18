---
layout: post
title: "HTM time series: Frequencies of column states"
date:   2016-01-16 12:00:00
---

<script src="//d3js.org/d3.v3.min.js" charset="utf-8"></script>
<style type="text/css">
text {
    font: 10px sans-serif;
}

.axis path {
    stroke: none;
    fill: none;
    stroke: #000;
    shape-rendering: crispEdges;
}

.axis line {
    stroke: #000;
    shape-rendering: crispEdges;
}

.brush .extent {
    stroke: #fff;
    fill-opacity: .125;
    shape-rendering: crispEdges;
}

.focus .layer rect {
    clip-path: url(#clip);
}
</style>

Here's a visualization.

<div id="putItHere" style="position:absolute; left:0; right:0; text-align:center"></div>
<div style="height:250px;"></div>

HTM column states are shown across time. This follows
[Felix](http://floybix.github.io/)'s column state frequencies design that is
built into Sanity
([example](https://nupic-community.github.io/sanity/demos/coordinates_2d.html),
click the "time plots" tab). Red means "active column, not predicted", Blue
means "predicted column, not active", and Purple means "active column,
predicted".

Because there are thousands of timesteps, your screen doesn't have enough pixels
to show every timestep at once, so the data is presented in two charts: the
context and the focus. You adjust the focus directly (by zooming or
click-dragging) or by choosing an interval on the context.

When there are more data points than pixels, the graphic has to choose how to
handle it. There are two approaches:

1. Smash the data into the required number of pixels, averaging it.
   - This is what Sanity does.
2. Sample the data, ignoring other points.
   - **This is what this chart does.**


## The code

### Fetch the data

Patch a NuPIC model to output a CSV.

{% highlight python %}

class ModelPatcher(object):
  def __init__(self):
    filename = 'out.%s.csv' % time.strftime('%Y%m%d-%H.%M.%S')
    self.outfile = open(filename, 'w')

  def patch(self, model):
    csvOutput = csv.writer(self.outfile)

    headerRow = [
      'n-unpredicted-active-columns',
      'n-predicted-inactive-columns',
      'n-predicted-active-columns',
    ]
    csvOutput.writerow(headerRow)

    runMethod = model.run
    def myRun(v):
      tp = model._getTPRegion().getSelf()._tfdr
      npPredictedCells = tp.getPredictedState().reshape(-1).nonzero()[0]
      predictedColumns = set(np.unique(npPredictedCells / tp.cellsPerColumn).tolist())

      runResult = runMethod(v)

      spOutput = model._getSPRegion().getSelf()._spatialPoolerOutput
      activeColumns = set(spOutput.nonzero()[0].tolist())

      row = (
        len(activeColumns - predictedColumns),
        len(predictedColumns - activeColumns),
        len(activeColumns & predictedColumns),
      )
      csvOutput.writerow(row)

      return runResult

    model.run = myRun

  def onFinished(self):
    self.outfile.close()

## Then, somewhere in code...
patcher = ModelPatcher()
patcher.patch(model)

# ...

patcher.onFinished()

{% endhighlight %}

### Draw the data

Here we use [d3.js](http://d3js.org/) to draw the CSV data.

**CSS**

{% highlight css %}

text {
    font: 10px sans-serif;
}

.axis path {
    stroke: none;
    fill: none;
    stroke: #000;
    shape-rendering: crispEdges;
}

.axis line {
    stroke: #000;
    shape-rendering: crispEdges;
}

.brush .extent {
    stroke: #fff;
    fill-opacity: .125;
    shape-rendering: crispEdges;
}

.focus .layer rect {
    clip-path: url(#clip);
}

{% endhighlight %}

**JavaScript**

{% highlight javascript %}

// preferredInts must be sorted
function sampleInts(n, start, stop, preferredInts) {
    if (n < stop - start) {
        var ret = new Array(n);
        var inc = (stop - start) / n;
        var intervalStart = start;
        var iPreferred = 0;
        for(var i = 0; i < n; i++) {
            var intervalStop = start + ((i + 1) * inc);
            while (iPreferred < preferredInts.length &&
                   preferredInts[iPreferred] < intervalStart) {
                iPreferred++;
            }
            if (iPreferred < preferredInts.length &&
                preferredInts[iPreferred] < intervalStop) {
                // When zooming / panning, the behavior is more intuitive if
                // we reuse samples when possible rather than throwing out
                // the data and grabbing a new random sample.
                ret[i] = preferredInts[iPreferred];
            }
            else {
                // Choose randomly from the interval.
                // Otherwise with repeating patterns we'll have aliasing.
                ret[i] = Math.floor(Math.random() * (intervalStop - intervalStart)
                                    + intervalStart);
            }
            intervalStart = intervalStop;
        }
        return ret;
    }
    else {
        return d3.range(Math.floor(start), stop);
    }
}

function rowsAsStackedLayers(rows, stackOrder) {
    var layers = [];
    stackOrder.forEach(function(label) {
        layers.push(rows.map(function(row, i) {
            return {x: i, y: row[label]};
        }));
    });
    d3.layout.stack()(layers); // inserts y0 values
    return layers;
}

function stackedTimeSeries() {
    var colors, stepWidth, yScale;

    var chart = function(selection) {
        selection.each(function(data) {
            var layer = d3.select(this).selectAll('.layer').data(data);
            layer.enter()
                .append('g')
                .attr('class', 'layer')
                .attr('shape-rendering', 'crispEdges')
                .style('fill', function(d, i) { return colors[i]; });

            var rect = layer.selectAll('rect')
                    .data(function(d) { return d; });
            rect.enter().append('rect');
            rect.attr('x', function(d) { return d.x * stepWidth; })
                .attr('y', function(d) { return yScale(d.y0 + d.y); })
                .attr('width', stepWidth)
                .attr('height', function(d) { return yScale(d.y0) - yScale(d.y0 + d.y); });
            rect.exit().remove();
        });
    };

    chart.colors = function(_) {
        if (!arguments.length) return colors;
        colors = _;
        return chart;
    };

    chart.stepWidth = function(_) {
        if (!arguments.length) return stepWidth;
        stepWidth = _;
        return chart;
    };

    chart.yScale = function(_) {
        if (!arguments.length) return yScale;
        yScale = _;
        return chart;
    };

    return chart;
}

var margin = {top: 40, right: 20, bottom: 20, left: 50},
    contextWidth = 200 - margin.left - margin.right,
    focusWidth = 800 - margin.left - margin.right,
    height = 200 - margin.top - margin.bottom;

var svg = d3.select('#putItHere').append('svg')
        .attr('width', contextWidth + focusWidth + 2*(margin.left + margin.right))
        .attr('height', height + margin.top + margin.bottom);

svg.append('defs')
    .append('clipPath')
    .attr('id', 'clip')
    .append('rect')
    .attr('width', focusWidth)
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
    .get(function(error, rows) {
        var stackOrder = ['n-unpredicted-active-columns',
                          'n-predicted-active-columns',
                          'n-predicted-inactive-columns'],
            colors = ['red', 'purple', 'blue'],
            yStackMax = d3.max(rows, function(d) {
                return stackOrder.reduce(function(sum, k) { return sum + d[k]; }, 0);
            });

        // The context
        var contextX = d3.scale.linear()
                .domain([0, rows.length])
                .range([0, contextWidth]),
            contextY = d3.scale.linear()
                .domain([0, yStackMax])
                .range([height, 0]),
            context = svg.append('g')
                .attr("transform", "translate(" + margin.left + "," + margin.top + ")"),
            contextSample = sampleInts(contextWidth, 0, rows.length, [])
                .map(function(i) { return rows[i]; });
        context.append('g')
            .datum(rowsAsStackedLayers(contextSample, stackOrder))
            .call(stackedTimeSeries()
                  .colors(colors)
                  .stepWidth(contextWidth / contextSample.length)
                  .yScale(contextY));
        context.append('g')
            .attr('class', 'x axis')
            .attr('transform', 'translate(0,' + height + ')')
            .call(d3.svg.axis()
                  .scale(contextX)
                  .tickPadding(2)
                  .tickSize(4)
                  .tickValues(contextX.domain())
                  .outerTickSize(0)
                  .orient('bottom'));
        context.append('g')
            .attr('class', 'y axis')
            .call(d3.svg.axis()
                  .scale(contextY)
                  .ticks(4)
                  .tickPadding(2)
                  .tickSize(4)
                  .outerTickSize(0)
                  .orient('left'));

        // Shared state between the brush and the focus
        var focusX = d3.scale.linear()
                .domain([0, Math.min(rows.length, focusWidth)])
                .range([0, focusWidth]),
            onfocusxchanged = []; // callbacks

        // The brush
        var brush = d3.svg.brush()
                .x(contextX)
                .on('brush', function() {
                    var extent = brush.extent();
                    if (brush.empty()) {
                        // Center the brush on the click point.
                        var radius = Math.floor(focusWidth / 2);
                        if (extent[0] - radius <= 0) {
                            extent[0] = 0;
                            extent[1] = Math.min(focusWidth, rows.length);
                        }
                        else {
                            extent[1] = Math.min(extent[1] + radius, rows.length);
                            extent[0] = Math.max(0, extent[1] - focusWidth);
                        }
                    }
                    focusX.domain(extent);
                    onfocusxchanged.forEach(function (f) { f(); });
                });
        var brushNode = context.append('g')
                .attr('class', 'x brush');
        onfocusxchanged.push(function() {
            brush.extent(focusX.domain());
            brushNode.call(brush)
                .selectAll('rect')
                .attr('y', -6)
                .attr('height', height + 7);
        });

        // The focus
        var focusY = d3.scale.linear()
                .domain([0, yStackMax])
                .range([height, 0]),
            focusLeft = margin.left + contextWidth + margin.right + margin.left,
            focus = svg.append('g')
                .attr('class', 'focus')
                .attr('transform', 'translate(' + focusLeft + ',' + margin.top + ')'),
            focusChart = stackedTimeSeries()
                .colors(colors)
                .yScale(focusY),
            focusPicture = focus.append('g'),
            previousSampleIndices = [];
        onfocusxchanged.push(function () {
            var extent = focusX.domain();
            var focusSampleIndices = sampleInts(focusWidth, extent[0], extent[1],
                                                previousSampleIndices);
            var sampled = focusSampleIndices.map(function(i) { return rows[i]; });
            focusChart.stepWidth(focusWidth / sampled.length);
            focusPicture.datum(rowsAsStackedLayers(sampled, stackOrder))
                .call(focusChart);
            previousSampleIndices = focusSampleIndices;
        });
        var xAxis = d3.svg.axis()
                .scale(focusX)
                .tickPadding(2)
                .tickSize(4)
                .outerTickSize(0)
                .orient('bottom'),
            xAxisNode = focus.append('g')
                .attr('class', 'x axis')
                .attr('transform', 'translate(0,' + height + ')');
        onfocusxchanged.push(function () {
            xAxis.tickValues(focusX.domain());
            xAxisNode.call(xAxis);
        });

        // Zoomable focus
        var isZooming = false;
        var zoom = d3.behavior.zoom()
                .x(focusX)
                .scaleExtent([0, 50])
                .on("zoom", function() {
                    var extent = focusX.domain();
                    var corrected = extent.map(function(i) {
                        return Math.min(Math.max(0, Math.round(i)), rows.length);
                    });
                    focusX.domain(corrected);
                    isZooming = true;
                    onfocusxchanged.forEach(function (f) { f(); });
                    isZooming = false;
                });
        onfocusxchanged.push(function () {
            if (!isZooming) {
                zoom.x(focusX);
            }
        });
        focus.append('rect')
            .attr('fill-opacity', 0)
            .attr('x', 0)
            .attr('y', 0)
            .attr('width', focusWidth)
            .attr('height', height)
            .call(zoom);

        // Initial change
        onfocusxchanged.forEach(function (f) { f(); });
    });

{% endhighlight %}

<script type="text/javascript">
(function() {
    // preferredInts must be sorted
    function sampleInts(n, start, stop, preferredInts) {
        if (n < stop - start) {
            var ret = new Array(n);
            var inc = (stop - start) / n;
            var intervalStart = start;
            var iPreferred = 0;
            for(var i = 0; i < n; i++) {
                var intervalStop = start + ((i + 1) * inc);
                while (iPreferred < preferredInts.length &&
                       preferredInts[iPreferred] < intervalStart) {
                    iPreferred++;
                }
                if (iPreferred < preferredInts.length &&
                    preferredInts[iPreferred] < intervalStop) {
                    // When zooming / panning, the behavior is more intuitive if
                    // we reuse samples when possible rather than throwing out
                    // the data and grabbing a new random sample.
                    ret[i] = preferredInts[iPreferred];
                }
                else {
                    // Choose randomly from the interval.
                    // Otherwise with repeating patterns we'll have aliasing.
                    ret[i] = Math.floor(Math.random() * (intervalStop - intervalStart)
                                        + intervalStart);
                }
                intervalStart = intervalStop;
            }
            return ret;
        }
        else {
            return d3.range(Math.floor(start), stop);
        }
    }

    function rowsAsStackedLayers(rows, stackOrder) {
        var layers = [];
        stackOrder.forEach(function(label) {
            layers.push(rows.map(function(row, i) {
                return {x: i, y: row[label]};
            }));
        });
        d3.layout.stack()(layers); // inserts y0 values
        return layers;
    }

    function stackedTimeSeries() {
        var colors, stepWidth, yScale;

        var chart = function(selection) {
            selection.each(function(data) {
                var layer = d3.select(this).selectAll('.layer').data(data);
                layer.enter()
                    .append('g')
                    .attr('class', 'layer')
                    .attr('shape-rendering', 'crispEdges')
                    .style('fill', function(d, i) { return colors[i]; });

                var rect = layer.selectAll('rect')
                        .data(function(d) { return d; });
                rect.enter().append('rect');
                rect.attr('x', function(d) { return d.x * stepWidth; })
                    .attr('y', function(d) { return yScale(d.y0 + d.y); })
                    .attr('width', stepWidth)
                    .attr('height', function(d) { return yScale(d.y0) - yScale(d.y0 + d.y); });
                rect.exit().remove();
            });
        };

        chart.colors = function(_) {
            if (!arguments.length) return colors;
            colors = _;
            return chart;
        };

        chart.stepWidth = function(_) {
            if (!arguments.length) return stepWidth;
            stepWidth = _;
            return chart;
        };

        chart.yScale = function(_) {
            if (!arguments.length) return yScale;
            yScale = _;
            return chart;
        };

        return chart;
    }

    var margin = {top: 40, right: 20, bottom: 20, left: 50},
        contextWidth = 200 - margin.left - margin.right,
        focusWidth = 800 - margin.left - margin.right,
        height = 200 - margin.top - margin.bottom;

    var svg = d3.select('#putItHere').append('svg')
            .attr('width', contextWidth + focusWidth + 2*(margin.left + margin.right))
            .attr('height', height + margin.top + margin.bottom);

    svg.append('defs')
        .append('clipPath')
        .attr('id', 'clip')
        .append('rect')
        .attr('width', focusWidth)
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
        .get(function(error, rows) {
            var stackOrder = ['n-unpredicted-active-columns',
                              'n-predicted-active-columns',
                              'n-predicted-inactive-columns'],
                colors = ['red', 'purple', 'blue'],
                yStackMax = d3.max(rows, function(d) {
                    return stackOrder.reduce(function(sum, k) { return sum + d[k]; }, 0);
                });

            // The context
            var contextX = d3.scale.linear()
                    .domain([0, rows.length])
                    .range([0, contextWidth]),
                contextY = d3.scale.linear()
                    .domain([0, yStackMax])
                    .range([height, 0]),
                context = svg.append('g')
                    .attr("transform", "translate(" + margin.left + "," + margin.top + ")"),
                contextSample = sampleInts(contextWidth, 0, rows.length, [])
                    .map(function(i) { return rows[i]; });
            context.append('g')
                .datum(rowsAsStackedLayers(contextSample, stackOrder))
                .call(stackedTimeSeries()
                      .colors(colors)
                      .stepWidth(contextWidth / contextSample.length)
                      .yScale(contextY));
            context.append('g')
                .attr('class', 'x axis')
                .attr('transform', 'translate(0,' + height + ')')
                .call(d3.svg.axis()
                      .scale(contextX)
                      .tickPadding(2)
                      .tickSize(4)
                      .tickValues(contextX.domain())
                      .outerTickSize(0)
                      .orient('bottom'));
            context.append('g')
                .attr('class', 'y axis')
                .call(d3.svg.axis()
                      .scale(contextY)
                      .ticks(4)
                      .tickPadding(2)
                      .tickSize(4)
                      .outerTickSize(0)
                      .orient('left'));

            // Shared state between the brush and the focus
            var focusX = d3.scale.linear()
                    .domain([0, Math.min(rows.length, focusWidth)])
                    .range([0, focusWidth]),
                onfocusxchanged = []; // callbacks

            // The brush
            var brush = d3.svg.brush()
                    .x(contextX)
                    .on('brush', function() {
                        var extent = brush.extent();
                        if (brush.empty()) {
                            // Center the brush on the click point.
                            var radius = Math.floor(focusWidth / 2);
                            if (extent[0] - radius <= 0) {
                                extent[0] = 0;
                                extent[1] = Math.min(focusWidth, rows.length);
                            }
                            else {
                                extent[1] = Math.min(extent[1] + radius, rows.length);
                                extent[0] = Math.max(0, extent[1] - focusWidth);
                            }
                        }
                        focusX.domain(extent);
                        onfocusxchanged.forEach(function (f) { f(); });
                    });
            var brushNode = context.append('g')
                    .attr('class', 'x brush');
            onfocusxchanged.push(function() {
                brush.extent(focusX.domain());
                brushNode.call(brush)
                    .selectAll('rect')
                    .attr('y', -6)
                    .attr('height', height + 7);
            });

            // The focus
            var focusY = d3.scale.linear()
                    .domain([0, yStackMax])
                    .range([height, 0]),
                focusLeft = margin.left + contextWidth + margin.right + margin.left,
                focus = svg.append('g')
                    .attr('class', 'focus')
                    .attr('transform', 'translate(' + focusLeft + ',' + margin.top + ')'),
                focusChart = stackedTimeSeries()
                    .colors(colors)
                    .yScale(focusY),
                focusPicture = focus.append('g'),
                previousSampleIndices = [];
            onfocusxchanged.push(function () {
                var extent = focusX.domain();
                var focusSampleIndices = sampleInts(focusWidth, extent[0], extent[1],
                                                    previousSampleIndices);
                var sampled = focusSampleIndices.map(function(i) { return rows[i]; });
                focusChart.stepWidth(focusWidth / sampled.length);
                focusPicture.datum(rowsAsStackedLayers(sampled, stackOrder))
                    .call(focusChart);
                previousSampleIndices = focusSampleIndices;
            });
            var xAxis = d3.svg.axis()
                    .scale(focusX)
                    .tickPadding(2)
                    .tickSize(4)
                    .outerTickSize(0)
                    .orient('bottom'),
                xAxisNode = focus.append('g')
                    .attr('class', 'x axis')
                    .attr('transform', 'translate(0,' + height + ')');
            onfocusxchanged.push(function () {
                xAxis.tickValues(focusX.domain());
                xAxisNode.call(xAxis);
            });

            // Zoomable focus
            var isZooming = false;
            var zoom = d3.behavior.zoom()
                    .x(focusX)
                    .scaleExtent([0, 50])
                    .on("zoom", function() {
                        var extent = focusX.domain();
                        var corrected = extent.map(function(i) {
                            return Math.min(Math.max(0, Math.round(i)), rows.length);
                        });
                        focusX.domain(corrected);
                        isZooming = true;
                        onfocusxchanged.forEach(function (f) { f(); });
                        isZooming = false;
                    });
            onfocusxchanged.push(function () {
                if (!isZooming) {
                    zoom.x(focusX);
                }
            });
            focus.append('rect')
                .attr('fill-opacity', 0)
                .attr('x', 0)
                .attr('y', 0)
                .attr('width', focusWidth)
                .attr('height', height)
                .call(zoom);

            // Initial change
            onfocusxchanged.forEach(function (f) { f(); });
        });
})();
</script>
