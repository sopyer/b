---
title: "Plotting Functions, Equations, Shapes and Combinations of Them using Shadertoy"
date: 2019-02-14T14:00:00+02:00
mathjax : true
---

In computer graphics plotting functions is invaluable tool in order to understand any topic. There are a lot of specialized software, but nothing that can visualize 2d, 3d and custom content easily. One of the greatest tools for visualization is Shadertoy. So my goal was to develop tools to be able to generate all kinds of plots.

Plotting shapes is quite straightforward just evaluate $F(x) >= 0$ and you are done. Plotting functions is not so easy. In order to do that coverage is required per pixel. One of simplest coverage measures is just evaluating distance in pixels and using `smoothstep` function in order to generate gradient over some interval, usually `1px` or `1.5px`. Same coverage strategy can be used for shapes as well.

The simplest way to estimate distance I have found in [Loop, Blinn. Rendering Vector Art on the GPU ](https://developer.nvidia.com/gpugems/GPUGems3/gpugems3_ch25.html) GPU Gems 3 article. Ideas is to estimate distance as:
$$sd=\frac{F(x, y)}{||\nabla F(x, y)||}=\frac{F(x, y)}{\sqrt{(\frac{\partial F}{\partial x})^2+(\frac{\partial F}{\partial y})^2}}$$

With this knowledge its simple to render any implicit curve or function. As we deal with 2d screen space function our $sd$ will be always multiple of screen space pixels. So we need just to map it to $[0..1]$ interval. This is can be done for example with `1.0-smoothstep(px*(w/2.0-0.5), px*(w/2.0+0.5), abs(sd))` where `w` is approximate width required.

So when we have factors in $[0..1]$ interval we can do even more interesting things like taking intersection or union of shapes. List of possible operations and corresponding formulas is in following table([source](https://www.shadertoy.com/view/Xdj3Rh)):

### Logic operations as continuous functions

Function|Formula|Note
---|---|---
$\neg A$|$c = 1.0 - a$|NOT
$A \land B$|$c = a \cdot b$|AND
$A \oplus B$|$c = a + b - 2.0 \cdot a \cdot b$|XOR
$A \lor B$|$c = a + b - a \cdot b$|OR
$\neg(A \land B)$|$c = 1.0 - a \cdot b$|NAND
$\neg(A \lor B)$|$c = 1.0 - a - b + a \cdot b$|NOR
$\neg(A \oplus B)$|$c = 1.0 - b - a + 2.0 \cdot a \cdot b$|XNOR

This technique generates artifacts in some cases, specifically near critical points of $F(x, y)$ function, where $\nabla F(x, y)=0$, or discontinuities like in $\tan(x)$ function.

Here is shadertoy illustration of everything discussed above:

<iframe src="https://www.shadertoy.com/embed/wsSGRz" style="width: 640px; height: 380px;"></iframe>



