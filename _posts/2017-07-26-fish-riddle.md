---
layout: post
title:  "Fish Riddle"
date:   2017-07-26
author: xfbs
---

The internet is full of distractions, and unfortunately, I am not always impervious to all of them. Some of them can lead to interesting results. Today, my distraction came in the shape of a riddle from a riddle from a TED-Ed video, which got me to explore my (rusty, but still somewhat present) math skills.

{% youtube https://youtu.be/lLOALyWls2k&rel=0 %}

Now, I was excited to learn about the puzzle to see if I could use programming (I was thinking of a constraint solver or possibly just brute-forcing it) to solve it. But alas, it turns out that it’s just solvable with plain maths 🤷🏽‍♀️. So let’s dive in and see what we can do here.

## The problem

You can watch the video to get the story of the puzzle, but it breaks down like this: you have three quadrants, each with a number of fish tanks and sharks in it. You know how many there are in both the first and second quadrants, and you must find out how many there are in the third quadrant.

So let’s first introduce a number of variables to help is keep track of things. We define:
- $$ s_i$$ as the number of creatures in sector $$ i$$,
- $$ h_i$$ as the number of sharks in sector $$ i$$,
- $$ f_i$$ as the number of fish in sector $$ i$$,
- $$ t_i$$ as the number of fish per tank in sector $$ i$$,
- $$ n_i$$ as the number of fish tanks in sector $$ i$$.

## The constraints

There are six comstraints given in the puzzle.

1. **There are 50 creatures in total,** including sharks and fish.

$$\begin{array}{rcl}
f_i & = & n_i t_i\\
s_i & = & h_i f_i\\
\sum_{i=1}^{3} s_i & = & 50
\end{array}$$

2. **Each sector has anywhere from one to seven sharks, with no two sectors
having the same nmber of sharks**.

$$ \begin{array}{rl}
\forall i \in \{1, 2, 3\}: & 1 \leq h_i \leq 7\\
\forall i, j \in \{1, 2, 3\}: & h_i \neq h_j
\end{array}$$

3. **Each tank has an equal number of fish**.
*Since this is true, we will just use $$t$$ to refer to any of $$t_i$$, simce they are all the same*.

$$ \begin{array}{rl}
\forall i, j \in \{1, 2, 3\}: & t_i = t_j
\end{array}$$

4. **In total, there are 13 or fewer fish tanks**.

$$ \begin{array}{rcl}
\sum_{i=1}^{3} n_i & \leq & 13
\end{array}$$

5. **Sector Alpha has two sharks and four tanks**.

$$ \begin{array}{rcl}
h_1 & = & 2\\
n_1 & = & 4
\end{array}$$

6. **Sector Beta has four sharks and two tanks**.

$$ \begin{array}{rcl}
h_2 & = & 4\\
n_2 & = & 2
\end{array}$$

## Solution

The objective for us is to find both the values of $$ t$$, $$ h_3$$ and $$ n_3$$. To do this, I started out by using the given constraints to find the number of possible values for each of them.

Applying the given constraint #2, we can limit the search space for $$ h_3$$ easily.

$$ \begin{array}{rcl}
h_3 & \in & \{ x \in \mathbb{N} | 1 \leq x \leq 7, \forall i \in \{1, 2\}: x \neq h_i\}\\
h_3 & \in & \{1, 3, 5, 6, 7\}
\end{array}$$

Similarly, using the given constraint #3, we can limit the search space for $$ n_3$$ easily.

$$ \begin{array}{rcl}
n_3 & \in & \{ x \in \mathbb{N} | 0 \leq x \leq \sum_{i=1}^{2} n_i\}\\
n_3 & \in & \{0, 1, 2, 3, 4, 5, 6, 7\}
\end{array}$$

Finding out the search space for $$ t$$ is unfortunately not that simple. First, we need to know how many creatures are currently accounted for.

$$ \begin{array}{rcl}
\sum_{i=1}^{2} s_i & = & 6 + 6t\\
\end{array}$$

From constraint 1, we know that there are 50 creatures. Thus, from the amount of creatures we have right now and from that, we can calculate how many are not accounted for yet, $$ r$$ (for *rest*). We also know that the missing creatures must be in our sector (sector Gamma), so we have ourselves a nice simple equation.

$$ \begin{array}{rcl}
r & = & 50 - 6 + 6t\\
r & = & h_3 + n_3 t\\
44 - 6t & = & h_3 + n_3 t\\
44 & = & h_3 + (6 + n_3) t
\end{array}$$

Now, given this equation and knowing the search space for both $$ h_3$$ and $$ n_3$$, we can easily restrict the search space for $$ t$$. If we pop in the maximum values for $$ h_3$$ and $$ n_3$$ and solve it, we can find the minimum value for $$ t$$, and vice versa for the maximum value.

$$ \begin{array}{rcl}
44 & = & 7 + (6 + 7) t_{min}\\
t_{min} & \approx & 3\\
44 & = & 1 + (6 + 0) t_{max}\\
t_{max} & \approx & 7
\end{array}$$

With this information, we can limit the search space for $$ t$$, because we know that it must be within the bounds of $$ t_{min}$$ and $$ t_{max}$$.

$$ \begin{array}{rcl}
t & \in & \{ x \in \mathbb{N} | t_{min} \leq x \leq t_{max}\}\\
t & \in & \{3, 4, 5, 6, 7\}
\end{array}$$

Now, to actually solve this whole mess, we need to rearrange our equation a little bit.

$$ \begin{array}{rcl}
44 & = & h_3 + (6 + n_3) t\\
44 - h_3 & = & (6 + n_3) t
\end{array}$$

With this equation, we can see that $$ 44 - h_3$$ must be divisible by both $$ (6 + n_3)$$ and $$ t$$. So, given that we have a list of candidates for $$ h_3$$, we can simply check their divisors and see if any of them are candidates for $$ t$$.

| $$h_3$$ | equation | $$\{x \in \mathbb{N} \vert x \mid (44 - h_3), t_{min} \leq x \leq t_{max}\}$$ |
|---------|----------|---------------------------------------------------------------------------|
| $$ 1$$  | $$ 43 = (6 + n_3) t$$ | $$ \{\}$$ |
| $$ 3$$  | $$ 41 = (6 + n_3) t$$ | $$ \{\}$$ |
| $$ 5$$  | $$ 39 = (6 + n_3) t$$ | $$ \{3\}$$ |
| $$ 6$$  | $$ 38 = (6 + n_3) t$$ | $$ \{\}$$ |
| $$ 7$$  | $$ 37 = (6 + n_3) t$$ | $$ \{\}$$ |

Seeing that only $$ h_3 = 5$$ produced a valid $$ t = 3$$, these must be our values. Now, all that is left to do is pop them right back into the equation to find $$ n_3$$.

$$ \begin{array}{rcl}
39 & = & (6 + n_3) 3\\
21 & = & 3n_3\\
7 & = & n_3
\end{array}$$

There we go, $$n_3 = 7$$. This means that in sector Gamma, there are five sharks and seven fish tanks. Every fish tank contains three fish.

Too bad that this could be solved on paper, I'm hoping that next time I can finally get an excuse to play around with a fancy constraint solver or implement something. But in the meantime, it was fun to do and I hope I didn't get anything wrong.
