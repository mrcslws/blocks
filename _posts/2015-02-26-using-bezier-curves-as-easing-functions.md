---
layout: post
title: "Using Bézier curves as easing functions"
date:   2015-02-26 12:00:00
---

You have two endpoints.

At x 0, y is 20.
At x 200, y is 50.

You're going to plot x along some dimension. If you're building an animation, this dimension is "time". In other cases, you'll plot x along an actual axis of a graph.

You need to choose midpoints. You can generalize this problem as

$$ y = 20 + (50 - 20) * E(\frac{x} {200}) $$

Now you just need to define an easing function (E). You'll have a conversation that goes something like:

> **E**: On a scale of 0 to 1, what is x?  
> **You**: \_\_\_  
> **E**: Okay, on a scale of 0 to 1, y is \_\_\_  
> **You** *(sometimes)*: That's not between 0 and 1!  
> **E** *(sometimes)*: I do what I want.  

A linear E is simply:

$$ E(x) = x $$

A sinusoidal E is more gentle.

$$ E(x) = \frac{\sin(\pi(x - \frac{1}{2}))}{2} $$

These are fine. But we're speaking the wrong language. You're trying to do something visual... why would you speak in equations? Let's remove the handcuffs.

## Cubic Bézier curves

A 2D cubic Bézier curve is defined by:

$$
x(t) = x_0(1 - t)^3 + 3x_1(1-t)^2t + 3x_2(1-t)t^2 + x_3t^3 \\
y(t) = y_0(1 - t)^3 + 3y_1(1-t)^2t + 3y_2(1-t)t^2 + y_3t^3 \\
t \in [0,1]
$$

You just plug in four points. If you want to coerce a cubic Bézier curve into an easing function, two of these points are already known.

$$
x_0 = 0, y_0 = 0 \\
x_3 = 1, y_3 = 1
$$

The others are up to you. For example:

$$
x_1 = 0.785, y_1 = 0.135 \\
x_2 = 0.15, y_2 = 0.86
$$

So this seems useful for drawing a line... but how do we get an easing function out of this? We need a function of x, not a function of t.

Here's a useful strategy: appeal to authority!

## How do Gecko / Blink / WebKit solve this?

CSS transitions can use "cubic-bezier" as their timing function. So let's just assume that the browser vendors have figured out the best way to do this.

Gecko:  
<https://dxr.mozilla.org/mozilla-central/source/dom/smil/nsSMILKeySpline.cpp>

WebKit / Blink: (mostly the same)  
<https://chromium.googlesource.com/chromium/blink/+/master/Source/platform/animation/UnitBezier.h>
<http://www.opensource.apple.com/source/WebCore/WebCore-7600.1.25/platform/graphics/UnitBezier.h>

Chrome Compositor:  
<https://chromium.googlesource.com/chromium/src/+/master/ui/gfx/geometry/cubic_bezier.cc>


### The general approach:
Search for the t value that corresponds with the given x. Use the Newton-Raphson method for better perf, but fall back to bisection as necessary / convenient.

All three use Horner's method to calculate both the Bézier of t and the slope at t.

### Bisection
The Chrome Compositor's approach is the least advanced -- it jumps right to bisection. It starts with t of 0 and takes steps of size 1, 0.5, 0.25, 0.125 ... until it finds a t that results in an x value within 0.0000001 of the specified value.

WebKit and Blink's fallback is just slightly more advanced. It starts with a better guess, using t of x.

Gecko stores 11 sample values for a Bézier function and uses them to choose a good first guess. So its first guess is going to choose a t within 0.1 of the correct answer, and its steps will be 0.05, 0.025, ...

### Newton's method
We can calculate the slope of a Bézier curve w.r.t. t via:

$$
x'(t) = 3(1 - t)^2(x_1 - x_0) + 6(1-t)t(x_2 - x_1) + 3t^2(x_3 - x_2)
$$

Instead of binary-searching for the correct value, use a smarter step approach.

\\[ [next\ guess] = [previous\ guess] - \frac{[previous\ error]}{[slope\ at\ previous\ guess]} \\]

