---
layout: post
title: "See your HTM run: Stacks of time series"
date:   2016-02-15 12:00:00
---

<script src="//d3js.org/d3.v3.min.js" charset="utf-8"></script>
<script>
function insertCharts(node, inputDataUrl, columnStatesUrl, segmentsUrl, segmentLearningUrl) {
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
        container = d3.select(node).append('div')
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
        function interpolateZoom(translate, scale, onchange) {
            var self = this;
            return d3.transition().duration(350).tween('zoom', function () {
                var iTranslate = d3.interpolate(zoom.translate(), translate),
                    iScale = d3.interpolate(zoom.scale(), scale);
                return function (t) {
                    zoom
                        .scale(iScale(t))
                        .translate(iTranslate(t));
                    onchange();
                };
            });
        }

        function zoomTowardCenter(scale, animate) {
            var extent = x.domain(),
                center = ((extent[1] - extent[0]) / 2) + extent[0],
                timestepsPerPixel = 1/scale,
                nTimesteps = chartWidth * timestepsPerPixel,
                newLeftmost = Math.max(0, center - (nTimesteps/2)),
                translate = [-newLeftmost * scale, 0];

            if (animate) {
                interpolateZoom(translate, scale,
                                function () {
                                    onxscalechanged.forEach(function(f) { f(true); });
                                });
            }
            else {
                zoom.scale(scale);
                zoom.translate(translate);
                onxscalechanged.forEach(function(f) { f(false); });
            }
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
                zoomTowardCenter(grooveY.invert(y), false);
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
    function layeredTimeSeries() {
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
    // SHARED X AXIS: TOP
    //
    (function() {
        var xAxis = d3.svg.axis()
                .scale(x)
                .tickPadding(3)
                .tickSize(0)
                .outerTickSize(0)
                .orient('top')
                .tickFormat(d3.format('d')),
            marginLeft = 3,
            svg = charts.append('svg')
                .attr('width', chartWidth + 10)
                .attr('height', 30)
                .style('position', 'relative')
                .style('left', -marginLeft + 'px'),
            xAxisNode = svg.append('g')
                .attr('class', 'x axis noselect')
                .attr('transform', 'translate(' + marginLeft + ',30)')
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
    })();

    //
    // INPUT DATA PLOT
    //
    (function() {
        var margin = {top: 4, right: 300, bottom: 4, left: 0},
            height = 60 - margin.top - margin.bottom;

        var svg = charts.append('svg')
                .attr('width', chartWidth + margin.left + margin.right)
                .attr('height', height + margin.top + margin.bottom);

        var defs = svg.append('defs');
        defs.append('clipPath')
            .attr('id', 'clip0')
            .append('rect')
            .attr('width', chartWidth)
            .attr('height', height);

        d3.csv(inputDataUrl)
            .row(function(d) {
                var intKeys = ['input'];
                intKeys.forEach(function(k) {
                    d[k] = parseInt(d[k]);
                });
                return d;
            })
            .get(function(error, timesteps) {
                onTimestepCountKnown(timesteps.length);
                var chartNode = svg.append('g')
                        .attr('class', 'chart')
                        .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')'),
                    help = chartNode.append('g')
                        .attr('transform', 'translate(' + (chartWidth+50) + ',10)');

                // Title and legend
                help.append('text')
                    .attr('class', 'noselect')
                    .attr('text-anchor', 'right')
                    .attr('x', 5)
                    .attr('y', 0)
                    .text('input data');

                // Chart
                var y = d3.scale.linear()
                        .domain(d3.extent(timesteps, function(d) { return d.input; }))
                        .range([height - 1, 0]),
                    drawTimeout = null,
                    chart = layeredTimeSeries()
                        .colorScale(d3.scale.ordinal()
                                    .domain([0])
                                    .range(['black']))
                        .x(x)
                        .y(y),
                    chartNodeInner = chartNode.append('g')
                        .style('clip-path', 'url(#clip0)');

                chartNode.append('rect')
                    .attr('fill-opacity', 0)
                    .attr('x', 0)
                    .attr('y', 0)
                    .attr('width', chartWidth)
                    .attr('height', height)
                    .call(zoom);

                function draw () {
                    // For this chart, don't sample. It obscures the data.
                    var layers = [
                        timesteps.map(function(step, i) {
                        return {x0: i, x1: i + 1,
                                y0: step['input'] - 0.5,
                                y: 1};
                        })];

                    chartNodeInner.datum(layers)
                        .call(chart);
                }

                onxscalechanged.push(function() {
                    chartNodeInner.call(chart.stretch);
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
    // COLUMN STATES PLOT
    //
    (function() {
        var margin = {top: 0, right: 300, bottom: 4, left: 0},
            height = 100 - margin.top - margin.bottom;

        var svg = charts.append('svg')
                .attr('width', chartWidth + margin.left + margin.right)
                .attr('height', height + margin.top + margin.bottom);

        var defs = svg.append('defs');
        defs.append('clipPath')
            .attr('id', 'clip1')
            .append('rect')
            .attr('width', chartWidth)
            .attr('height', height);

        d3.csv(columnStatesUrl)
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
                        .domain(d3.range(stackOrder.length))
                        .range(stackOrder.map(function(d) { return d.color; })),
                    help = chartNode.append('g')
                        .attr('transform', 'translate(' + (chartWidth+50) + ',10)');

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
                    chart = layeredTimeSeries()
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
            height = 100 - margin.top - margin.bottom;

        var svg = charts.append('svg')
                .attr('width', chartWidth + margin.left + margin.right)
                .attr('height', height + margin.top + margin.bottom);

        var defs = svg.append('defs');
        defs.append('clipPath')
            .attr('id', 'clip2')
            .append('rect')
            .attr('width', chartWidth)
            .attr('height', height);

        d3.text(segmentsUrl, 'text/csv', function(error, contents) {
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
                    .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')')
                    .attr('class', 'crispLayers'),
                stackOrder = d3.range(maxConnectedSynapsesInSegment + 1).reverse(),
                colorExtent = ['whitesmoke', 'black'],
                connectedSynapsesColorScale = d3.scale.linear()
                    .domain([0, maxConnectedSynapsesInSegment])
                    .range(colorExtent),
                stackColorScale = d3.scale.linear()
                    .domain([maxConnectedSynapsesInSegment, 0])
                    .range(colorExtent),
                help = chartNode.append('g')
                    .attr('transform', 'translate(' + (chartWidth+50) + ',0)');

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
                chart = layeredTimeSeries()
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
    // SEGMENT LEARNING PLOTS
    //
    (function() {
        var margin = {top: 5, right: 300, bottom: 5, left: 0},
            height = 80 - margin.top - margin.bottom,
            specs = [{addKey: 'n-added-segments',
                      removeKey: 'n-destroyed-segments',
                      title: 'distal segment',
                      title2: 'addition / removal'},
                     {addKey: 'n-newly-connected-synapses',
                      removeKey: 'n-newly-disconnected-synapses',
                      title: 'distal connected synapse',
                      title2: 'addition / removal'},
                     {addKey: 'n-added-synapses',
                      removeKey: 'n-destroyed-synapses',
                      title: 'distal potential synapse',
                      title2: 'addition / removal'},
                     {addKey: 'n-strengthened-synapses',
                      removeKey: 'n-weakened-synapses',
                      title: 'distal synapse permanence',
                      title2: 'strengthening / weakening'},
                    ];

        specs.forEach(function (spec) {
            spec.svg = charts.append('svg')
                .attr('width', chartWidth + margin.left + margin.right)
                .attr('height', height + margin.top + margin.bottom);

            spec.svg
                .append('defs')
                .append('clipPath')
                .attr('id', 'clip3')
                .append('rect')
                .attr('width', chartWidth)
                .attr('height', height);
        });

        d3.csv(segmentLearningUrl)
            .row(function(d) {
                var intKeys = ['n-added-segments',
                               'n-destroyed-segments',
                               'n-added-synapses',
                               'n-destroyed-synapses',
                               'n-strengthened-synapses',
                               'n-weakened-synapses',
                               'n-newly-connected-synapses',
                               'n-newly-disconnected-synapses'];
                intKeys.forEach(function(k) {
                    d[k] = parseInt(d[k]);
                });
                return d;
            })
            .get(function(error, timesteps) {
                onTimestepCountKnown(timesteps.length);
                specs.forEach(function(spec) {
                    var maxAdd = Math.max(10, d3.max(timesteps, function(d) {
                        return d[spec.addKey];
                    })),
                        maxDestroy = Math.max(10, d3.max(timesteps, function(d) {
                            return d[spec.removeKey];
                        })),
                        chartNode = spec.svg.append('g')
                            .attr('class', 'chart')
                            .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')'),
                        stackColorScale = d3.scale.ordinal()
                            .domain([0, 1])
                            .range(['green', 'lightgreen']),
                        help = chartNode.append('g')
                            .attr('transform', 'translate(' + (chartWidth+50) + ',5)');

                    // Title
                    help.append('text')
                        .attr('class', 'noselect')
                        .attr('text-anchor', 'right')
                        .attr('x', 5)
                        .attr('y', 0)
                        .text(spec.title);

                    help.append('text')
                        .attr('class', 'noselect')
                        .attr('text-anchor', 'right')
                        .attr('x', 5)
                        .attr('y', '1.5em')
                        .text(spec.title2);

                    // Chart
                    var y = d3.scale.linear()
                            .domain([-maxDestroy, maxAdd])
                            .range([height, 0]),
                        drawTimeout = null,
                        chart = layeredTimeSeries()
                            .colorScale(stackColorScale)
                            .x(x)
                            .y(y),
                        chartNodeInner = chartNode.append('g')
                            .style('clip-path', 'url(#clip3)');

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

                        var layers = [
                            xSamples.map(function(data) {
                                return {
                                    x0: data.x0,
                                    x: data.x,
                                    x1: data.x1,
                                    y0: 0,
                                    y: timesteps[data.x][spec.addKey]
                                };
                            }),
                            xSamples.map(function(data) {
                                return {
                                    x0: data.x0,
                                    x: data.x,
                                    x1: data.x1,
                                    y0: 0,
                                    y: -timesteps[data.x][spec.removeKey]
                                };
                            }),
                        ];

                        // d3.layout.stack()(layers); // inserts y0 values
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
                        .attr('class', 'y axis showAxis')
                        .call(d3.svg.axis()
                              .scale(y)
                              .ticks(4)
                              .tickPadding(2)
                              .tickSize(4)
                              .outerTickSize(0)
                              .tickFormat(d3.format('d'))
                              .orient('right'));

                    // Initial change
                    draw();
                    onxscalechanged.forEach(function (f) { f(); });
                });

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
                .style('left', -marginLeft + 'px'),
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
};

</script>

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
</style>

_This post is about HTM. It's written mostly in pictures. The underlying code is discussed in
[my previous posts](/blocks/2016/02/14/htm-time-series-synapse-learning.html)._

[Seeing your HTM run](https://github.com/nupic-community/sanity) is an art. You
have to jump around and look at it from multiple angles.

These graphics are zoomable, just use your mouse wheel or gestures.

## A totally predictable sequence

Here the input data is a repeating sequence
`[0, 4, 8, ..., 92, 96, 0, 4, 8, ...]`. This intentionally mimics the hotgym data set, using a period near 24.

<div style="position:absolute; left:0; right:0;">
<div id="experiment2" style="width:900px;margin:auto;position:relative;"></div>
</div>
<div style="height:710px;"></div>

<script>
insertCharts(document.getElementById('experiment2'),
'/stuff/fixed_seq.inputs.20160215-13.39.11.csv',
'/stuff/fixed_seq.column_states.20160215-13.39.11.csv',
'/stuff/fixed_seq.segments.20160215-13.39.11.csv',
'/stuff/fixed_seq.segment_learning.20160215-13.39.11.csv');
</script>

There's always some bursting, and it really starts freaking out around step
3000!

## A sine wave: mostly predictable

The input data is a sine wave that oscillates between 0 and 100, using a period
of 8π. It, too, intentionally mimics the hotgym data set, using a period near 24.

<div style="position:absolute; left:0; right:0;">
<div id="experiment3" style="width:900px;margin:auto;position:relative;"></div>
</div>
<div style="height:710px;"></div>

<script>
insertCharts(document.getElementById('experiment3'),
  '/stuff/sine.inputs.20160215-13.45.02.csv',
  '/stuff/sine.column_states.20160215-13.45.02.csv',
  '/stuff/sine.segments.20160215-13.45.02.csv',
  '/stuff/sine.segment_learning.20160215-13.45.02.csv');
</script>

Similar to before, the HTM has a sort of mid-life crisis after a few thousand
timesteps.

## Actual data: kinda predictable

This is Numenta's "hotgym" data set.

<div style="position:absolute; left:0; right:0;">
<div id="experiment1" style="width:900px;margin:auto;position:relative;"></div>
</div>
<div style="height:710px;"></div>

<script>
insertCharts(document.getElementById('experiment1'),
'/stuff/hotgym.inputs.20160215-15.20.28.csv',
'/stuff/hotgym.column_states.20160215-15.20.28.csv',
'/stuff/hotgym.segments.20160215-15.20.28.csv',
'/stuff/hotgym.segment_learning.20160215-15.20.28.csv');
</script>

It's interesting seeing how many segments have zero connected synapses at any
given time. Now I want to sort them by segment ID and see whether these empty
segments are recently created, or if some of them are old segments.

Also, notice that there's a little burst of predictions toward the
beginning. The burst ends at step 51. At this point, synapse permanences (and,
hence, the number of connected synapses) are held constant for almost 50
timesteps. Meanwhile, potential synapses are added again and again.

## Not predictable: random integers

Now let's look at the other extreme: random integers between 0 and 100.

<div style="position:absolute; left:0; right:0;">
<div id="experiment4" style="width:900px;margin:auto;position:relative;"></div>
</div>
<div style="height:710px;"></div>

<script>
insertCharts(document.getElementById('experiment4'),
'/stuff/random.inputs.20160215-13.55.21.csv',
'/stuff/random.column_states.20160215-13.55.21.csv',
'/stuff/random.segments.20160215-13.55.21.csv',
'/stuff/random.segment_learning.20160215-13.55.21.csv');
</script>

A really big segment count. Sure enough, looking across these data streams, the
number of segments increased as the predictability of the stream decreased.

## What am I looking at?

Oh, sorry, I should have explained.

These plots show:

- The input data
- The number of active and predicted columns
  - I used [Felix](http://floybix.github.io/)'s design which appears in [Sanity](https://github.com/nupic-community/sanity).
- The total distribution of dendrite segments, colored by their number of
  connected synapses
- The delta of dendrite segments
- The delta of connected synapses
  - i.e. when a potential synapse's permanence crosses the connected threshold
- The delta of potential synapses
- The number of synapses whose permanence was strengthened or weakened

The left-right dimension is time.

These plots focus on distal synapses, i.e. on the Temporal Memory.

I generated this data using [nupic](https://github.com/numenta/nupic). Here's my
[ipython notebook](/stuff/4 experiments.ipynb), feel free to tweak it.

## Next steps ##

- Combine this with Sanity. When you zoom in on individual timesteps, it'd be
  nice to be able to inspect the HTM at those timesteps.
- Using Sanity, show why the HTM receiving predictable data suddenly starts
  freaking out around timestep 3000.
- Allow different sort orders on the distal segments chart. I want to sort by
  segment ID to see whether that pool of segments-with-no-connected-synapses is
  filled with recently created segments, or if lots of them are old
  segments. Although the file size of this trace will be huge, because it means
  one column per segment per timestep.

_Thanks to Felix Andrews for providing feedback and insights._
