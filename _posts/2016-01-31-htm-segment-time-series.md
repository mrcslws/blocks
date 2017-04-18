---
layout: post
title: "HTM time series: Number of dendrite segments"
date:   2016-01-31 12:00:00
---

<script src="//d3js.org/d3.v3.min.js" charset="utf-8"></script>
<style type="text/css">
      text {
      font: 10px sans-serif;
      }

      .axis path {
      stroke: none;
      fill: none;
      stroke: none;
      shape-rendering: crispEdges;
      }

      .axis line {
      stroke: none;
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

    tp = model._getTPRegion().getSelf()._tfdr
    htmData = (tp.activationThreshold,)
    csvOutput.writerow(htmData)

    runMethod = model.run
    def myRun(v):
      runResult = runMethod(v)

      nSegmentsByConnectedCount = []
      if hasattr(tp, "connections"): # temporal_memory.py
        for segment, _ in tp.connections._segments.items():
          nConnected = 0
          for syn in tp.connections.synapsesForSegment(segment):
            synapseData = tp.connections.dataForSynapse(syn)
            if synapseData.permanence >= tp.connectedPermanence:
              nConnected += 1

          while len(nSegmentsByConnectedCount) <= nConnected:
            nSegmentsByConnectedCount.append(0)
          nSegmentsByConnectedCount[nConnected] += 1
      else: # tp.py
        for col in xrange(tp.numberOfCols):
          for cell in xrange(tp.cellsPerColumn):
            for segIdx in xrange(tp.getNumSegmentsInCell(col, cell)):
              nConnected = 0
              v = tp.getSegmentOnCell(col, cell, segIdx)
              segData = v[0]
              for _, _, perm in v[1:]:
                if perm >= tp.connectedPerm:
                  nConnected += 1

              while len(nSegmentsByConnectedCount) <= nConnected:
                nSegmentsByConnectedCount.append(0)
              nSegmentsByConnectedCount[nConnected] += 1

      csvOutput.writerow(nSegmentsByConnectedCount)

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
    stackOrder.forEach(function(key) {
        layers.push(rows.map(function(row, i) {
            return {x: i, y: row[key] || 0};
        }));
    });
    d3.layout.stack()(layers); // inserts y0 values
    return layers;
}