The WebKit / Blink code starts by guessing t of x, and does up to 8 steps until it finds an x within [epsilon] of the specified value. It uses epsilon: (via [`accuracyForDuration`](https://chromium.googlesource.com/chromium/blink/+/master/Source/platform/animation/AnimationUtilities.h))
\\[ \epsilon = \frac{1}{200 * [animation\ duration]} \\]

It also gives up if it sees a slope of less than 0.000001. If it doesn't find what it's looking for, it falls back to bisection.

Gecko starts with a better guess (via its 11 sample values), then does 4 steps and then just returns whatever it ends up with. It only uses the fallback if it sees the slope for its first guess is less than 0.02. And it only returns early if it reaches a point where the slope is 0, to avoid divide-by-zero -- i.e., error handling. x'(t) should never be 0, otherwise this wouldn't be a valid easing function.

[hello]: https://chromium.googlesource.com/chromium/blink/+/master/Source/platform/animation/AnimationUtilities.h "goodbye"



### Horner's method

All three implemenations transform

$$
x(t) = x_0(1 - t)^3 + 3x_1(1-t)^2t + 3x_2(1-t)t^2 + x_3t^3 \\
x'(t) = 3(1 - t)^2(x_1 - x_0) + 6(1-t)t(x_2 - x_1) + 3t^2(x_3 - x_2)
$$

into

$$
x(t) = (((1 - 3x_2 + 3x_1)t + 3x_2 - 6x_1)t + 3x_1)t \\
x'(t) = 3(1 - 3x_2 + 3x_1)t^2 + 2(3x_2 - 6x_1)t + 3x_1
$$

by plugging in the start and end points and then using Horner's method. Blink / WebKit only calculate these coefficients once. In fact, the code is quite pretty.

{% highlight c %}
cx = 3.0 * p1x;
bx = 3.0 * (p2x - p1x) - cx;
ax = 1.0 - cx - bx;
{% endhighlight %}


## The code

Enter ClojureScript.

{% highlight clojurescript %}
(defprotocol PBezierDimension
  (t->coord [this t])
  (t->coord-slope [this t]))

(defprotocol PBezier
  (t->x [this t])
  (t->y [this t]))

(defprotocol PEasingBezier
  (x->y [this x])

  ;; It's simple to implement this on top of x->y, but that will duplicate a
  ;; lot of searching. Give implementors the chance to use a faster approach.
  (foreach-xy [this x-count f]))

;; Terminology: a "Unit Bézier" uses 0 and 1 as its first and last points.
(deftype CubicUnitBezierDimension [a b c]
  PBezierDimension
  (t->coord [this t]
    (-> t
        (* a)
        (+ b)
        (* t)
        (+ c)
        (* t)))
  (t->coord-slope [this t]
    (-> t
        (* 3 a)
        (+ (* 2 b))
        (* t)
        (+ c))))

(defn cubic-unit-bezier-dimension [coord1 coord2]
  ;; Horner's method. All the browsers do this, so it must be right.
  (let [c (* 3 coord1)
        b (- (* 3 (- coord2 coord1))
             c)
        a (- 1 c b)]
    (CubicUnitBezierDimension. a b c)))

(defn x->t [xdim x guess epsilon]
  (let [;; Newton method
        t (loop [t x
                 i 0]
            (when (< i 8)
              (let [error (- (t->coord xdim t)
                             x)]
                (if (< (js/Math.abs error) epsilon)
                  t
                  (let [d (t->coord-slope xdim t)]
                    (when (>= (js/Math.abs d) 1e-6)
                      (recur (- t (/ error d))
                             (inc i))))))))

        ;; Fall back to bisection
        t (if t
            t
            (loop [t 0
                   i 0
                   step 1]
              (assert (< i 30))
              (let [error (- (t->coord xdim t)
                             x)]
                (if (< (js/Math.abs error) epsilon)
                  t
                  (recur (if (pos? error)
                           (- t step)
                           (+ t step))
                         (inc i)
                         (/ step 2))))))]
    t))

(deftype CubicBezierEasing [xdim ydim]
  PBezier
  (t->x [_ t]
    (t->coord xdim t))
  (t->y [_ t]
    (t->coord ydim t))

  PEasingBezier
  (x->y [this x]
    (t->y this
          (x->t xdim x x 1e-7)))
  (foreach-xy [this xcount f]
    (let [step (/ 1 xcount)
          epsilon (/ step 200)]
      (loop [i 0
             guess 0]
        (when (< i xcount)
          (let [x (* i step)
                t (x->t xdim x guess epsilon)]
            (f x (t->y this t))
            ;; Use this t to improve the next guess.
            (recur (inc i)
                   (+ t step))))))))

(defn cubic-bezier-easing [x1 y1 x2 y2]
  (CubicBezierEasing. (cubic-unit-bezier-dimension x1 x2)
                      (cubic-unit-bezier-dimension y1 y2)))
{% endhighlight %}

Now just do

{% highlight clojurescript %}
(def my-easing-function (cubic-bezier-easing 0.785 0.135 0.15 0.86))

;; For animations, use x->y, using (/ time duration) as the x.
(x->y my-easing-function (/ (- (js/Date.now) animation-start) duration))

;; For graphing, use foreach-xy to get better perf.
(foreach-xy my-easing-function column-count
            (fn [x y]
              (let [col (js/Math.round (* x width))
                    row (js/Math.round (* (- 1 y) height))]
                ;; do stuff
              )))
{% endhighlight %}

## Freedom

You've escaped the world of equations. Go choose your Bézier curve!

You might appreciate <http://cubic-bezier.com>

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
