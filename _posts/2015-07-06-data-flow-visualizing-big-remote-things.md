---
layout: post
title: "Data flow: Visualizing big remote things"
date:   2015-07-06 12:00:00
---

_\[This approach was flawed. I explain why in [Part 2]({% post_url 2015-07-30-data-flow-visualizing-big-remote-things-part-2 %}).\]_

Here's an interesting problem.

You have this big data structure. Here, I'll show it to you. Don't get bogged down in the details, just scroll past it.

{% highlight clojure %}

{:ff-deps {:rgn-0 [:input], :rgn-1 [:rgn-0]},
 :fb-deps {:input (:rgn-0), :rgn-0 (:rgn-1)},
 :strata [#{:input} #{:rgn-0} #{:rgn-1}],
 :inputs
  {:encoder
   #object[org.nfrac.comportex.encoders$encat$reify__10420 0x3443b2d2 "org.nfrac.comportex.encoders$encat$reify__10420@3443b2d2"],
   :motor-encoder nil,
   :value [:up 0]},
 :regions
 {:rgn-0
  {:layer-3
   {:spec
	{:ff-perm-connected 0.2,
	 :distal-punish? true,
	 :global-inhibition? false,
	 :ff-seg-max-synapse-count 300,
	 :ff-perm-init-hi 0.25,
	 :ff-perm-stable-inc 0.15,
	 :seg-new-synapse-count 12,
	 :ff-perm-init-lo 0.1,
	 :ff-seg-learn-threshold 7,
	 :max-segments 5,
	 :distal-perm-dec 0.01,
	 :boost-active-duty-ratio 0.001,
	 :lateral-synapses? true,
	 :temporal-pooling-max-exc 50.0,
	 :distal-perm-connected 0.2,
	 :seg-learn-threshold 7,
	 :seg-stimulus-threshold 9,
	 :column-dimensions [500],
	 :distal-vs-proximal-weight 0.0,
	 :ff-init-frac 0.25,
	 :distal-perm-inc 0.05,
	 :seg-max-synapse-count 22,
	 :distal-topdown-dimensions [500 4],
	 :ff-seg-new-synapse-count 12,
	 :ff-max-segments 1,
	 :use-feedback? false,
	 :distal-motor-dimensions [0],
	 :boost-active-every 10000,
	 :max-boost 3.0,
	 :ff-potential-radius 0.2,
	 :ff-perm-inc 0.04,
	 :temporal-pooling-amp 5.0,
	 :activation-level 0.04,
	 :distal-perm-punish 0.002,
	 :ff-perm-dec 0.01,
	 :input-dimensions [300],
	 :depth 4,
	 :inhibition-base-distance 1,
	 :duty-cycle-period 1000,
	 :dominance-margin 10.0,
	 :spontaneous-activation? false,
	 :inh-radius-every 1000,
	 :distal-perm-init 0.16,
	 :ff-stimulus-threshold 3,
	 :temporal-pooling-fall 10.0},
	:topology {:size 500},
	:input-topology {:size 300},
	:inh-radius 35,
	:boosts [1.0 1.0 1.0 :BIG ...],
	:active-duty-cycles [0.0 0.0 0.0 :BIG ...],
	:proximal-sg
	{:int-sg
	 {:syns-by-target
	  [{50 0.21874464576529382,
		33 0.21077352910106273,
		22 0.17598592249053957,
		:BIG :BIG
		...}
	   {27 0.1998359104039797,
		4 0.13813471489449805,
		15 0.23550839586167205,
		:BIG :BIG
		...}
	   :ENORMOUS
	   ...],
	  :targets-by-source
	  [#{24 13 51 :BIG ...},
	   #{20 101 85 :BIG ...},
	   #{72 55 54 :BIG ...}
	   :ENORMOUS
	   ...],
	  :pcon 0.2,
	  :cull-zeros? false},
	 :n-cols nil,
	 :depth 1,
	 :max-segs 1},
	:distal-sg
	{:int-sg
	 {:syns-by-target [{} {} {} :ENORMOUS ...],
	  :targets-by-source [#{} #{} #{} :ENORMOUS ...],
	  :pcon 0.2,
	  :cull-zeros? true},
	 :n-cols nil,
	 :depth 4,
	 :max-segs 5},
	:state
	{:in-ff-bits
	 #{65 70 62 74 59 86 72 58 60 69 55 85 39 88 46 77 54 48 50 75 31
	   32 40 56 33 36 41 89 43 61 44 64 51 34 66 47 35 82 76 57 68 83
	   45 53 78 81 79 38 87 30 73 52 67 71 42 80 37 63 49 84},
	 :in-stable-ff-bits #{},
	 :out-ff-bits
	 #{558 287 65 774 620 468 153 621 154 355 352 773 354 683 681 682
	   743 470 646 222 504 505 471 192 221 871 327 353 557 645 402 740
	   31 827 825 506 284 193 869 401 223 826 742 195 29 28 64 623 155
	   285 507 220 697 775 325 741 622 286 152 66 556 772 870 644 868
	   698 824 699 324 680 326 647 559 30 400 696 194 403 469 67},
	 :out-stable-ff-bits #{},
	 :col-overlaps
	 {[126 0] 7,
	  [91 0] 6,
	  [158 0] 6,
	  :BIG :BIG,
	  ...},
	 :matching-ff-seg-paths
	 {[126 0] [126 0 0],
	  [91 0] [91 0 0],
	  [158 0] [158 0 0],
	  :BIG :BIG,
	  ...},
	 :well-matching-ff-seg-paths {},
	 :temporal-pooling-exc {},
	 :active-cols
	 #{7 55 206 88 217 48 139 174 193 117 100 155 170 16 81 38 126 185
	   161 71},
	 :burst-cols
	 #{7 55 206 88 217 48 139 174 193 117 100 155 170 16 81 38 126 185
	   161 71},
	 :active-cells
	 #{[126 0] [7 1] [38 3] [185 0] [88 3] [100 3] [170 1] [139 3]
	   [71 3] [48 1] [88 0] [7 2] [71 0] [100 2] [174 3] [38 2] [174 1]
	   [55 2] [155 3] [88 2] [117 2] [7 3] [81 3] [185 3] [126 2]
	   [206 3] [126 1] [170 0] [71 1] [16 2] [88 1] [161 3] [185 1]
	   [100 1] [117 1] [217 2] [217 1] [38 1] [170 2] [206 2] [206 1]
	   [81 2] [16 3] [117 0] [81 1] [16 0] [38 0] [55 3] [48 0] [48 2]
	   [55 1] [193 2] [174 0] [100 0] [139 0] [174 2] [193 3] [139 1]
	   [155 0] [155 2] [55 0] [7 0] [71 2] [155 1] [193 1] [16 1]
	   [206 0] [170 3] [217 3] [126 3] [139 2] [185 2] [217 0] [193 0]
	   [48 3] [161 2] [81 0] [117 3] [161 1] [161 0]},
	 :learn-cells
	 #{[126 0] [185 0] [88 0] [71 0] [170 0] [117 0] [16 0] [38 0]
	   [48 0] [174 0] [100 0] [139 0] [155 0] [55 0] [7 0] [206 0]
	   [217 0] [193 0] [81 0] [161 0]},
	 :timestep 1,
	 :distal-learning {},
	 :proximal-learning
	 {[126 0] {:target-id [126 0 0],
			   :operation :learn,
			   :grow-sources nil,
			   :die-sources nil},
	  [185 0] {:target-id [185 0 0],
			   :operation :learn,
			   :grow-sources nil,
			   :die-sources nil},
	  :BIG :BIG,
	  ...}},
	:distal-state
	{:distal-bits
	 #{558 287 65 774 620 468 153 621 154 355 352 773 354 683 681 682
	   743 470 646 222 504 505 471 192 221 871 327 353 557 645 402 740
	   31 827 825 506 284 193 869 401 223 826 742 195 29 28 64 623 155
	   285 507 220 697 775 325 741 622 286 152 66 556 772 870 644 868
	   698 824 699 324 680 326 647 559 30 400 696 194 403 469 67},
	 :distal-lc-bits
	 #{620 468 352 504 192 740 284 28 64 220 152 556 772 644 868 824
	   324 680 400 696},
	 :distal-exc {},
	 :pred-cells #{},
	 :matching-seg-paths {},
	 :well-matching-seg-paths {}},
	:prior-distal-state
	{:distal-bits #{},
	 :distal-lc-bits nil,
	 :distal-exc {},
	 :pred-cells #{},
	 :matching-seg-paths nil,
	 :well-matching-seg-paths {}},
	:overlap-duty-cycles nil}},
  :rgn-1
  {:layer-3
   {:spec
	{:ff-perm-connected 0.2,
	 :distal-punish? true,
	 :global-inhibition? false,
	 :ff-seg-max-synapse-count 300,
	 :ff-perm-init-hi 0.25,
	 :ff-perm-stable-inc 0.15,
	 :seg-new-synapse-count 12,
	 :ff-perm-init-lo 0.1,
	 :ff-seg-learn-threshold 7,
	 :max-segments 5,
	 :distal-perm-dec 0.01,
	 :boost-active-duty-ratio 0.001,
	 :lateral-synapses? true,
	 :temporal-pooling-max-exc 50.0,
	 :distal-perm-connected 0.2,
	 :seg-learn-threshold 7,
	 :seg-stimulus-threshold 9,
	 :column-dimensions [500],
	 :distal-vs-proximal-weight 0.0,
	 :ff-init-frac 0.25,
	 :distal-perm-inc 0.05,
	 :seg-max-synapse-count 22,
	 :distal-topdown-dimensions [0],
	 :ff-seg-new-synapse-count 12,
	 :ff-max-segments 5,
	 :use-feedback? false,
	 :distal-motor-dimensions [0],
	 :boost-active-every 10000,
	 :max-boost 3.0,
	 :ff-potential-radius 0.2,
	 :ff-perm-inc 0.04,
	 :temporal-pooling-amp 5.0,
	 :activation-level 0.04,
	 :distal-perm-punish 0.002,
	 :ff-perm-dec 0.01,
	 :input-dimensions [500 4],
	 :depth 4,
	 :inhibition-base-distance 1,
	 :duty-cycle-period 1000,
	 :dominance-margin 10.0,
	 :spontaneous-activation? false,
	 :inh-radius-every 1000,
	 :distal-perm-init 0.16,
	 :ff-stimulus-threshold 3,
	 :temporal-pooling-fall 10.0},
	:topology {:size 500},
	:input-topology {:width 500, :height 4},
	:inh-radius 44,
	:boosts [1.0 1.0 1.0 ...],
	:active-duty-cycles [0.0 0.0 0.0 ...],
	:proximal-sg
	{:int-sg
	 {:syns-by-target
	  [{389 0.21725997970135502,
		291 0.22024731146896143,
		62 0.2497302349798605,
		:BIG :BIG
		...}
	   {}
	   {}
	   :ENORMOUS
	   ...],
	  :targets-by-source
	  [#{410 20 490 :BIG ...} #{70 460 340 ...} :ENORMOUS ...],
	  :pcon 0.2,
	  :cull-zeros? false},
	 :n-cols nil,
	 :depth 1,
	 :max-segs 5},
	:distal-sg
	{:int-sg
	 {:syns-by-target [{} {} {} ...],
	  :targets-by-source [#{} #{} #{} ...],
	  :pcon 0.2,
	  :cull-zeros? true},
	 :n-cols nil,
	 :depth 4,
	 :max-segs 5},
	:state
	{:in-ff-bits
	 #{558 287 65 774 620 468 153 621 154 355 352 773 354 683 681 682
	   743 470 646 222 504 505 471 192 221 871 327 353 557 645 402 740
	   31 827 825 506 284 193 869 401 223 826 742 195 29 28 64 623 155
	   285 507 220 697 775 325 741 622 286 152 66 556 772 870 644 868
	   698 824 699 324 680 326 647 559 30 400 696 194 403 469 67},
	 :in-stable-ff-bits #{},
	 :out-ff-bits
	 #{765 287 62 213 670 592 463 60 322 901 1006 527 300 902 119 293
	   141 945 394 116 669 811 827 825 460 284 214 294 810 117 944 947
	   1007 826 292 525 143 707 668 303 671 118 393 461 61 301 323 295
	   808 285 1005 594 286 142 526 302 215 704 946 462 706 824 524 705
	   900 766 392 140 321 320 593 595 767 809 903 395 63 212 764
	   1004},
	 :out-stable-ff-bits #{},
	 :col-overlaps
	 {[126 0] 3,
	  [120 0] 7,
	  [91 0] 4,
	  :BIG :BIG,
	  ...},
	 :matching-ff-seg-paths
	 {[126 0] [126 0 0],
	  [120 0] [120 0 0],
	  [91 0] [91 0 0],
	  :BIG :BIG,
	  ...},
	 :well-matching-ff-seg-paths {[115 0] [115 0 0]},
	 :temporal-pooling-exc {},
	 :active-cols
	 #{206 225 176 15 251 75 191 167 131 29 148 236 35 202 115 53 98 73
	   71 80},
	 :burst-cols
	 #{206 225 176 15 251 75 191 167 131 29 148 236 35 202 115 53 98 73
	   71 80},
	 :active-cells
	 #{[251 1] [53 2] [191 0] [202 3] [80 3] [225 0] [35 2] [148 2]
	   [71 3] [80 0] [75 2] [15 0] [98 2] [71 0] [131 0] [236 1] [15 3]
	   [202 1] [98 0] [115 3] [98 1] [202 2] [53 1] [148 3] [167 2]
	   [225 2] [191 1] [176 3] [206 3] [176 0] [71 1] [80 1] [35 3]
	   [225 3] [191 2] [53 3] [167 0] [115 2] [75 1] [176 1] [29 0]
	   [206 2] [206 1] [98 3] [251 3] [167 3] [191 3] [35 0] [148 1]
	   [29 2] [35 1] [131 2] [73 2] [15 1] [202 0] [29 1] [75 0]
	   [131 1] [167 1] [80 2] [236 3] [73 0] [148 0] [75 3] [73 3]
	   [176 2] [71 2] [236 0] [29 3] [251 2] [131 3] [115 1] [53 0]
	   [206 0] [251 0] [115 0] [73 1] [236 2] [225 1] [15 2]},
	 :learn-cells
	 #{[191 0] [225 0] [80 0] [15 0] [71 0] [131 0] [98 0] [176 0]
	   [167 0] [29 0] [35 0] [202 0] [75 0] [73 0] [148 0] [236 0]
	   [53 0] [206 0] [251 0] [115 0]},
	 :timestep 1,
	 :distal-learning {},
	 :proximal-learning
	 {[191 0] {:target-id [191 0 0],
			   :operation :learn,
			   :grow-sources nil,
			   :die-sources nil},
	  [225 0] {:target-id [225 0 0],
			   :operation :learn,
			   :grow-sources nil,
			   :die-sources nil},
	  :BIG :BIG,
	  ...}},
	:distal-state
	{:distal-bits
	 #{765 287 62 213 670 592 463 60 322 901 1006 527 300 902 119 293
	   141 945 394 116 669 811 827 825 460 284 214 294 810 117 944 947
	   1007 826 292 525 143 707 668 303 671 118 393 461 61 301 323 295
	   808 285 1005 594 286 142 526 302 215 704 946 462 706 824 524 705
	   900 766 392 140 321 320 593 595 767 809 903 395 63 212 764
	   1004},
	 :distal-lc-bits
	 #{592 60 300 116 460 284 944 292 668 808 704 824 524 900 392 140
	   320 212 764 1004},
	 :distal-exc {},
	 :pred-cells #{},
	 :matching-seg-paths {},
	 :well-matching-seg-paths {}},
	:prior-distal-state
	{:distal-bits #{},
	 :distal-lc-bits nil,
	 :distal-exc {},
	 :pred-cells #{},
	 :matching-seg-paths nil,
	 :well-matching-seg-paths {}},
	:overlap-duty-cycles nil}}}}

