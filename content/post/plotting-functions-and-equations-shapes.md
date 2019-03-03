---
title: "Plotting Functions, Equations, Shapes and Combinations of Them using Shadertoy"
date: 2019-02-14T14:00:00+02:00
mathjax : true
---

> **UPDATE**: added automatic differentiation via dual numbers section

In computer graphics plotting functions is invaluable tool in order to understand any topic. There are a lot of specialized software, but nothing that can visualize 2d, 3d and custom content easily. One of the greatest tools for visualization is Shadertoy. So my goal was to develop tools to be able to generate all kinds of plots.

Plotting shapes is quite straightforward just evaluate $F(x) >= 0$ and you are done. Plotting functions is not so easy. In order to do that coverage is required per pixel. One of simplest coverage measures is just evaluating distance in pixels and using `smoothstep` function in order to generate gradient over some interval, usually `1px` or `1.5px`. Same coverage strategy can be used for shapes as well.

The simplest way to estimate distance I have found in [Loop, Blinn. Rendering Vector Art on the GPU ](https://developer.nvidia.com/gpugems/GPUGems3/gpugems3_ch25.html) GPU Gems 3 article. Ideas is to estimate distance as:
$$sd=\frac{F(x, y)}{||\nabla F(x, y)||}=\frac{F(x, y)}{\sqrt{(\frac{\partial F}{\partial x})^2+(\frac{\partial F}{\partial y})^2}}$$

With this knowledge its simple to render any implicit curve or function. As we deal with 2d screen space function our $sd$ will be always multiple of screen space pixels. So we need just to map it to $[0..1]$ interval. This is can be done for example with `1.0-smoothstep(px*(w/2.0-0.5), px*(w/2.0+0.5), abs(sd))` where `w` is approximate width required.

This technique generates artifacts in some cases, specifically near critical points of $F(x, y)$ function, where $\nabla F(x, y)=0$, or discontinuities like in $\tan(x)$ function.

### Logic operations as continuous functions

When we have factors in $[0..1]$ interval we can do even more interesting things like taking intersection or union of shapes. List of possible operations and corresponding formulas is in the following table([source](https://www.shadertoy.com/view/Xdj3Rh)):

Function|Formula|Note
---|---|---
$\neg A$|$c = 1.0 - a$|NOT
$A \land B$|$c = a \cdot b$|AND
$A \oplus B$|$c = a + b - 2.0 \cdot a \cdot b$|XOR
$A \lor B$|$c = a + b - a \cdot b$|OR
$\neg(A \land B)$|$c = 1.0 - a \cdot b$|NAND
$\neg(A \lor B)$|$c = 1.0 - a - b + a \cdot b$|NOR
$\neg(A \oplus B)$|$c = 1.0 - b - a + 2.0 \cdot a \cdot b$|XNOR

### Automatic differentiation via dual numbers

Major limitation with approach described above is manually writing derivatives. Fortunately exist a way to automatically differentiate functions. More details can be found in:

 * [Dual Numbers & Automatic Differentiation](https://blog.demofox.org/2014/12/30/dual-numbers-automatic-differentiation/)
 * [Multivariable Dual Numbers & Automatic Differentiation](https://blog.demofox.org/2017/02/20/multivariable-dual-numbers-automatic-differentiation/)

Below is short excerpt on dual numbers basic operations:

Operation|Formula
---|---
$x$ <br/> $x,\ y$ |$x + \epsilon 1$ <br/> $x + \epsilon\_x1 + \epsilon\_y0,\ y + \epsilon\_x0 + \epsilon\_y1$
$(a + \epsilon b) + (c + \epsilon d)$ <br/> $(a + \epsilon\_x b + \epsilon\_y c) + (d + \epsilon\_x e + \epsilon\_y f)$|$ (a + c) + \epsilon (b + d)$ <br/> $(a + d) + \epsilon\_x(b + e) + \epsilon\_y(c + f)$
$(a + \epsilon b)(c + \epsilon d)$ <br/> $(a + \epsilon\_x b + \epsilon\_y c)(d + \epsilon\_x e + \epsilon\_y f)$|$ac + \epsilon (ad + bc)$ <br/> $ad+\epsilon\_x(ae+bd)+\epsilon\_y(af+cd)$
$f(a + \epsilon b)$ <br/> $f(a + \epsilon\_x b + \epsilon\_y c)$|$f(a) + \epsilon b f^\prime(a)$ <br/> $f(a) + \epsilon\_x b f^\prime(a) + \epsilon\_y c f^\prime(a)$


--------

Here is shadertoy illustration of everything discussed above:

<iframe src="https://www.shadertoy.com/embed/wsSGRz" style="width: 640px; height: 380px;"></iframe>

-----

### Appendix: GLSL shader library code

```c++
// The MIT License
// Copyright © 2019 Mykhailo Parfeniuk
// Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions: The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software. THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

// f(x)

vec2 const_dn2(float x)
{
    return vec2(x, 0.0);
}

vec2 dn(float x)
{
    return vec2(x, 1.0);
}

vec2 mul_dn(vec2 a, vec2 b)
{
    return vec2(a.x*b.x, a.x*b.y+b.x*a.y);
}

vec2 rcp_dn(vec2 a)
{
    return vec2(1.0/a.x, -a.y/(a.x*a.x));
}

vec2 exp_dn(vec2 x)
{
    return vec2(exp(x.x), x.y*exp(x.x));
}

vec2 sin_dn(vec2 x)
{
    return vec2(sin(x.x), x.y*cos(x.x));
}

vec2 cos_dn(vec2 x)
{
    return vec2(cos(x.x), -x.y*sin(x.x));
}

vec2 tan_dn(vec2 x)
{
    float c = cos(x.x);
    return vec2(tan(x.x), x.y/(c*c));
}

vec2 clamp_dn(vec2 x, float a, float b)
{
    return vec2(clamp(x.x, a, b), x.y*(step(a, x.x) - step(b, x.x)));
}

vec2 saturate_dn(vec2 x)
{
    return clamp_dn(x, 0.0, 1.0);
}

float estimateDist_dn(in float y, vec2 f)
{
    return abs(y - f.x)/length(vec2(1.0, f.y));
}

// F(x, y)

vec3 const_dn3(float x)
{
    return vec3(x, 0.0, 0.0);
}

vec3 dn_x(float x)
{
    return vec3(x, 1.0, 0.0);
}

vec3 dn_y(float x)
{
    return vec3(x, 0.0, 1.0);
}

vec3 mul_dn(vec3 x, vec3 y)
{
    return vec3(x.x*y.x, x.x*y.y+y.x*x.y, x.x*y.z+y.x*x.z);
}

vec3 rcp_dn(vec3 a)
{
    return vec3(1.0/a.x, -a.yz/(a.x*a.x));
}

vec3 exp_dn(vec3 x)
{
    return vec3(exp(x.x), x.yz*exp(x.x));
}

vec3 sin_dn(vec3 x)
{
    return vec3(sin(x.x), x.yz*cos(x.x));
}

vec3 cos_dn(vec3 x)
{
    return vec3(cos(x.x), -x.yz*sin(x.x));
}

vec3 tan_dn(vec3 x)
{
    float c = cos(x.x);
    return vec3(tan(x.x), x.yz/(c*c));
}

vec3 clamp_dn(vec3 x, float a, float b)
{
    return vec3(clamp(x.x, a, b), x.yz*(step(a, x.x) - step(b, x.x)));
}

vec3 saturate_dn(vec3 x)
{
    return clamp_dn(x, 0.0, 1.0);
}

float estimateDist_dn(vec3 F)
{
    float grad_norm = length(F.yz);
    return grad_norm > 1e-10 ? F.x/grad_norm : 1e37;
}

// Draw utils

float opNOT(float a)
{
	return 1.0-a;   
}

float opAND(float a, float b)
{
	return a*b;   
}

float opOR(float a, float b)
{
	return a+b-a*b;   
}

float opXOR(float a, float b)
{
	return a+b-2.0*a*b;   
}

float mask_dn(float px, float y, vec2 f)
{
    return 1.0-smoothstep(1.0*px, 2.5*px, estimateDist_dn(y, f));
}

float maskeq_dn(float px, vec3 F)
{
    return 1.0-smoothstep(1.0*px, 2.5*px, abs(estimateDist_dn(F)));
}

float maskneq_dn(float px, vec3 F)
{
    return 1.0-smoothstep(-1.0*px, 1.0*px, estimateDist_dn(F));
}

// Helpers

vec3 drawBg(vec2 p, float px)
{
    vec3 col = vec3(0.3 + 0.04*mod(floor(p.x)+floor(p.y),2.0));
    col *= smoothstep( 0.5*px, 1.5*px, abs(p.x) );
    col *= smoothstep( 0.5*px, 1.5*px, abs(p.y) );
    return col;
}

float maskRect(vec2 p, float dx, float dy)
{
    return (step(-dx, p.x) - step(dx, p.x))*(step(-dy, p.y) - step(dy, p.y));
}

float saturate(float x)
{
    return clamp(x, 0.0, 1.0);
}
```


