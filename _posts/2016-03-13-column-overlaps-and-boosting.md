---
layout: post
title: "HTM time series: Column overlaps and boosting"
date:   2016-03-13 10:00:00
---

<script src="//d3js.org/d3.v3.min.js" charset="utf-8"></script>
<script>
function insertCharts(node, inputDataUrl, columnStatesUrl, segmentsUrl, segmentLearningUrl,
                      columnOverlapsUrl, columnBoostedOverlapsUrl, allBoostFactorsUrl,
                      activeDueToBoostingUrl, timeSinceActiveUrl) {
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
        xSamples = [],
        overlapColorScale = d3.scale.linear()
            .domain([Number.MAX_VALUE, -1])
            .range(['whitesmoke', 'black']),
        onOverlapColorScaleChanged = []; // callbacks

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
            y;

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
                    .attr('stroke', 'none');
                layer
                    .attr('fill', function(d, i) { return colorScale(d.key); });
                layer.exit()
                    .remove();

                layer.attr('d', function (data) {
                    var ds = data.values;
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

    function boxPlotTimeSeries() {
        var x,
            y;

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

                var column = stretchMe.selectAll('.column')
                        .data(function (d) { return d; });

                var entering = column.enter()
                        .append('g')
                        .attr('class', 'column');

                column
                    .attr('transform', function(data, i) {
                        return 'translate(' + data.x0 + ',0)';
                    });

                entering.append('rect')
                    .attr('class', 'top')
                    .attr('x', 0);
                entering.append('rect')
                    .attr('class', 'bottom')
                    .attr('x', 0);
                entering.append('ellipse')
                    .attr('class', 'median');

                column.exit()
                    .remove();

                column.select('rect.top')
                    .attr('y', function(data, i) { return y(data['max']); })
                    .attr('height', function(data, i) {
                        return Math.max(1, y(data.q3) - y(data.max));
                    })
                    .attr('width', function(data, i) { return data.x1 - data.x0; });

                column.select('rect.bottom')
                    .attr('y', function(data, i) { return y(data['q1']); })
                    .attr('height', function(data, i) {
                        return Math.max(1, y(data.min) - y(data.q1));
                    })
                    .attr('width', function(data, i) { return data.x1 - data.x0; });

                column.select('ellipse.median')
                    .attr('cx', function(data, i) {
                        return (data.x1 - data.x0) / 2;
                    })
                    .attr('cy', function(data, i) { return y(data.median); })
                    .attr('rx', function(data, i) { return (data.x1 - data.x0) / 2; })
                    .attr('ry', function(data, i) { return 1; });
            }).call(stretch);
        };

        chart.stretch = stretch;

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
        if (!inputDataUrl) {
            return;
        }
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
                    var layers = [{
                        key: 0,
                        values: timesteps.map(function(step, i) {
                            return {x0: i, x1: i + 1,
                                    y0: step['input'] - 0.5,
                                    y: 1};
                        })}];

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
    })
    ();

    //
    // COLUMN STATES PLOT
    //
    (function() {
        if (!columnStatesUrl) {
            return;
        }
        var margin = {top: 0, right: 300, bottom: 4, left: 0},
            height = 60 - margin.top - margin.bottom;

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
                        .domain(stackOrder.map(function(d) { return d.key; }))
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
                        var values = xSamples.map(function(data) {
                            var y = timesteps[data.x][o.key] || 0;
                            return {x: data.x, x0: data.x0, x1: data.x1, y: y};
                        });
                        layers.push({
                            key: o.key,
                            values: values
                        });
                    });
                    var stack = d3.layout.stack()
                            .values(function(d) { return d.values; });
                    stack(layers); // inserts y0 values
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
    // COLUMN OVERLAP PLOT
    //
    (function() {
        if (!columnOverlapsUrl) {
            return;
        }

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

        d3.text(columnOverlapsUrl, 'text/csv', function(error, contents) {
            var timesteps = d3.csv.parseRows(contents),
                maxColumns = 0,
                minOverlap = Number.MAX_VALUE,
                maxOverlap = 0;

            onTimestepCountKnown(timesteps.length);

            timesteps.forEach(function(row) {
                maxColumns = Math.max(maxColumns, row.length);
                if (row.length > 0) {
                    minOverlap = Math.min(minOverlap, row[row.length - 1]);
                    maxOverlap = Math.max(maxOverlap, row[0]);
                }

                for (var i = 0; i < row.length; i++) {
                    row[i] = parseInt(row[i]);
                }
            });

            var overlapExtent = overlapColorScale.domain();
            overlapExtent[0] = Math.min(minOverlap, overlapExtent[0]);
            overlapExtent[1] = Math.max(maxOverlap, overlapExtent[1]);
            overlapColorScale.domain(overlapExtent);
            onOverlapColorScaleChanged.forEach(function (f) { f(); });

            var chartNode = svg.append('g')
                    .attr('class', 'chart')
                    .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')')
                    .attr('class', 'crispLayers'),
                help = chartNode.append('g')
                    .attr('transform', 'translate(' + (chartWidth+50) + ',0)');

            // Title and legend
            help.append('text')
                .attr('class', 'noselect')
                .attr('text-anchor', 'right')
                .attr('x', 2)
                .attr('y', 0)
                .text('active columns, stacked by:');

            var legendWidth = 200,
                legend = help.append('g')
                    .attr('class', 'noselect')
                    .attr('transform', 'translate(0, 20)'),
                minText = legend.append('text')
                    .attr('y', 14)
                    .attr('text-anchor', 'middle'),
                maxText = legend.append('text')
                    .attr('x', legendWidth)
                    .attr('text-anchor', 'middle')
                    .attr('y', 14);

            legend.append('text')
                .attr('x', legendWidth / 2)
                .attr('y', -4)
                .attr('text-anchor', 'middle')
                .text('overlap with the input');;

            function drawLegend() {
                var overlapExtent = overlapColorScale.domain();
                var rect = legend.selectAll('rect')
                    .data(d3.range(overlapExtent[0], overlapExtent[1],
                                   (overlapExtent[1] - overlapExtent[0]) / legendWidth));
                rect.enter()
                    .append('rect')
                    .attr('x', function(d, i) { return i; })
                    .attr('width', 1)
                    .attr('height', 4);
                rect.attr('fill', function(d, i) { return overlapColorScale(d); });

                minText.text(overlapExtent[0]);
                maxText.text(overlapExtent[1]);
            }

            drawLegend();

            // Chart
            var y = d3.scale.linear()
                    .domain([0, maxColumns])
                    .range([height, 0]),
                drawTimeout = null,
                chart = layeredTimeSeries()
                    .colorScale(overlapColorScale)
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

                // Format:
                // [{key: 22, values: [{x: 0, y: 0}, {x: 1, y: 2}, ...]},
                //  {key: 25, values: [{x: 0, y: 2}, {x: 1, y: 0}, ...]},
                //  ... ]
                var layers = [];

                var reverse = true;
                xSamples.forEach(function(data, sampleIndex) {
                    var overlaps = timesteps[data.x];
                    overlaps.forEach(function(overlap) {
                        var layer;
                        for (var i = 0; i < layers.length + 1; i++) {
                            if (i >= layers.length ||
                                (reverse && layers[i].key < overlap) ||
                                (!reverse && layers[i].key > overlap)) {
                                layer = {
                                    key: overlap,
                                    values: xSamples.map(function(data) {
                                        return {
                                            x0: data.x0,
                                            x: data.x,
                                            x1: data.x1,
                                            y: 0
                                        };
                                    })
                                };
                                layers.splice(i, 0, layer);
                                break;
                            }
                            else if (layers[i].key == overlap) {
                                layer = layers[i];
                                break;
                            }
                        }

                        layer.values[sampleIndex].y++;
                    });
                });

                var stack = d3.layout.stack()
                        .values(function (d) { return d.values; });
                stack(layers); // inserts y0 values
                chartNodeInner.datum(layers)
                    .call(chart);
            }

            onxscalechanged.push(function(maybeCoalesce) {
                chartNodeInner.call(chart.stretch);
                if (!drawTimeout) {
                    drawTimeout = setTimeout(draw, maybeCoalesce ? 250 : 1000/60);
                }
            });

            onOverlapColorScaleChanged.push(function() {
                drawLegend();
                draw();
            });

            // Initial change
            draw();
            drawLegend();
            onxscalechanged.forEach(function (f) { f(); });
        });
    })
    ();

    //
    // BOOSTED COLUMN OVERLAP PLOT
    //
    (function() {
        if (!columnBoostedOverlapsUrl) {
            return;
        }

        var margin = {top: 20, right: 300, bottom: 20, left: 0},
            height = 100 - margin.top - margin.bottom;

        var svg = charts.append('svg')
                .attr('width', chartWidth + margin.left + margin.right)
                .attr('height', height + margin.top + margin.bottom);

        var defs = svg.append('defs');
        defs.append('clipPath')
            .attr('id', 'clip3')
            .append('rect')
            .attr('width', chartWidth)
            .attr('height', height);

        d3.text(columnBoostedOverlapsUrl, 'text/csv', function(error, contents) {
            var timesteps = d3.csv.parseRows(contents),
                maxColumns = 0,
                minOverlap = Number.MAX_VALUE,
                maxOverlap = 0;

            onTimestepCountKnown(timesteps.length);

            timesteps.forEach(function(row) {
                maxColumns = Math.max(maxColumns, row.length);
                if (row.length > 0) {
                    minOverlap = Math.min(minOverlap, row[row.length - 1]);
                    maxOverlap = Math.max(maxOverlap, row[0]);
                }

                for (var i = 0; i < row.length; i++) {
                    // Much faster, because there are lots more potential floats
                    // than potential ints, and we need one layer per unique
                    // value.  And it's actually easier to read with the bigger
                    // color changes, despite being less precise.
                    row[i] = Math.round(parseFloat(row[i]));
                }
            });

            var overlapExtent = overlapColorScale.domain();
            overlapExtent[0] = Math.min(minOverlap, overlapExtent[0]);
            overlapExtent[1] = Math.max(maxOverlap, overlapExtent[1]);
            overlapColorScale.domain(overlapExtent);
            onOverlapColorScaleChanged.forEach(function (f) { f(); });

            var chartNode = svg.append('g')
                    .attr('class', 'chart')
                    .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')')
                    .attr('class', 'crispLayers'),
                help = chartNode.append('g')
                    .attr('transform', 'translate(' + (chartWidth+50) + ',0)');

            // Title and legend
            help.append('text')
                .attr('class', 'noselect')
                .attr('text-anchor', 'right')
                .attr('x', 2)
                .attr('y', 0)
                .text('active columns, stacked by:');

            var legendWidth = 200,
                legend = help.append('g')
                    .attr('class', 'noselect')
                    .attr('transform', 'translate(0, 20)'),
                minText = legend.append('text')
                    .attr('y', 14)
                    .attr('text-anchor', 'middle'),
                maxText = legend.append('text')
                    .attr('x', legendWidth)
                    .attr('text-anchor', 'middle')
                    .attr('y', 14);

            legend.append('text')
                .attr('x', legendWidth / 2)
                .attr('y', -4)
                .attr('text-anchor', 'middle')
                .text('overlap with the input, boosted');;

            function drawLegend() {
                var overlapExtent = overlapColorScale.domain();
                var rect = legend.selectAll('rect')
                    .data(d3.range(overlapExtent[0], overlapExtent[1],
                                   (overlapExtent[1] - overlapExtent[0]) / legendWidth));
                rect.enter()
                    .append('rect')
                    .attr('x', function(d, i) { return i; })
                    .attr('width', 1)
                    .attr('height', 4);
                rect.attr('fill', function(d, i) { return overlapColorScale(d); });

                minText.text(overlapExtent[0]);
                maxText.text(overlapExtent[1]);
            }

            // Chart
            var y = d3.scale.linear()
                    .domain([0, maxColumns])
                    .range([height, 0]),
                drawTimeout = null,
                chart = layeredTimeSeries()
                    .colorScale(overlapColorScale)
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

                // Format:
                // [{key: 22, values: [{x: 0, y: 0}, {x: 1, y: 2}, ...]},
                //  {key: 25, values: [{x: 0, y: 2}, {x: 1, y: 0}, ...]},
                //  ... ]
                var layers = [];

                var reverse = true;
                xSamples.forEach(function(data, sampleIndex) {
                    var overlaps = timesteps[data.x];
                    overlaps.forEach(function(overlap) {
                        var layer;
                        for (var i = 0; i < layers.length + 1; i++) {
                            if (i >= layers.length ||
                                (reverse && layers[i].key < overlap) ||
                                (!reverse && layers[i].key > overlap)) {
                                layer = {
                                    key: overlap,
                                    values: xSamples.map(function(data) {
                                        return {
                                            x0: data.x0,
                                            x: data.x,
                                            x1: data.x1,
                                            y: 0
                                        };
                                    })
                                };
                                layers.splice(i, 0, layer);
                                break;
                            }
                            else if (layers[i].key == overlap) {
                                layer = layers[i];
                                break;
                            }
                        }

                        layer.values[sampleIndex].y++;
                    });
                });
                var stack = d3.layout.stack()
                        .values(function (d) { return d.values; });
                stack(layers); // inserts y0 values
                chartNodeInner.datum(layers)
                    .call(chart);
            }

            onxscalechanged.push(function(maybeCoalesce) {
                chartNodeInner.call(chart.stretch);
                if (!drawTimeout) {
                    drawTimeout = setTimeout(draw, maybeCoalesce ? 250 : 1000/60);
                }
            });

            onOverlapColorScaleChanged.push(function() {
                drawLegend();
                draw();
            });

            // Initial change
            draw();
            drawLegend();
            onxscalechanged.forEach(function (f) { f(); });
        });
    })
    ();

    //
    // ALL BOOST FACTORS PLOT
    //
    (function() {
        if (!allBoostFactorsUrl) {
            return;
        }

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

        d3.text(allBoostFactorsUrl, 'text/csv', function(error, contents) {
            var timesteps = d3.csv.parseRows(contents),
                maxBoosted = 0,
                minFactor = Number.MAX_VALUE,
                maxFactor = 0,
                nColors = 20; // perf

            onTimestepCountKnown(timesteps.length);

            var boostCounts = timesteps.map(function(row) {
                var nBoosted = 0,
                    freqs = [];
                for (var i = 0; i < row.length; i += 2) {
                    var boostFactor = Math.round(parseFloat(row[i]) * nColors) / nColors,
                        count = parseInt(row[i+1]);

                    if (boostFactor <= 1) {
                        continue;
                    }
                    else {
                        nBoosted += count;
                        minFactor = Math.min(minFactor, boostFactor);
                        maxFactor = Math.max(maxFactor, boostFactor);
                        freqs.push([boostFactor, count]);
                    }
                }

                maxBoosted = Math.max(maxBoosted, nBoosted);

                return freqs;
            });

            var chartNode = svg.append('g')
                    .attr('class', 'chart')
                    .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')')
                    .attr('class', 'crispLayers'),
                colorScale = d3.scale.linear()
                    .domain([minFactor, maxFactor])
                    .range(['whitesmoke', 'black']),
                help = chartNode.append('g')
                    .attr('transform', 'translate(' + (chartWidth+50) + ',0)');

            // Title and legend
            help.append('text')
                .attr('class', 'noselect')
                .attr('text-anchor', 'right')
                .attr('x', 2)
                .attr('y', 0)
                .text('all boosted columns (including inactive)');

            var legend = help.append('g')
                    .attr('class', 'noselect')
                    .attr('transform', 'translate(0, 20)'),
                legendWidth = 200,
                rect = legend.selectAll('rect')
                    .data(d3.range(minFactor, maxFactor, (maxFactor - minFactor) / legendWidth));
            rect.enter()
                .append('rect')
                .attr('x', function(d, i) { return i; })
                .attr('width', 1)
                .attr('height', 4);
            rect.attr('fill', function(d, i) { return colorScale(d); });
            legend.append('text')
                .attr('x', legendWidth / 2)
                .attr('y', -4)
                .attr('text-anchor', 'middle')
                .text('boost factor');

            legend.append('text')
                .attr('y', 14)
                .attr('text-anchor', 'start')
                .text(minFactor.toFixed(4));
            legend.append('text')
                .attr('x', legendWidth)
                .attr('text-anchor', 'end')
                .attr('y', 14)
                .text(maxFactor.toFixed(4));

            // Chart
            var y = d3.scale.pow().exponent(6/10)
                    .domain([0, maxBoosted])
                    .range([height, 0]),
                drawTimeout = null,
                chart = layeredTimeSeries()
                    .colorScale(colorScale)
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

                // Format:
                // [{key: 22, values: [{x: 0, y: 0}, {x: 1, y: 2}, ...]},
                //  {key: 25, values: [{x: 0, y: 2}, {x: 1, y: 0}, ...]},
                //  ... ]
                var layers = [];

                var reverse = false;
                xSamples.forEach(function(data, sampleIndex) {
                    var freqs = boostCounts[data.x];
                    freqs.forEach(function(kv) {
                        var boostFactor = kv[0],
                            count = kv[1],
                            layer;
                        for (var i = 0; i < layers.length + 1; i++) {
                            if (i >= layers.length ||
                                (reverse && layers[i].key < boostFactor) ||
                                (!reverse && layers[i].key > boostFactor)) {
                                layer = {
                                    key: boostFactor,
                                    values: xSamples.map(function(data) {
                                        return {
                                            x0: data.x0,
                                            x: data.x,
                                            x1: data.x1,
                                            y: 0
                                        };
                                    })
                                };
                                layers.splice(i, 0, layer);
                                break;
                            }
                            else if (layers[i].key == boostFactor) {
                                layer = layers[i];
                                break;
                            }
                        }

                        layer.values[sampleIndex].y = count;
                    });
                });
                var stack = d3.layout.stack()
                        .values(function (d) { return d.values; });
                stack(layers); // inserts y0 values
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
                      .ticks(3)
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
    // ACTIVE DUE TO BOOSTING PLOT
    //
    (function() {
        if (!activeDueToBoostingUrl) {
            return;
        }

        var margin = {top: 5, right: 300, bottom: 4, left: 0},
            height = 100 - margin.top - margin.bottom;

        var svg = charts.append('svg')
                .attr('width', chartWidth + margin.left + margin.right)
                .attr('height', height + margin.top + margin.bottom);

        var defs = svg.append('defs');
        defs.append('clipPath')
            .attr('id', 'clip3')
            .append('rect')
            .attr('width', chartWidth)
            .attr('height', height);

        d3.csv(activeDueToBoostingUrl)
            .row(function(d) {
                var intKeys = ['n-active-due-to-boosting',];
                intKeys.forEach(function(k) {
                    d[k] = parseInt(d[k]);
                });
                return d;
            })
            .get(function(error, timesteps) {
                onTimestepCountKnown(timesteps.length);
                var stackOrder = [{key: 'n-active-due-to-boosting',
                                   color: 'maroon'},],
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
                        .attr('transform', 'translate(' + (chartWidth+50) + ',10)');

                // Title and legend
                help.append('text')
                    .attr('class', 'noselect')
                    .attr('text-anchor', 'right')
                    .attr('x', 5)
                    .attr('y', 0)
                    .text('columns active due to boosting');;

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

                    var layers = [];
                    stackOrder.forEach(function(o) {
                        var values = xSamples.map(function(data) {
                            var y = timesteps[data.x][o.key] || 0;
                            return {x: data.x, x0: data.x0, x1: data.x1, y: y};
                        });
                        layers.push({
                            key: o.key,
                            values: values
                        });
                    });
                    var stack = d3.layout.stack()
                            .values(function(d) { return d.values; });
                    stack(layers); // inserts y0 values
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
                          .ticks(2)
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
        if (!segmentsUrl) {
            return;
        }

        var margin = {top: 20, right: 300, bottom: 20, left: 0},
            height = 100 - margin.top - margin.bottom;

        var svg = charts.append('svg')
                .attr('width', chartWidth + margin.left + margin.right)
                .attr('height', height + margin.top + margin.bottom);

        var defs = svg.append('defs');
        defs.append('clipPath')
            .attr('id', 'clip4')
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
                colorScale = d3.scale.linear()
                    .domain([0, maxConnectedSynapsesInSegment])
                    .range(['whitesmoke', 'black']),
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
            rect.attr('fill', function(d, i) { return colorScale(d); });
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
                    .colorScale(colorScale)
                    .x(x)
                    .y(y),
                chartNodeInner = chartNode.append('g')
                    .style('clip-path', 'url(#clip4)');

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
                    var values = xSamples.map(function(data) {
                        var y = timesteps[data.x][key] || 0;
                        return {x: data.x, x0: data.x0, x1: data.x1, y: y};
                    });
                    layers.push({
                        key: key,
                        values: values
                    });
                });
                var stack = d3.layout.stack()
                        .values(function (d) { return d.values; });
                stack(layers); // inserts y0 values
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
        if (!segmentLearningUrl) {
            return;
        }

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
                .attr('id', 'clip5')
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
                        colorScale = d3.scale.ordinal()
                            .domain(['add', 'remove'])
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
                            .colorScale(colorScale)
                            .x(x)
                            .y(y),
                        chartNodeInner = chartNode.append('g')
                            .style('clip-path', 'url(#clip5)');

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
                            {key: 'add',
                             values: xSamples.map(function(data) {
                                 return {
                                     x0: data.x0,
                                     x: data.x,
                                     x1: data.x1,
                                     y0: 0,
                                     y: timesteps[data.x][spec.addKey]
                                 };
                             })},
                            {key: 'remove',
                             values: xSamples.map(function(data) {
                                 return {
                                     x0: data.x0,
                                     x: data.x,
                                     x1: data.x1,
                                     y0: 0,
                                     y: -timesteps[data.x][spec.removeKey]
                                 };
                             })},
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
    // TIME SINCE ACTIVE PLOT
    //
    (function() {
        if (!timeSinceActiveUrl) {
            return;
        }

        var margin = {top: 5, right: 300, bottom: 4, left: 0},
            height = 600 - margin.top - margin.bottom;

        var svg = charts.append('svg')
                .attr('width', chartWidth + margin.left + margin.right)
                .attr('height', height + margin.top + margin.bottom);

        var defs = svg.append('defs');
        defs.append('clipPath')
            .attr('id', 'clip6')
            .append('rect')
            .attr('width', chartWidth)
            .attr('height', height);

        d3.csv(timeSinceActiveUrl)
            .row(function(d) {
                var intKeys = ['min',
                               'q1',
                               'median',
                               'q3',
                               'max'];
                intKeys.forEach(function(k) {
                    d[k] = parseFloat(d[k]);
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
                    .text('distribution: timesteps since column last active');;

                // Chart
                var y = d3.scale.linear()
                        .domain([0, d3.max(timesteps, function(d) { return d.max; })])
                        .range([height, 0]),
                    drawTimeout = null,
                    chart = boxPlotTimeSeries()
                        .x(x)
                        .y(y),
                    chartNodeInner = chartNode.append('g')
                        .style('clip-path', 'url(#clip6)');

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

                    var prepared = xSamples.map(function(data) {
                        var step = timesteps[data.x];
                        return {
                            min: step.min,
                            q1: step.q1,
                            median: step.median,
                            q3: step.q3,
                            max: step.max,
                            x0: data.x0,
                            x1: data.x1
                        };
                    });

                    chartNodeInner.datum(prepared)
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
                          .ticks(8)
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

In my [last post](/blocks/2016/02/15/htm-stacks-of-time-series.html), HTMs were
doing mysterious things. The graphic caused you to point and ask:

> "What's happening right there?"

In this post, the graphic answers that question.<br /><br />

## Bring back the trivial sequence

Let's reuse the "totally predictable sequence" from the
[previous post](/blocks/2016/02/15/htm-stacks-of-time-series.html).

Again, the input data is a repeating sequence: **[0, 4, 8, ..., 92, 96, 0, 4, 8, ...]**

<div style="position:absolute; left:0; right:0;">
<div id="focusOnBoosting" style="width:900px;margin:auto;position:relative;"></div>
</div>
<div style="height:650px;"></div>

<script>
insertCharts(document.getElementById('focusOnBoosting'),
"/stuff/fixed_seq.inputs.20160313-10.31.40.csv",
"/stuff/fixed_seq.column_states.20160313-10.31.40.csv",
null,
null,
"/stuff/fixed_seq.column_overlaps.20160313-10.31.40.csv",
"/stuff/fixed_seq.column_boosted_overlaps.20160313-10.31.40.csv",
"/stuff/fixed_seq.boosted_columns.20160313-10.31.40.csv",
"/stuff/fixed_seq.active_due_to_boosting.20160313-10.31.40.csv",
null
      );
</script>

We're now showing:

- Column overlaps with the input, before and after boosting.
  - If a column has synapses connected to 14 active bits, its overlap is 14.
  - If this column has a boost factor of 1.5, it's final overlap is 21.
- Boost factors.
- How many columns are active because of boosting.
  - Equivalently, how many columns were suppressed because of boosting.
  - To compute this, I pass the pre-boosted overlaps directly into the C++
    spatial pooler's `inhibitColumns_` method, then diffed the results.

You can now see that lots of the big unexpected events in the timeline were
caused by boosting.

A few things to note:

- An enormous round of boosting happens at step 51. Every single active column
  is active because of boosting. This means the 40 columns with the highest
  overlaps are all being inhibited because of boosting.
- The "columns active due to boosting" chart almost fits like a puzzle piece
  into the pre-boosting overlaps chart. Just turn it upside down.
- Independent of all of this, it's illuminating to see the overlaps. I was
  excited to build this just so I could see what it'd look like.

<br />

## The next question (an ugly preview)

Now that I know what's going on, I'm wondering: **is boosting working?**

More precisely, is boosting causing the HTM to make use of all of its columns?
Here's an early glance at whether this is happening.

It isn't pretty, but here's a rough time series of
[Tufte-style box plots](https://samcliffordinfo.files.wordpress.com/2012/12/boxplots.png).

<div style="position:absolute; left:0; right:0;">
<div id="uglyPreview" style="width:900px;margin:auto;position:relative;"></div>
</div>
<div style="height:1050px;"></div>

<script>
insertCharts(document.getElementById('uglyPreview'),
"/stuff/fixed_seq.inputs.20160313-10.31.40.csv",
"/stuff/fixed_seq.column_states.20160313-10.31.40.csv",
null,
null,
null,
null,
"/stuff/fixed_seq.boosted_columns.20160313-10.31.40.csv",
"/stuff/fixed_seq.active_due_to_boosting.20160313-10.31.40.csv",
"/stuff/fixed_seq.time_since_column_active.20160313-10.31.40.csv"
      );
</script>

<br />

The distribution of columns' "time since last activation" is shown by its
quartiles:

- The minimum is the bottom of the lower black area.
- The 25th percentile is the top of the lower black area.
- The median is the dot in the center.
- The 75th percentile is the bottom of the upper black area.
- The maximum is the top of the upper black area.

This graphic isn't polished, but it tells a story of neglected columns. Until
step 3244, the median column has not been active in the past couple thousand
timesteps. Then, boosting causes the median and 75th percentile to drop, meaning
a lot of the neglected columns got activated.

With this input sequence, it makes a lot of sense that most columns don't get
used by default. With only 25 unique input patterns and 40 columns per pattern,
we will not use all 2048 columns.

In this graphic, the data is too aggregated. Looking at it, we have no idea how
the upper quartile is distributed. Toward the end, the enormous black section
only tells us that at least one column is still not getting activated, while the
75th percentile was activated much more recently. Though the 75th percentile is
still pretty high!


## Next steps

- Find a prettier way of showing whether boosting is working.
- Add [Sanity](https://github.com/nupic-community/sanity) to these charts. When
  you zoom in on individual timesteps, it’d be nice to be able to inspect the
  HTM at those timesteps.
- Add these charts to Sanity. Generate these charts in real time as an HTM runs.

<br />

## Appendix: Stack all the things

Here are all four data sets from the
[previous post](/blocks/2016/02/15/htm-stacks-of-time-series.html), showing
every chart. Even the ugly one.

### A totally predictable sequence

Repeating sequence: **[0, 4, 8, ..., 92, 96, 0, 4, 8, ...]**

<div style="position:absolute; left:0; right:0;">
<div id="appendix0" style="width:900px;margin:auto;position:relative;"></div>
</div>
<div style="height:1750px;"></div>

<script>
insertCharts(document.getElementById('appendix0'),
"/stuff/fixed_seq.inputs.20160313-10.31.40.csv",
"/stuff/fixed_seq.column_states.20160313-10.31.40.csv",
"/stuff/fixed_seq.segments.20160313-10.31.40.csv",
"/stuff/fixed_seq.segment_learning.20160313-10.31.40.csv",
"/stuff/fixed_seq.column_overlaps.20160313-10.31.40.csv",
"/stuff/fixed_seq.column_boosted_overlaps.20160313-10.31.40.csv",
"/stuff/fixed_seq.boosted_columns.20160313-10.31.40.csv",
"/stuff/fixed_seq.active_due_to_boosting.20160313-10.31.40.csv",
"/stuff/fixed_seq.time_since_column_active.20160313-10.31.40.csv"
      );
</script>

### A sine wave: mostly predictable

A sine wave that oscillates between 0 and 100, using a period of 8π.

<div style="position:absolute; left:0; right:0;">
<div id="appendix1" style="width:900px;margin:auto;position:relative;"></div>
</div>
<div style="height:1750px;"></div>

<script>
insertCharts(document.getElementById('appendix1'),
"/stuff/sine.inputs.20160313-10.36.33.csv",
"/stuff/sine.column_states.20160313-10.36.33.csv",
"/stuff/sine.segments.20160313-10.36.33.csv",
"/stuff/sine.segment_learning.20160313-10.36.33.csv",
"/stuff/sine.column_overlaps.20160313-10.36.33.csv",
"/stuff/sine.column_boosted_overlaps.20160313-10.36.33.csv",
"/stuff/sine.boosted_columns.20160313-10.36.33.csv",
"/stuff/sine.active_due_to_boosting.20160313-10.36.33.csv",
"/stuff/sine.time_since_column_active.20160313-10.36.33.csv"
      );
</script>

### Actual data: kinda predictable

This is Numenta’s “hotgym” data set.

<div style="position:absolute; left:0; right:0;">
<div id="appendix2" style="width:900px;margin:auto;position:relative;"></div>
</div>
<div style="height:1750px;"></div>

<script>
insertCharts(document.getElementById('appendix2'),
"/stuff/hotgym.inputs.20160313-10.44.42.csv",
"/stuff/hotgym.column_states.20160313-10.44.42.csv",
"/stuff/hotgym.segments.20160313-10.44.42.csv",
"/stuff/hotgym.segment_learning.20160313-10.44.42.csv",
"/stuff/hotgym.column_overlaps.20160313-10.44.42.csv",
"/stuff/hotgym.column_boosted_overlaps.20160313-10.44.42.csv",
"/stuff/hotgym.boosted_columns.20160313-10.44.42.csv",
"/stuff/hotgym.active_due_to_boosting.20160313-10.44.42.csv",
"/stuff/hotgym.time_since_column_active.20160313-10.44.42.csv"
      );
</script>


### Not predictable: random integers

Random integers between 0 and 100.

<div style="position:absolute; left:0; right:0;">
<div id="appendix3" style="width:900px;margin:auto;position:relative;"></div>
</div>
<div style="height:1750px;"></div>

<script>
insertCharts(document.getElementById('appendix3'),
"/stuff/random.inputs.20160313-10.55.13.csv",
"/stuff/random.column_states.20160313-10.55.13.csv",
"/stuff/random.segments.20160313-10.55.13.csv",
"/stuff/random.segment_learning.20160313-10.55.13.csv",
"/stuff/random.column_overlaps.20160313-10.55.13.csv",
"/stuff/random.column_boosted_overlaps.20160313-10.55.13.csv",
"/stuff/random.boosted_columns.20160313-10.55.13.csv",
"/stuff/random.active_due_to_boosting.20160313-10.55.13.csv",
"/stuff/random.time_since_column_active.20160313-10.55.13.csv"
      );
</script>

## Appendix: How I got this data

I generated this data using [nupic](https://github.com/numenta/nupic). I used a
custom build of nupic.core that adds two methods to the `SpatialPooler` class
in `SpatialPooler.hpp`:

{% highlight c++ %}

  vector<UInt> getOverlaps()
  {
    return overlaps_;
  }

  vector<Real> getBoostedOverlaps()
  {
    return boostedOverlaps_;
  }

{% endhighlight %}

Here's my Jupyter notebook:

- [See it as HTML](/stuff/column-overlaps-and-boosting.html)
- [Download the notebook](/stuff/column-overlaps-and-boosting.ipynb)

My HTM model parameters came from a swarm on the hotgym data set, then I changed
'temporalImp' to 'tm_py' to use `temporal_memory.py`. These parameters could
definitely be improved. I wasn't focusing on tuning parameters, I was focused on
seeing what's going on.