function stackedTimeSeries() {
    var colorScale, stepWidth, yScale;

    var chart = function(selection) {
        selection.each(function(data) {
            var layer = d3.select(this).selectAll('.layer').data(data);
            layer.enter()
                .append('path')
                .attr('class', 'layer')
                .attr('shape-rendering', 'crispEdges')
                .attr('fill', function(d, i) { return colorScale(i); })
                .attr('stroke', 'none');
            layer.attr('d', function(ds) {
                return ds.map(function(d) {
                    var x = d.x * stepWidth;
                    var width = stepWidth;

                    if (width >= 4) {
                        x += 1;
                        width -= 2;
                    }

                    var y = yScale(d.y0 + d.y);
                    var height = yScale(d.y0) - yScale(d.y0 + d.y);
                    return "M " + x + " " + y +
                        " l " + width + " " + 0 +
                        " l " + 0 + " " + height +
                        " l " + (-width) + " " + 0 +
                        " Z";
                }).join(' ');
            });
            layer.exit().remove();
        });
    };

    chart.colorScale = function(_) {
        if (!arguments.length) return colorScale;
        colorScale = _;
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

var margin = {top: 40, right: 120, bottom: 20, left: 0},
    contextWidth = 200 - margin.left - margin.right,
    contextHeight = 100 - margin.top - margin.bottom,
    focusWidth = 800 - margin.left - margin.right,
    focusHeight = 200 - margin.top - margin.bottom;

var svg = d3.select('#putItHere').append('svg')
        .attr('width', contextWidth + focusWidth + 2*(margin.left + margin.right))
        .attr('height', focusHeight + margin.top + margin.bottom + 70);

var defs = svg.append('defs');
defs.append('clipPath')
    .attr('id', 'clip')
    .append('rect')
    .attr('width', focusWidth)
    .attr('height', focusHeight);

d3.text('/stuff/hotgym.segments.2016-01-31.1400.csv', 'text/csv', function(error, contents) {
    var rows = d3.csv.parseRows(contents),
        htmData = rows[0],
        timesteps = rows.splice(1),
        activationThreshold = parseInt(htmData[0]),
        maxTotalSegments = 0,
        maxConnectedSynapsesInSegment = 0;

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

    var stackOrder = d3.range(maxConnectedSynapsesInSegment + 1).reverse(),
        colorExtent = ['whitesmoke', 'black'],
        connectedSynapsesColorScale = d3.scale.linear()
            .domain([0, maxConnectedSynapsesInSegment])
            .range(colorExtent),
        stackColorScale = d3.scale.linear()
            .domain([maxConnectedSynapsesInSegment, 0])
            .range(colorExtent);

    // The context
    var contextX = d3.scale.linear()
            .domain([0, timesteps.length])
            .range([0, contextWidth]),
        contextY = d3.scale.linear()
            .domain([0, maxTotalSegments])
            .range([contextHeight, 0]),
        context = svg.append('g')
            .attr("transform", "translate(" + margin.left + "," + margin.top + ")"),
        contextSample = sampleInts(contextWidth, 0, timesteps.length, [])
            .map(function(i) { return timesteps[i]; });
    context.append('g')
        .datum(rowsAsStackedLayers(contextSample, stackOrder))
        .call(stackedTimeSeries()
              .colorScale(stackColorScale)
              .stepWidth(contextWidth / contextSample.length)
              .yScale(contextY));

    // Shared state between the brush and the focus
    var focusX = d3.scale.linear()
            .domain([0, Math.min(timesteps.length, focusWidth)])
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
                        extent[1] = Math.min(focusWidth, timesteps.length);
                    }
                    else {
                        extent[1] = Math.min(extent[1] + radius, timesteps.length);
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
            .attr('height', contextHeight + 7);
    });

    // The focus
    var focusY = d3.scale.linear()
            .domain([0, maxTotalSegments])
            .range([focusHeight, 0]),
        focusLeft = margin.left + contextWidth + margin.right + margin.left,
        focus = svg.append('g')
            .attr('class', 'focus')
            .attr('transform', 'translate(' + focusLeft + ',' + margin.top + ')'),
        focusChart = stackedTimeSeries()
            .colorScale(stackColorScale)
            .yScale(focusY),
        focusPicture = focus.append('g'),
        previousSampleIndices = [];
    onfocusxchanged.push(function () {
        var extent = focusX.domain();
        var focusSampleIndices = sampleInts(focusWidth, extent[0], extent[1],
                                            previousSampleIndices);
        var sampled = focusSampleIndices.map(function(i) { return timesteps[i]; });
        focusChart.stepWidth(focusWidth / sampled.length);
        focusPicture.datum(rowsAsStackedLayers(sampled, stackOrder))
            .call(focusChart);
        previousSampleIndices = focusSampleIndices;
    });

    var xAxis = d3.svg.axis()
            .scale(focusX)
            .tickPadding(3)
            .tickSize(0)
            .outerTickSize(0)
            .orient('bottom')
            .tickFormat(d3.format('timestep %d')),
        xAxisNode = focus.append('g')
            .attr('class', 'x axis')
            .attr('transform', 'translate(0,' + (focusHeight + 5) + ')');
    onfocusxchanged.push(function () {
        var extent = focusX.domain();
        var domainWidth = extent[1] - extent[0];
        var tickShift = focusChart.stepWidth() / 2;
        xAxis.ticks(Math.min(domainWidth - 1, 20));
        xAxisNode.call(xAxis);
        xAxisNode.attr('transform', 'translate(' + tickShift + ',' +
                       (focusHeight + 5) + ')');
    });

    focus.append('text')
        .attr('class', 'x label')
        .attr('x', -2)
        .attr('y', focusHeight + 28)
        .text('timestep');

    var yLabels = focus.append('g')
            .attr('transform', 'translate(' + (focusWidth + 2) + ', 0)');

    onfocusxchanged.push(function () {
        var finalRow = timesteps[previousSampleIndices[previousSampleIndices.length - 1]],
            spikable = finalRow.slice(activationThreshold)
                .reduce(function(sum, v) { return sum + v; }, 0),
            total = finalRow.reduce(function(sum, v) { return sum + v; }, 0);
        var groups = yLabels.selectAll('g')
                .data([[spikable, spikable + ' spikable segments'],
                       [total, total + ' total segments']]);
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
            .attr('x', 6)
            .attr('dy', '.32em');
        groups.attr('transform', function(d, i) {
            return 'translate(0,' + focusY(d[0]) + ')';
        })
            .select('text')
            .text(function(d, i) { return d[1]; });
    });

    // Zoomable focus
    var isZooming = false;
    var zoom = d3.behavior.zoom()
            .x(focusX)
            .scaleExtent([0, 50])
            .on("zoom", function() {
                var extent = focusX.domain();
                var corrected = extent.map(function(i) {
                    return Math.min(Math.max(0, Math.round(i)), timesteps.length);
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
        .attr('height', focusHeight)
        .call(zoom);

    focus.append('text')
        .attr('text-anchor', 'right')
        .attr('x', 0)
        .attr('y', 0)
        .text('distal segments, stacked by:');

    var legend = focus.append('g')
            .attr('transform', 'translate(132, -5)'),
        domainWidth = maxConnectedSynapsesInSegment + 1,
        legendWidth = 200,
        unitWidth = legendWidth / domainWidth;

    var rect = legend.selectAll('rect')
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

    // Initial change
    onfocusxchanged.forEach(function (f) { f(); });
});

{% endhighlight %}

<script>
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
        stackOrder.forEach(function(key) {
            layers.push(rows.map(function(row, i) {
                return {x: i, y: row[key] || 0};
            }));
        });
        d3.layout.stack()(layers); // inserts y0 values
        return layers;
    }

    function stackedTimeSeries() {
        var colorScale, stepWidth, yScale;

        var chart = function(selection) {
            selection.each(function(data) {
                var layer = d3.select(this).selectAll('.layer').data(data);
                layer.enter()
                    .append('path')
                    .attr('class', 'layer')
                    .attr('shape-rendering', 'crispEdges')
                    .attr('fill', function(d, i) { return colorScale(i); })
                    .attr('stroke', 'none');
                layer.attr('d', function(ds) {
                    return ds.map(function(d) {
                        var x = d.x * stepWidth;
                        var width = stepWidth;

                        if (width >= 4) {
                            x += 1;
                            width -= 2;
                        }

                        var y = yScale(d.y0 + d.y);
                        var height = yScale(d.y0) - yScale(d.y0 + d.y);
                        return "M " + x + " " + y +
                            " l " + width + " " + 0 +
                            " l " + 0 + " " + height +
                            " l " + (-width) + " " + 0 +
                            " Z";
                    }).join(' ');
                });
                layer.exit().remove();
            });
        };

        chart.colorScale = function(_) {
            if (!arguments.length) return colorScale;
            colorScale = _;
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

    var margin = {top: 40, right: 120, bottom: 20, left: 0},
        contextWidth = 200 - margin.left - margin.right,
        contextHeight = 100 - margin.top - margin.bottom,
        focusWidth = 800 - margin.left - margin.right,
        focusHeight = 200 - margin.top - margin.bottom;

    var svg = d3.select('#putItHere').append('svg')
            .attr('width', contextWidth + focusWidth + 2*(margin.left + margin.right))
            .attr('height', focusHeight + margin.top + margin.bottom + 70);

    var defs = svg.append('defs');
    defs.append('clipPath')
        .attr('id', 'clip')
        .append('rect')
        .attr('width', focusWidth)
        .attr('height', focusHeight);

    d3.text('/stuff/hotgym.segments.2016-01-31.1400.csv', 'text/csv', function(error, contents) {
        var rows = d3.csv.parseRows(contents),
            htmData = rows[0],
            timesteps = rows.splice(1),
            activationThreshold = parseInt(htmData[0]),
            maxTotalSegments = 0,
            maxConnectedSynapsesInSegment = 0;

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

        var stackOrder = d3.range(maxConnectedSynapsesInSegment + 1).reverse(),
            colorExtent = ['whitesmoke', 'black'],
            connectedSynapsesColorScale = d3.scale.linear()
                .domain([0, maxConnectedSynapsesInSegment])
                .range(colorExtent),
            stackColorScale = d3.scale.linear()
                .domain([maxConnectedSynapsesInSegment, 0])
                .range(colorExtent);

        // The context
        var contextX = d3.scale.linear()
                .domain([0, timesteps.length])
                .range([0, contextWidth]),
            contextY = d3.scale.linear()
                .domain([0, maxTotalSegments])
                .range([contextHeight, 0]),
            context = svg.append('g')
                .attr("transform", "translate(" + margin.left + "," + margin.top + ")"),
            contextSample = sampleInts(contextWidth, 0, timesteps.length, [])
                .map(function(i) { return timesteps[i]; });
        context.append('g')
            .datum(rowsAsStackedLayers(contextSample, stackOrder))
            .call(stackedTimeSeries()
                  .colorScale(stackColorScale)
                  .stepWidth(contextWidth / contextSample.length)
                  .yScale(contextY));

        // Shared state between the brush and the focus
        var focusX = d3.scale.linear()
                .domain([0, Math.min(timesteps.length, focusWidth)])
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
                            extent[1] = Math.min(focusWidth, timesteps.length);
                        }
                        else {
                            extent[1] = Math.min(extent[1] + radius, timesteps.length);
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
                .attr('height', contextHeight + 7);
        });

        // The focus
        var focusY = d3.scale.linear()
                .domain([0, maxTotalSegments])
                .range([focusHeight, 0]),
            focusLeft = margin.left + contextWidth + margin.right + margin.left,
            focus = svg.append('g')
                .attr('class', 'focus')
                .attr('transform', 'translate(' + focusLeft + ',' + margin.top + ')'),
            focusChart = stackedTimeSeries()
                .colorScale(stackColorScale)
                .yScale(focusY),
            focusPicture = focus.append('g'),
            previousSampleIndices = [];
        onfocusxchanged.push(function () {
            var extent = focusX.domain();
            var focusSampleIndices = sampleInts(focusWidth, extent[0], extent[1],
                                                previousSampleIndices);
            var sampled = focusSampleIndices.map(function(i) { return timesteps[i]; });
            focusChart.stepWidth(focusWidth / sampled.length);
            focusPicture.datum(rowsAsStackedLayers(sampled, stackOrder))
                .call(focusChart);
            previousSampleIndices = focusSampleIndices;
        });

        var xAxis = d3.svg.axis()
                .scale(focusX)
                .tickPadding(3)
                .tickSize(0)
                .outerTickSize(0)
                .orient('bottom')
                .tickFormat(d3.format('timestep %d')),
            xAxisNode = focus.append('g')
                .attr('class', 'x axis')
                .attr('transform', 'translate(0,' + (focusHeight + 5) + ')');
        onfocusxchanged.push(function () {
            var extent = focusX.domain();
            var domainWidth = extent[1] - extent[0];
            var tickShift = focusChart.stepWidth() / 2;
            xAxis.ticks(Math.min(domainWidth, 20));
            xAxisNode.call(xAxis);
            xAxisNode.attr('transform', 'translate(' + tickShift + ',' +
                           (focusHeight + 5) + ')');
        });

        focus.append('text')
            .attr('class', 'x label')
            .attr('x', -2)
            .attr('y', focusHeight + 28)
            .text('timestep');

        var yLabels = focus.append('g')
                .attr('transform', 'translate(' + (focusWidth + 2) + ', 0)');

        onfocusxchanged.push(function () {
            var finalRow = timesteps[previousSampleIndices[previousSampleIndices.length - 1]],
                spikable = finalRow.slice(activationThreshold)
                    .reduce(function(sum, v) { return sum + v; }, 0),
                total = finalRow.reduce(function(sum, v) { return sum + v; }, 0);
            var groups = yLabels.selectAll('g')
                    .data([[spikable, spikable + ' spikable segments'],
                           [total, total + ' total segments']]);
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
                .attr('x', 6)
                .attr('dy', '.32em');
            groups.attr('transform', function(d, i) {
                return 'translate(0,' + focusY(d[0]) + ')';
            })
                .select('text')
                .text(function(d, i) { return d[1]; });
        });

        // Zoomable focus
        var isZooming = false;
        var zoom = d3.behavior.zoom()
                .x(focusX)
                .scaleExtent([0, 50])
                .on("zoom", function() {
                    var extent = focusX.domain();
                    var corrected = extent.map(function(i) {
                        return Math.min(Math.max(0, Math.round(i)), timesteps.length);
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
            .attr('height', focusHeight)
            .call(zoom);

        focus.append('text')
            .attr('text-anchor', 'right')
            .attr('x', 0)
            .attr('y', 0)
            .text('distal segments, stacked by:');

        var legend = focus.append('g')
                .attr('transform', 'translate(132, -5)'),
            domainWidth = maxConnectedSynapsesInSegment + 1,
            legendWidth = 200,
            unitWidth = legendWidth / domainWidth;

        var rect = legend.selectAll('rect')
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

        // Initial change
        onfocusxchanged.forEach(function (f) { f(); });
    });
})();

</script>
