---
layout: post
title: "AlMundo Challenge - ECI 2017"
date: 2017-08-05
categories: challenge
tags: [tsp, integer-programming, challenge]
image:
---

Last week I participated in the competition [AlMundo Challenge](https://almundo.com.ar/eci/contest)
organized by [almundo.com](https://almundo.com.ar/) in the context of [ECI2017](https://www.dc.uba.ar/events/eci/2017).

I found the problem interesting and, although the model I worked with came up in the 4th place (within the [ranking](https://almundo.com.ar/eci/ranking)), I would like to share it as an example of an **integer programming** application.

# The problem

The complete description can be found [here](https://almundo.com.ar/eci/contest).

In a nutshell, you are given the location of 41011 cities, and as a tourist you must fly to **all** the cities, visiting each of them **exactly once**. In addition, in each flight he must decide which travel agency to use. There are 4 travel agencies and each of them offers discounts if certain conditions are met. For example, one of the agencies grants a 35% discount in case of flying with them for 3 consecutive times.

The problem's objective is to decide *in which order* should the cities be visited and *which agency to use* in each flight, seeking to minimize the total cost (all agencies have the same rate: $0.01 per km).

A part of this problem (*city order*) is related to another very famous problem: [The traveling salesman problem](https://en.wikipedia.org/wiki/Travelling_salesman_problem), which is well-known and which has been studied by several people (see [TSP site](http://www.math.uwaterloo.ca/tsp/index.html)).

# My approach

One of my favorite tools for tackling optimization problems is applying [**integer programming**](https://en.wikipedia.org/wiki/Integer_programming). To simplify, the idea is to construct a mathematical model that describes the conditions imposed by the problem and the objective function to be minimized. As its name implies, it is important that the relationships between the model variables are **linear**.

I decided to split the original problem into two parts:

  1. Get an initial tour without taking into account the agencies, and therefore without discounts (the tour should visit all cities exactly once and be as cheap as possible).

  2. Look for **the best** possible way of assigning agencies to a given route.

Note that this is a *heuristic* since the optimal solution to the original problem is not necessarily obtained by assigning agencies to the shortest route, and often using a higher-cost route could induce greater discounts (given the conditions of the agencies), and consequently a lower total cost.

## Stage I: the tour

To get an initial tour (leaving aside the agencies), first I thought about using [Concorde](https://en.wikipedia.org/wiki/Concorde_TSP_Solver). This program allows you to get the optimal route as long as the number of cities is *reasonable*. I could not make Concorde work for 41011 cities, so I tried with the well-known [Lin-Kernighan heuristic](https://en.wikipedia.org/wiki/Lin%E2%80%93Kernighan_heuristic), that is used [here](http://www.math.uwaterloo.ca/tsp/concorde/downloads/downloads.htm).

The best route I got at this stage was **7754930.68** km.

## Stage II: assigment
### Model

If we consider the route of the previous stage and assign to all trips agency type D, we obtain a total cost of **$65924.31** that serves as a baseline.

Now we want to get the **best** way to allocate agencies. For this purpose, I constructed a model with the variables $$ X_ {a, e} $$ where $$ a \in \{A, B, C, D \} $$ and $$ e $$ is an edge of the tour (the trip between two cities).

If we define:

$$X_{a,e} = \begin{cases} 1 & \mbox{si } a\mbox{ is assigned to edge } e \\ 0 & \mbox{other case} \end{cases}$$

it is possible to model the constraint ```every edge must have an agency assigned``` as:

$$X_{A,e} + X_{B,e} + X_{C,e} + X_{D,e} = 1$$

for each edge $$ e $$ in the route.

It is also necessary to model the cost function that we want to minimize. ```Agency B applies a discount of 15% provided that the assigned edge is more than 200km long```. Taking this into account, we can model the discount of agency B as:

$$ \sum_{e\in T} X_{B,e} * w_e * 0.15 * (w_e > 200) * 0.01 $$

where $$ T $$ is the tour, and $$ w_e $$ is the distance in kilometers of the edge $$ e \in T $$.

On the other hand, ```agency C applies a discount of 20% provided that the previous trip is assigned to agency B```. To model this discount, one possibility is to incorporate the following variables:

$$Y_{e_i,e_{i+1}} = \begin{cases} 1 & \mbox{si } X_{B,e_i} = 1 \wedge X_{C,e_{i+1}} = 1 \\ 0 & \mbox{other case} \end{cases}$$

for each pair of consecutive edges $$(e_i, e_{i + 1})$$ in the tour.

The motivation of this variable is to be able to know when there is an assigned agency B before agency C, and thus to be able to apply the corresponding discount. The discount of agency C is then modeled as:

$$ \sum_{1\leq i \leq 41010} Y_{e_i,e_{i+1}} * w_{e_{i+1}} * 0.20 * 0.01 $$

In addition, it is necessary to linearly establish the relationship between the variables $$Y_{e_i,e_{i+1}}$$, $$X_{B,e_i}$$ and $$X_{C,e_{i+1}}$$ to meet the definition above:

$$Y_{e_i,e_{i+1}} \leq X_{B,e_i}$$

$$Y_{e_i,e_{i+1}} \leq X_{C,e_{i+1}}$$

$$Y_{e_i,e_{i+1}} + 1 \geq X_{B,e_i} + X_{C,e_{i+1}}$$

for each $$i$$ such that $$ 1 \leq i \leq 41010 $$.

To model the frequent flyer discount, where ```agency A applies a 35% discount every time the agency is used for the third consecutive time```, we use a similar idea. We add the variable:

$$Y_{e_i,e_{i+1},_{i+2}} = \begin{cases} 1 & \mbox{si } X_{A,e_i} = 1 \wedge X_{A,e_{i+1}} = 1 \wedge X_{A,e_{i+2}} = 1 \\ 0 & \mbox{other case} \end{cases}$$

for each $$ i $$ such that $$ 1 \leq i \leq 41009 $$.

The motivation is to know that there are 3 consecutive assignments of agency A and that it is worth applying the discount. The detail in this case is that if you have for example, 8 consecutive times agency A, the discount applies only in the cost of the third and sixth time, as clarified in the [FAQ](https://almundo.com.ar/eci/faq). For this reason, we also add the following constraint:

$$Y_{e_i,e_{i+1},e_{i+2}} + Y_{e_{i+1},e_{i+2},e_{i+3}} + Y_{e_{i+2},e_{i+3},e_{i+4}} \leq 1$$

for each $$ i $$ such that $$ 1 \leq i \leq 41007 $$.

And also an adapted version of the previous restrictions:

$$Y_{e_i,e_{i+1},e_{i+2}} \leq X_{A,e_i}$$

$$Y_{e_i,e_{i+1},e_{i+2}} \leq X_{A,e_{i+1}}$$

$$Y_{e_i,e_{i+1},e_{i+2}} \leq X_{A,e_{i+2}}$$

$$Y_{e_{i-2},e_{i-1},e_i} + Y_{e_{i-1},e_i,e_{i+1}} + Y_{e_i,e_{i+1},e_{i+2}} + 2 \geq X_{A,e_i} + X_{A,e_{i+1}} + X_{A,e_{i+2}}$$

with the corresponding adjustment in the first 2 trips of the tour, where $$Y_{e_{i-2},e_{i-1},e_i}$$ and/or $$Y_{e_{i-1},e_i,e_{i+1}}$$ must not be added.

We can then model the discount as:

$$ \sum_{1\leq i \leq 41009} Y_{e_i,e_{i+1},e_{i+2}} * w_{e_{i+2}} * 0.35 * 0.01 $$

The last detail is about the discount of accumulating kilometers, where ```agency D returns a fixed amount of $15 per 10000km traveled with this agency```. We define $$Y_D \in \mathbb{Z}_{\geq 0}$$ as:

$$Y_D = \left\lfloor \frac{1}{10000} * \sum_{e \in T} X_{D,e} * w_e \right\rfloor$$

as the amount of times we discount $15. To linearize $$ Y_D $$ we add the following constraints:

$$\frac{1}{10000} * \sum_{e \in T} X_{D,e} * w_e \leq Y_D + (1-\epsilon)$$

$$Y_D \leq \frac{1}{10000} * \sum_{e \in T} X_{D,e} * w_e$$

and we model the discount simply as:

$$Y_D * 15$$

Interestingly, since the objective function seeks to maximize the discount, the first restriction on $$ Y_D $$ will not be necessary and the optimal solution will be such that the restriction is met.

Finally, the objective function to be maximized is:

$$discount =$$

$$\sum_{e\in T} X_{B,e} * w_e * 0.15 * (w_e > 200) * 0.01$$

$$+ \sum_{1\leq i \leq 41010} Y_{e_i,e_{i+1}} * w_{e_{i+1}} * 0.20 * 0.01$$

$$+ \sum_{1\leq i \leq 41009} Y_{e_i,e_{i+1},e_{i+2}} * w_{e_{i+2}} * 0.35 * 0.01$$

$$+ Y_D * 15$$

or equivalently, we want to minimize:

$$\sum_{e\in T} w_e * 0.01 - discount$$

### Solving the model

To solve the model there are many alternatives, each with different advantages and disadvantages. During the competition I considered:

  * [PuLP](https://pypi.python.org/pypi/PuLP), a Python library that allows you to code the model and call general-purpose solvers.

  * [CPLEX](https://en.wikipedia.org/wiki/CPLEX), one of the best (if not the best) general purpose solvers.

I used PuLP to quickly code the model and CPLEX to solve it.

The optimal assignment over the aforementioned tour has a cost of **$63828.5** (that is, we improved the solution by $2095.81).

# Notes

I got my best final solution by incorporating a little local search on the previous solution trying to improve it. I managed to get down to **$63815.82** only by swapping the order of consecutive cities.

An interesting observation is that it is possible to model the original (**complete**) problem using integer programming. With only a few hours left to finish the competition, a friend who was playing (*fedepousa*) shared me the model he had built (since he was going on vacation and could not spend more time in the competition). This model was exact for the complete problem, but it is unthinkable to run it on the whole instance. I took his model and incorporated it as a local search operator, applying it in *chunks* of my best route. It is an expensive operator in terms of time, and applying it in sub-routes of 20 cities, every 20 cities, took a little less than 2 hours. Although the **optimum** in each chunk is being solved, there is a risk of deteriorating the total solution since the operator has no overall visibility of the route. A final combination of this operator, the simple swaps, and the optimal allocation that I presented before, was the one that got me in fourth place with a tour of **$63801.03**. Thanks *fedepousa*!

Finally, I would like to mention that he amount of time I was able to spend during the few days in which the competition took place, was limited. I am sure there are infinite improvements to the model I ended up with. Moreover, I think it would be valuable to experiment with more local search operators (and in particular with the exact model for the whole problem).

# Literature
If you liked the way I tried to the problem, feel curious about integer programming or the traveling salesman problem, I recommend reading:

* [Discrete-optimization](https://www.coursera.org/learn/discrete-optimization), an excellent course on Discrete Optimization (and integer programming in particular).
* Cook, W. (2012). In pursuit of the traveling salesman: mathematics at the limits of computation. Princeton University Press.
* Applegate, D. L., Bixby, R. E., Chvatal, V., & Cook, W. J. (2011). The traveling salesman problem: a computational study. Princeton university press.
* Wolsey, L. A. (1998). Integer programming. Wiley.
* Williams, H. P. (2013). Model building in mathematical programming. John Wiley & Sons. (thanks Isa for sharing!)
