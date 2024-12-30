---
date: "2024-12-30"
description: My write-up of the final week of Advent of Code problems
draft: true
keywords:
- julia
slug: aoc_2024_week_four
tags:
- julia
title: 2024 Advent of Code Week 4
toc: false
bsky_thread: 
---

## Introduction

The [Advent of Code](https://adventofcode.com/) (AOC) is a series of programming problems that are released daily from December 1st to December 25th, each problem more challenging than the last. As a means of practicing my Julia skills, I decided to try to tackle the AOC this year using just Julia! You can view my solutions to each week at this links: [Week One](https://johnbedwards.io/blog/aoc_2024_week_one/), [Week Two](https://johnbedwards.io/blog/aoc_2024_week_two/), and [Week Three](https://johnbedwards.io/blog/aoc_2024_week_three/). Let's finish strong with Week Four!

## [Day Twenty-Two: Monkey Market](https://adventofcode.com/2024/day/22)

``` julia
include("Utils.jl")
using .Utils
input = get_example(2024,22)
```

```         
4-element Vector{Int64}:
    1
   10
  100
 2024
```

### Part One

For this problem, we are at a market where we are selling bananas to monkeys. Each monkey has a secret number generated via a pseudorandom process. From their starting secret number (the puzzle input), the number is run through a function. Each iteration of the function on the secret number produces a new secret number. Our task for part one is to run through this process with each monkey and sum their secret numbers following 2000 iterations.

This was quite simple to put together--I simply defined the pseudorandom rules in Julia, put it into a function, and ran the function on each secret number 2000 times.

``` julia
function sim_secret(val, n)
    for i in 1:n
        val = ((val * 64) ⊻ val) % 16777216
        val = (Int(floor(val / 32)) ⊻ val) % 16777216
        val = ((val * 2048) ⊻ val) % 16777216
    end
    return val
end
println(sum(sim_secret.(input, [2000])))
```

```         
37327623
```

### Part Two

The second part was considerably more tricky. The price that each buyer purchases bananas at is the one digit of their secret number at a given iteration. We are employing a monkey to sell our bounty of bananas--however, the monkey only knows how to sell when it encounters a particular sequence of four consecutive changes in price from the buyer's side. So if a buyer had the following pseudorandom sequence:

```         
     123: 3 
15887950: 0 (-3)
16495136: 6 (6)
  527345: 5 (-1)
  704524: 4 (-1)
 1553684: 4 (0)
12683156: 6 (2)
11100544: 4 (-2)
12249484: 4 (0)
 7753432: 2 (-2)
```

The best price to sell at (6 bananas) occurs only after the buyer's price changes by `-1, -1, 0, 2`. We need to find the four digit sequence that would generate the most money from selling bananas to *all of the buyers in our input*, and return how much money we would make.

My solution turned out slower and less optimized than I liked, but it still ran in ~5 minutes on my dinky work laptop, so I didn't feel **too** bad about it. For each monkey, I simmed their secret number and stored all price change sequences in a dictionary. When I encountered a particular price sequence for the first time for a given monkey, I added the ones digit of their secret number to the value already stored for that sequence in the dictionary. From there, I found the maximum value stored in the dictionary.

``` julia
changes = Dict()
for secret in input
    all_sequences, tmp_changes = [], []
    for j in 1:2000
        current_price = digits(secret)[1]
        secret = sim_secret(secret, 1)
        new_price = digits(secret)[1]
        change = new_price - current_price
        if j >= 4
            push!(tmp_changes, change)
            if !in(tmp_changes,all_sequences) && tmp_changes in keys(changes) 
                changes[tmp_changes] = changes[tmp_changes] + new_price
            elseif !in(tmp_changes,all_sequences)
                changes[tmp_changes] = new_price
            end
            push!(all_sequences, tmp_changes)
            tmp_changes = tmp_changes[2:end]
        else push!(tmp_changes, change) end
    end
end
```

```         
24
```

## [Day Twenty-Three: LAN Party](https://adventofcode.com/2024/day/23)

``` julia
include("Utils.jl")
using .Utils
using Graphs
input = get_example(2024,23)
input = split.(input,"-")
```

```         
32-element Vector{Vector{SubString{String}}}:
 ["kh", "tc"]
 ["qp", "kh"]
 ["de", "cg"]
 ⋮
 ["wh", "qp"]
 ["tb", "vc"]
 ["td", "yn"]
```

### Part One

We are provided with a list of computer connections--each two-letter string is a computer, and each pairing indicates that one computer is connected to another. For the first part of the problem, we need to find the number of connections that involve three computers (e.g. there is some connection from `"aa"` to `"bb"` and there is some connection from `"bb"` to `"cc"` and there is some connection from `"cc"` to `"aa"`) and contain a computer whose name begins with a `t`.

First, I went through and parsed the input down into a couple useful pieces of data, including all computers listed in the input.

``` julia
input_x = [i[1] for i in input]
input_y = [i[2] for i in input]
computers = unique(collect(Base.Iterators.flatten(input)))
lookup = Dict(computers[x] => x for x in eachindex(computers))
```

From there, I relied on the `Graphs.jl` package, which came in super handy. I was able to construct a simple, undirected graph consisting of all the computers in the input and their connections.

``` julia
g = SimpleGraph(length(computers),0)
map((i) -> add_edge!(g, lookup[input_x[i]], 
                        lookup[input_y[i]]),
                     eachindex(input))
```

From there, I thought it would be arbitrary to simply use the `triangles()` function from `Graphs.jl` to grab these values. Unfortunately, `triangles(g, v)` returns the **number** of triangles that `v` is a part of, but it doesn't tell us what these triangles are. So in the example input, there is a triangle `tc,td,wh` that contains two potential values that start with the letter `t`. Calling `triangles` on each node in the graph that starts with the letter `t` will double count this specific triangle, and I couldn't figure out a way to filter it out.

I would up instead just looking at the immediate neighbors of each point that started with the letter `t`, then checking the immediate neighbors of those points to see if they were connected. I did this as a `Set()` to handle triangles with multiple `t` computers, as illustrated above.

``` julia
ts = [lookup[x] for x in computers[occursin.(r"^t", computers)]]
println(length(unique(vcat(vcat(
    [[[Set([t, n, x]) 
        for x in intersect(neighbors(g, t), neighbors(g, n))] 
        for n in neighbors(g, t)] 
        for t in ts]...)...)
)))
```

```         
7
```

### Part Two

The second part was actually *easier* than the first thanks to the `Graphs.jl` package, as if such a thing were possible this late in the Advent of Code. Our task was to find the largest set of computers that were all connected to each other--our solution is all of these computers arranged in alphabetical order.

The problem is asking for the [maximal clique](https://en.wikipedia.org/wiki/Clique_(graph_theory)) of the graph formed by these computers, and fortunately, there's a function in `Graphs.jl` that finds just that easy-peasy!

``` julia
replace(join(sort(
    computers[argmax(length, maximal_cliques(g))]
) .* ","),r",$"=>"")
```

```         
"co,de,ka,ta"
```

## [Day Twenty-Four: Crossed Wires](https://adventofcode.com/2024/day/24)

``` julia
# input
include("Utils.jl")
using .Utils
input = get_example(2024,24)
```

```         
9-element Vector{Vector{String}}:
 ["x00:", "1"]
 ["x01:", "1"]
 ["x02:", "1"]
 ["y00:", "0"]
 ["y01:", "1"]
 ["y02:", "0"]
 ["x00", "AND", "y00", "->", "z00"]
 ["x01", "XOR", "y01", "->", "z01"]
 ["x02", "OR", "y02", "->", "z02"]
```

### Part One

Here, we are provided with a series of inputs (`x00,y00` and so on), then a series of logical operations conducted on these inputs that produce output values `z00`. Though not shown in the example, the assignments in the input will use and produce intermediate values (like `ntg XOR fgs -> mjb`).

Our task for part one was to first uncover what the output of these values produces, by converting the outcomes of `zXX` from binary to base-10 (where `z00` reflects the `2^0`th place in the binary representation of the figure, `z01` represents the `2^1`th place, and so on).

For this, I wrote a function to simply parse the input and store these values in a dictionary. In the event that an operation referenced an intermediate value that wasn't defined already defined, I recursively searched for the line that defined it and executed that line. From there, I parsed the output of all of the z-values into a base-10 number.

``` julia
input_1 = [x for x in input if length(x) == 2]
initial_keys = Dict(replace(i[1],":"=>"") => parse(Int64, i[2]) for i in input_1)
input_2 = [x for x in input if length(x) != 2]
all_ins = collect(Base.Iterators.flatten([(x[1], x[3], x[5]) for x in input_2]))
z_sorted = [i for i in input_2 if occursin(r"^z",i[5])][sortperm([i[5] for i in input_2 if occursin(r"^z",i[5])],rev=true)]
# part one
function parse_input_2(i)
    if !in(i[1], keys(values))
        if i[1] in keys(initial_keys)
            values[i[1]] = initial_keys[i[1]]
        else
            new_line = [j for j in input_2 if j[5] == i[1]][1]
            values[i[1]] = parse_input_2(new_line)
        end
    end
    if !in(i[3], keys(values))
        if i[3] in keys(initial_keys)
            values[i[3]] = initial_keys[i[3]]
        else
            new_line = [j for j in input_2 if j[5] == i[3]][1]
            values[i[3]] = parse_input_2(new_line)
        end
    end
    if i[2] == "AND"
        values[i[5]] = values[i[1]] & values[i[3]]
    elseif i[2] == "OR"
        values[i[5]] = values[i[1]] | values[i[3]]
    elseif i[2] == "XOR"
        values[i[5]] = values[i[1]] ⊻ values[i[3]]
    end
end
values = Dict()
println(parse(Int64, join(parse_input_2.(z_sorted)), base=2)) # this doesn't work well with the example, sorry!
```

### Part Two

The second part revealed that this program was designed to add two numbers together--much as the `zXX` numbers were the binary representation of some base-10 number, so did `xXX` and `yXX`. The program was supposed to add `xXX` and `yXX` together to produce `zXX`, but some of the output values had been swapped around--so I needed to find out which where the swapped values, and return those as my answer in alphabetical order.

This was really tricky! There was no possible way to go through all of the potential swaps (the puzzle revealed that eight outputs had been swapped, yielding a number of potential combinations far greater than my computer could reasonably handle). After digging around online (and getting spoiled mildly on social media\...) I learned that this was supposed to represent a [full adder](https://en.wikipedia.org/wiki/Adder_(electronics)) which is used in electronics to add numbers together using logic gates.

Knowing the schema of the adder, I was able to write some rules to check the inputs and outputs of the gates to determine if they were valid or not. From there, I didn't need to identify which gates had been swapped with which--since we just needed to return the swapped gates in alphabetical order, all that was needed was to identify that the gates were wrong.

``` julia
my_vec = ['x','y','z']
println(join(sort(unique([i[5] for i in input_2 if 
    (
        (i[2] == "XOR") & 
        !in(i[1][1],my_vec) & 
        !in(i[3][1],my_vec) & 
        !in(i[5][1],my_vec)
    ) |
    (
        (i[2] == "XOR") && 
            length([j for j in input_2 if (i[5] in [j[1],j[3]]) & 
                    (j[2] == "OR")]) > 0
    ) |
    (
        (i[2] == "AND") & 
        !in("x00",[i[1],i[3]]) && 
            length([j for j in input_2 if (i[5] in [j[1],j[3]]) & 
                    (j[2] != "OR")]) > 0
    ) |
    (
        (i[5][1] == 'z') & 
        (i[2] != "XOR") & 
        (i[5] != "z45")
    )
])) .* ",")[1:end-1]) # again, doesn't really work for the example!
```

## [Day Twenty-Five: Code Chronicle](https://adventofcode.com/2024/day/25)

``` julia
include("Utils.jl")
using .Utils
input = get_example(2024,25)
input = [string(i) for i in mapreduce(permutedims, vcat, split.(input,""))]
```

```         
35×5 Matrix{String}:
 "#"  "#"  "#"  "#"  "#"
 "."  "#"  "#"  "#"  "#"
 "."  "#"  "#"  "#"  "#"
 "."  "#"  "#"  "#"  "#"
 "."  "#"  "."  "#"  "."
 "."  "#"  "."  "."  "."
 "."  "."  "."  "."  "."
 "#"  "#"  "#"  "#"  "#"
 "#"  "#"  "."  "#"  "#"
 "."  "#"  "."  "#"  "#"
 "."  "."  "."  "#"  "#"
 "."  "."  "."  "#"  "."
 "."  "."  "."  "#"  "."
 ⋮
 "#"  "."  "#"  "."  "."
 "#"  "#"  "#"  "."  "."
 "#"  "#"  "#"  "."  "#"
 "#"  "#"  "#"  "."  "#"
 "#"  "#"  "#"  "#"  "#"
 "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."
 "#"  "."  "."  "."  "."
 "#"  "."  "#"  "."  "."
 "#"  "."  "#"  "."  "#"
 "#"  "#"  "#"  "#"  "#"
```

### Part One

Much to my relief, the 25th day of the Advent of Code was short and sweet! The input consists of schematics with lock (a slice of the input matrix that starts with a row of all `#` and ends with a row of all `.`) and keys (starts with `.` and ends with `#`). So a key may look like this:

```         
#####
.####
.####
.####
.#.#.
.#...
.....
```

and a lock may look like this:

```         
.....
#....
#....
#...#
#.#.#
#.###
#####
```

A lock can only fit into a key if none of the `#` values overlap when the matrix of the key is overlaid on the map of the lock. We have to determine the number of unique pairs of locks and keys that would fit each other.

To solve this, I first split up my input into locks and keys:

``` julia
keys_and_locks = [input[i:i+6,:] for i in 1:7:size(input)[1]]
my_keys = [x for x in keys_and_locks if all(x[1:1,1:5] .== "#")]
my_locks = [x for x in keys_and_locks if all(x[7:7,1:5] .== "#")]
```

Then, I wrote a function that would test if a key fit into a given lock, by calculating how many `#` symbols were in each column, and added those together for each lock and key. If any values summed to more than five, the combination was invalid!

``` julia
function test_fit(my_key, my_lock)
    key_heights = [sum(my_key[2:7,i] .== "#") for i in 1:5]
    lock_heights = [sum(my_lock[1:6,i] .== "#") for i in 1:5]
    return all((key_heights .+ lock_heights) .<= 5)
end
```

Finally, I wrote a nested for loop to test all of the locks and keys. And I got my answer!

``` julia
counter = 0
for my_key in my_keys
    for my_lock in my_locks
        if test_fit(my_key, my_lock)
            global counter = counter + 1
        end
    end
end
println(counter)
```

```         
3
```

### Part Two

There was no part two for this problem!

## Closing thoughts

I had a wonderful time doing the Advent of Code this year (as opposed to 2022, when I became quite sick in early December and realized I was skipping out on desperately needed sleep just to try to score meaningless leaderboard points\...). I really struggled with some of these problems, but struggling with them was good--I learned a lot in the process (and I certainly learned more than the folks who just fed the problems into an LLM repeatedly until they got an answer)! I'm not sure I'll do it again next year--just having gone all the way through a full AoC was quite taxing, but thank you all for following along! Happy holidays!
