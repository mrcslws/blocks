---
layout: post
title: "Grid cells: Visualizing the CAN model"
date:   2017-04-16 12:00:00
---

_I simulated the
[Continuous Attractor Network model of grid cells](http://journals.plos.org/ploscompbiol/article?id=10.1371/journal.pcbi.1000291). Here's
an initial look._

Here are 4096 groups of 4 neurons. Below, I hover over a cell to show
its inhibitory output. The red cells are inhibited by the selected
cell.

<video src="/stuff/mexican-hat.mp4" poster="/stuff/mexican-hat.small.png" width="100%" controls loop></video>

<br />

These cells are arranged into groups of 4, with one cell devoted to
each direction. The "preferred direction" of the top cell in each
group is North, the right cell East, the bottom cell South, and the
left cell West. The North cell inhibits a ring of cells slightly
"north" of the cell. The North cell also receives excitatory
feedforward input when the animal moves north (this isn't
depicted). The same is true of the East, South, and West cells.

In the simulation below, I give each neuron a random initial firing
rate (shown in black), then I let some time pass. The velocity input
is 0. Each cell gets an equal amount of feedforward input.

<video src="/stuff/forming-lattice.mp4" poster="/stuff/forming-lattice.small.png" width="100%" controls loop></video>

<br />

It forms a near-perfect hexagonal lattice. The lattice is always
oriented parallel to either the horizontal or vertical axis.

I _kinda_ understand why the grid prefers to align with one of these
axes. It has something to do with the periodic map of cells. The space
here is similar to a game of "Asteroids" -- when your ship goes off
the right edge, it appears on the left side. If you go straight left
or straight up, you'll soon reach your starting point, but this isn't
true of other directions. If the topology were more like the surface
of a sphere, rather than this "Asteroids" model, then all orientations
would be equally likely.

Now I give it some velocity input. Below, I give extra feedforward
input to every S and E cell.

<video src="/stuff/moving-lattice.mp4" poster="/stuff/moving-lattice.small.png" width="100%" controls loop></video>

<br />

This causes the lattice of active cells to shift. If you record an
individual cell, you'll find that it has a hexagonal firing field,
just like a grid cell.
