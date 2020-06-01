---
layout: post
title: 《Design Principles and Design Patterns》读书
category: 读书笔记
tags: 笔记
---

## Architecture and Dependencies
### Symptoms of Rotting Design
**Rigidity.** Rigidity is the tendency for software to be difficult to change, even in 
simple ways.Every change causes a cascade of subsequent changes in dependent 
modules.

僵化。不仅仅是在扩张上很难，甚至修改一个小的问题也可能导致花费非常多时间。

The software design 
begins to take on some characteristics of a roach motel -- engineers check in, but they 
don’t check out.

这比喻也是没有谁了，软件开始有了蟑螂盒的特性，工程师开始解决问题check in，但一直没有check out。

**Fragility.** Closely related to rigidity is fragility. Fragility is the tendency of the 
software to break in many places every time it is changed.

脆弱。一个修改，引起一连串的问题。

**Immobility.** Immobility is the inability to reuse software from other projects or 
from parts of the same project. It often happens that one engineer will discover that he 
needs a module that is similar to one that another engineer wrote.

固化，不可服用。

**Viscosity.** Viscosity comes in two forms: viscosity of the design, and viscosity of 
the environment. When faced with a change, engineers usually find more than one 
way to make the change. Some of the ways preserve the design, others do not (i.e. 
they are hacks.) When the design preserving methods are harder to employ than the 
hacks, then the viscosity of the design is high. It is easy to do the wrong thing, but 
hard to do the right thing.

耦合高。