{% endhighlight %}

That's a truncated version. I marked a few places with `:BIG` and `:ENORMOUS`. From now on I'll refer to this data structure as a "model".

This model's meaning can be [visualized](https://nupic-community.github.io/comportexviz/).

Here's a new challenge: visualize it remotely. It lives on a server. You're a browser. Go.

This is similar to asking, "What if Google's Maps live on a server, and you want to visualize them in a browser?". The browser only shows a small fraction of the data in the model, so it does not make sense to download the whole thing.

## Just a simple get-in, yes?

The server could just give [`get-in`](https://clojuredocs.org/clojure.core/get-in) access to the model over HTTP. Wouldn't that be good enough?

Well, the model is really complicated, and you already have lots of code that knows how to make sense of it. For example:

{% highlight clojure %}

(defrecord LayerOfCells
	[spec topology input-topology inh-radius boosts active-duty-cycles
	 proximal-sg distal-sg state distal-state prior-distal-state]
  p/PLayerOfCells
  (temporal-pooling-cells [_]
	(keys (:temporal-pooling-exc state)))
  (predictive-cells [_]
	(:pred-cells distal-state))
  (prior-predictive-cells [_]
	(:pred-cells prior-distal-state))
  ...)

{% endhighlight %}

These three useful methods, `temporal-pooling-cells`, `predictive-cells`, and `prior-predictive-cells` each access different fields: `:state`, `:distal-state`, and `:prior-distal-state`. So if you want to use these methods, you first need to make sure you have these fields handy. But `:state` is quite large, so you should only fetch its `:temporal-pooling-exc` and `:pred-cells` fields.

Stop the car. You're manually prefetching the data that your methods touch? We took a wrong turn somewhere.

You (the browser) need to use this code, though, or we'll be maintaining two codebases and the project will collapse under its own weight.

The inevitable conclusion: the server needs to run code on your behalf and then give you the result. Rather than fetching `:state`, `:distal-state`, ..., we should fetch the return value of `temporal-pooling-cells`, `predictive-cells`, and so on. Then, in the browser, substitute your own `p/PLayerOfCells` methods which simply return this cached value.

So, no, a `get-in` server API is not sufficient. You need an API for calling whitelisted functions on values. For example the browser will send request

{% highlight clojure %}

[[:temporal-pooling-cells :my-arg1 :my-arg2] [:regions :rgn-0 :lyr-3]]

{% endhighlight %}

and the server will have

{% highlight clojure %}

{:temporal-pooling-cells p/temporal-pooling-cells}

{% endhighlight %}

in a lookup table somewhere. I'm being hand-wavey, you can fill in the blanks. Send over HTTP, etc.

## Oh, I get it, prefetching is always bad!

Nope. Prefetching is not bad. We were just doing it in a bad way. Prefetching should be the default, and we should only not do it when it results in too large of a download.

When you choose to grab a value on-demand, the code that needs it needs to know how to grab it, and needs to be asynchronous. And you have to decide whether or not to build some caching / memoize system. If you prefetch some subset of the model before doing anything with it, you save yourself a lot of complexity.

How much of the model is actually enormous? Can we just prune the big parts? If I can fit a pruned model in a blog post, surely we can download it in app code.

So we'll:

- Prune the big parts.
- Prune the parts we don't need.
- Add some computed values.

All of this will go on a "proxy" for the model. Via this proxy, you can do on-demand `get-in` or whitelisted function calls to the server, and you can access all of the prefetched data directly.

## The on-demand parts

Let's talk about the SynapseGraph, e.g. `:proximal-sg` in the model. There are potentially thousands of columns, and only one can be selected at a time, so this is exactly the kind of thing that is both:

1. the culprit of the hugeness
2. amenable to pruning

So, skip ahead. It's pruned. What now? How will we access it? The same way as above, via `get-in` and whitelisted function calls, providing the path down to it.

{% highlight clojure %}

[[:in-synapses [20 0 0]] [:regions :rgn-0 :lyr-3 :proximal-sg]]

{% endhighlight %}

This works, but sometimes you'll run into issues where the code that handles the SynapseGraph doesn't know enough to reconstruct this path. It's worth providing a way to create subproxies.

So, rather than simply pruning the big parts, replace them with subproxies.

## A query language

Here's my first shot. Here's a subset of what I'd send to the server for the initial prefetch:

{% highlight clojure %}

{:values [:ff-deps :fb-deps :strata :inputs]
 :methods [[:timestep]]
 :children {:regions
		{;; All regions
		 :all-children
		 {;; All layers
		  :all-children {:values [:spec]
				 :methods [[:active-columns]
					   [:active-cells]
					   [:source-of-bit 42]
					   [:source-of-bit 43]
					   [:source-of-bit 44]
					   [:params]]
				 :proxies [:proximal-sg
					   :distal-sg]
				 :children {:state {:values [:in-ff-bits]}}}}}}}

{% endhighlight %}

Lots of this is unambiguous. The server will return something similar to the model, but with only the specified values, with proxies in place of the other specified values. The specified method calls will happen. The only remaining question is where the results of the method calls go.

Easy enough. Here's one example of how the `:timestep` result might be stored:

{% highlight clojure %}

{:ff-deps {:rgn-0 [:input], :rgn-1 [:rgn-0]},
 :fb-deps {:input (:rgn-0), :rgn-0 (:rgn-1)},
 :method-results {[:timestep] 42},
 :strata [#{:input} #{:rgn-0} #{:rgn-1}]
 ...}

{% endhighlight %}

The client code that talks to the server could then specify `PTemporal` onto the model proxy, making the `timestep` method return `(get-in this [:method-results [:timestep]])`.
