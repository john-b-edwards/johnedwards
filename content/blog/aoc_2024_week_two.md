---
date: "2024-12-14"
description: My write-up of the second week of Advent of Code problems
draft: false
keywords:
- julia
slug: aoc_2024_week_two
tags:
- julia
title: 2024 Advent of Code Week 2
toc: false
bsky_thread: 
---

## Introduction

The [Advent of Code](https://adventofcode.com/) (AOC) is a series of programming problems that are released daily from December 1st to December 25th, each problem more challenging than the last. As a means of practicing my Julia skills, I decided to try to tackle the AOC this year using just Julia! If you'd like to see my work in completing the first week of AOC problems, click [here](https://johnbedwards.io/blog/aoc_2024_week_one/). Let's dive in!

## [Day Eight: Resonant Collinearity](https://adventofcode.com/2024/day/8)

``` julia
# input
include("Utils.jl")
using .Utils
input = get_example(2024,8)
input = split.(input,"")
input = mapreduce(permutedims, vcat, input)
```

```         
12×12 Matrix{SubString{String}}:
 "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "."  "0"  "."  "."  "."
 "."  "."  "."  "."  "."  "0"  "."  "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "0"  "."  "."  "."  "."
 "."  "."  "."  "."  "0"  "."  "."  "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "A"  "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "."  "A"  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "."  "."  "A"  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."
```

### Part One

The premise of part one is that there are a series of antennae broadcasting signals--each non-dot character reflects an antennae, and each character corresponds to the frequency broadcast by each antenna. For every pair of antennae broadcasting on the same frequency, there are two "antinodes" created that are along the straight line running through the antennae, a distance from each antenna corresponding to how far apart the two antennae are, in the opposite direction. See the example below, where `a` is an antenna broadcasting along the `a` frequency, and `#` is the location of an antinode. Note that antinodes can overlap with other antinodes.

```         
..........
...#......
..........
....a.....
..........
.....a....
..........
......#...
..........
..........
```

The task is to find how many unique locations within the bounds of the map contain antinodes.

For this, I wrote a quick helper function to test if a value is inbounds of a matrix. I should probably add this to `Utils.jl`, it seems like I use this logic a lot.

``` julia
inbounds_test(node, rows, cols) = (node[1] <= rows) & (node[1] > 0) & (node[2] <= cols) & (node[2] > 0)
```

I then grabbed all the different frequencies from the input matrix.

``` julia
chars = [string(char) for char in unique(Iterators.flatten(input)) if char != '.']
```

Next, I made an empty string matrix the same size as the input matrix for tracking antinode locations.

``` julia
rows,cols = size(input)
refmat = fill(".",(rows,cols))
```

Finally, I iterated over all antennae operating with the same frequency--for each antenna, I iterated over all other antenna on that frequency, calculated, and plotted an antinode. After that, I just found how many spots had at least one antinode to find my solution.

``` julia
for char in chars
    for pair in Iterators.product(findall(x->x == char, input),findall(x->x == char, input))
        if pair[1] != pair[2]
            antinode_one, antinode_two = pair[1] + (pair[1] - pair[2]), pair[2] + (pair[2] - pair[1])
            if inbounds_test(antinode_one, rows, cols) refmat[antinode_one] = "#" end
            if inbounds_test(antinode_two, rows, cols) refmat[antinode_two] = "#" end
        end
    end
end
println(length(findall(x->x == "#", refmat)))
```

```         
14
```

### Part Two

The next part revealed that antinodes actually existed at any point exactly in line with any two antennae, extending across the matrix. In the example below, `T` represents antennae operating along the `T` frequency, and `#` represents the new location of antinodes. Of note, antennae themselves *are now also considered antinodes*.

```         
T....#....
...T......
.T....#...
.........#
..#.......
..........
...#......
..........
....#.....
..........
```

Our task was to again find all unique antinode locations.

This required a slight modification to the loop we used before--rather than plot one antinode from each antenna pair, we needed to plot all these resonant antinodes. To do this, I created a "vector" that reflected the spatial difference between two antennae, then from each antenna, I updated the grid to reflect an antinode. From that new antinode, I treated that as an antenna, and moved along the same vector, updating the value again, until we reached the boundaries of the map.

``` julia
refmat = fill(".",(rows,cols))
for char in chars
    for pair in Iterators.product(findall(x->x == char, input),findall(x->x == char, input))
        if pair[1] != pair[2]
            antinode_one, antinode_two = pair[1] + (pair[1] - pair[2]), pair[2] + (pair[2] - pair[1])
            while inbounds_test(antinode_one, rows, cols) 
                refmat[antinode_one] = "#" 
                antinode_one = antinode_one + (pair[1] - pair[2])
            end
            while inbounds_test(antinode_two, rows, cols) 
                refmat[antinode_two] = "#" 
                antinode_two = antinode_two + (pair[2] - pair[1])
            end
        end
    end
end
println(length(unique(cat(findall(x->x != ".", refmat),findall(x->x != ".",input),dims=1))))
```

```         
34
```

## [Day Nine: Disk Fragmenter](https://adventofcode.com/2024/day/9)

``` julia
# input
include("Utils.jl")
using .Utils
input = get_example(2024,9)
input = split(string(input[1]),"")
```

```         
19-element Vector{SubString{String}}:
 "2"
 "3"
 "3"
 "3"
 "1"
 "3"
 "3"
 "1"
 "2"
 "1"
 "4"
 "1"
 "4"
 "1"
 "3"
 "1"
 "4"
 "0"
 "2"
```

We're provided with a dense file format which reflects where files are stored on a disk and how much free space is on the disk. The first number in the file format reflects the space allocated to file id `0`. The second number reflects the size of empty space stored after file id `0`. Then the third number reflects the space allocated to file id `1`, then the fourth reflects empty space stored after `1`, and so on. For disk map `12345`, the file space allocated to each file and the empty space available can be represented as:

```         
0..111....22222
```

Where `0` is the ID of the first file (which takes up one block), `1` is the ID of the second file (which takes up three blocks), and `2` is the id of the third file (which takes up five blocks).

Our task is to move files one block at a time from the right to the left to fill in the empty spaces, such that there are no remaining gaps between files. For the example above, this process would look like:

```         
0..111....22222
02.111....2222.
022111....222..
0221112...22...
02211122..2....
022111222......
```

From there, we need to calculate the filesystem checksum, which involves multiplying the position of each block (0 indexed) by its file ID number and then summing those values.

### Part One

I first converted my input to the full-sized representation of the file format, then found the location of all of the empty and full spaces.

``` julia
my_vec = collect(Base.Iterators.flatten([ifelse(i % 2 == 1,
    collect(Base.Iterators.repeated(floor(i / 2), parse(Int64, input[i]))),
    repeat(["."], parse(Int64, input[i])))
    for i in eachindex(input)
]))
empty = findall(x -> x == ".",my_vec)
full = findall(x -> x!= ".",my_vec)
```

From there, I reversed the original vector and took a slice equivalent in length to all of the empty spaces, then reversed that. I then assigned that slice to all of the empty spaces, then sliced from left-to-right the first occurrence through the length of the all full values, then finally calculated the checksum.

``` julia
my_vec[empty] = reverse(my_vec[full])[1:length(empty)]
my_vec = my_vec[1:length(full)]
println(Int(calc_checksum(my_vec)))
```

```         
1928
```

### Part Two

``` julia
my_vec = collect(Base.Iterators.flatten([ifelse(i % 2 == 1,
    collect(Base.Iterators.repeated(floor(i / 2), parse(Int64, input[i]))),
    repeat(["."], parse(Int64, input[i])))
    for i in eachindex(input)
]))
```

For the second part, our challenge remained much the same, only now--we needed to move the files in *blocks* (and only moving them if there was adequate space). So for this example:

```         
1..22..3..444
```

We would be unable to move the `444` file, because there is not empty space to fill it anywhere. However, we could move the `3` file, because there is empty space between `1` and `22`.

```         
13.22.....444
```

To solve part two, I first grabbed all unique file ids.

``` julia
file_types = reverse([file for file in unique(my_vec) if (file != ".") & (file != 0)])
```

From there, I iterated over each file type in reverse (so going from right to left), and grabbed the location and size of each file block. Then, I iterated left-to-right, trying to find an adequately-sized file space to hold the moving file. If such a space existed, I reallocated the file to the space. Finally, I recalculated the checksum.

This took a little bit of time thanks to all the allocations I was making--using the [`view` functionality in Julia](https://docs.julialang.org/en/v1/base/arrays/#Views-(SubArrays-and-other-view-types)) helped speed things up a good bit. Still, this solution could be improved a lot.

``` julia
for file_type in file_types
    file_loc = findall(x->x == file_type,my_vec)
    n_blocks = length(file_loc)
    j = findfirst(x->x == collect(Base.Iterators.repeated(".",n_blocks)),
        ((@view my_vec[i:i+n_blocks-1]) for i in 1:1:file_loc[1]))
    if !isnothing(j)
        my_vec[j:(j + n_blocks-1)] = my_vec[file_loc]
        my_vec[file_loc] .= "."
    end
end
println(Int(calc_checksum(my_vec)))
```

```         
2858
```

## [Day 10: Hoof It](https://adventofcode.com/2024/day/10)

``` julia
include("Utils.jl")
using .Utils
input = get_example(2024,10,false,override="89010123
78121874
87430965
96549874
45678903
32019012
01329801
10456732")
input = split.(input,"")
input = [parse.(Int64, x) for x in input]
input = mapreduce(permutedims, vcat, input)
```

```         
8×8 Matrix{Int64}:
 8  9  0  1  0  1  2  3
 7  8  1  2  1  8  7  4
 8  7  4  3  0  9  6  5
 9  6  5  4  9  8  7  4
 4  5  6  7  8  9  0  3
 3  2  0  1  9  0  1  2
 0  1  3  2  9  8  0  1
 1  0  4  5  6  7  3  2
```

### Part One

Day 10 provided us a map with a list of levels. Starting at the "trailhead" `0`, we can only proceed to a new square if moving to a square with a number that is exactly one more than our originating square, and our hiking path terminates when we reach a "summit" (a value of `9`.

Each trailhead has a score corresponding to the number of summits it can reach. Our task is to find the sum of the scores of all trailheads.

To begin, I found the locations of all trailheads.

``` julia
trailheads = findall(x->x == 0, input)
```

After that, I wrote a function to perform a [depth-first search](https://en.wikipedia.org/wiki/Depth-first_search) through the matrix. The depth-first search provides all feasible paths to reach a summit, so to calculate the score for part one, I provided an argument to filter to just unique summits, rather than unique paths. Finally, I broadcast this function to each trailhead and summed the returned number of unique trailheads.

``` julia
function find_path(trailhead, input, unique_paths)
    found, queue = [], [trailhead]
    while queue != []
        v = pop!(queue)
        if input[v] == 9
            push!(found, v)
        else
            for i in -1:1, j in -1:1
                if (abs(i) != abs(j)) && 
                    (v[1] + i > 0) & 
                    (v[1] + i <= size(input)[1]) & 
                    (v[2] + j > 0) & 
                    (v[2] + j <= size(input)[2]) && 
                    (input[v[1] + i, v[2] + j] == input[v] + 1)
                    push!(queue, CartesianIndex(v[1] + i, v[2] + j))
                end
            end
        end
    end
    return ifelse(unique_paths, length(unique(found)), length(found))
end
println(sum(find_path.(trailheads, [input], [true])))
```

```         
36
```

### Part Two

The wrinkle for part two was that we needed to find the number of distinct feasible trails-to-summits that originated from each trailhead (as multiple paths to the same summit are possible).

This proved a cinch, as all I needed to do was simply return the number of distinct paths rather than the number of distinct summits.

``` julia
println(sum(find_path.(trailheads, [input], [false])))
```

```         
81
```

## [Day 11: Plutonian Pebbles](https://adventofcode.com/2024/day/11)

``` julia
include("Utils.jl")
using .Utils
input = [125, 17]
```

```         
2-element Vector{Int64}:
 125
  17
```

### Part One

For this problem, we are presented with a series of strange, magic stones, each engraved with a number. While staring at the stones, every time we blink, the stones change according to the following rules:

-   If the stone is engraved with the number 0, the stone is replaced with a stone engraved with the number 1.

-   If the stone is engraved with a number with an even number of digits, the stone is replaced with two stones, the first stone with the first half of digits, the second stone with the second half of digits. So a stone engraved with 101001 becomes two stones, one engraved with 101 and the other engraved with 1.

-   If none of the above rules apply, the number on the stone is multiplied by 2024. So a stone engraved with 2 is replaced by a stone engraved with 4048.

Our task is to find how many stones we have after 25 blinks.

This was a tricky problem, deliberately designed to force users to rely on efficient memory storage. Writing a `for` loop and keeping track of all of the stones in a vector would result in the vector and operations involving the vector becoming too unwieldy after many iterations.

The trick here came from keeping track not of individual vector elements, rather, keeping track of the numbers engraved on the stone. For example, five stones each engraved with `2` after one step will follow identical cycles in the above rules, so rather than storing them in a vector like this: `[2 2 2 2 2]`, it was more efficient to store them in a dictionary like this: `Dict(2 => 5)`.

I first wrote a function that applies the rules of the loop to a given number.

``` julia
function process_loop(x)
    dig = length(digits(x))
    if x == 0
        return [1]
    elseif dig % 2 == 0
        return [Int(floor(x / 10 ^ (dig/2))), Int(x - floor(x / 10 ^ (dig/2)) * 10 ^ (dig/2))]
    else
        return [x * 2024]
    end
end
```

Then, I wrote a function that would take in an input an apply the looping rules n-times, keeping track of the stones as value =\> count pairs in a dictionary, and updating the counts with the rules, rather than individual elements. At the conclusion of blinking a number of times, I return the total number of counts we tracked in the dictionary.

``` julia
function my_loop(input, times)
    loop_counts = Dict(i => sum(input .== i) for i in unique(input))
    for n in 1:times
        new_dict = Dict()
        for x in keys(loop_counts)
            count = loop_counts[x]
            vals = process_loop(x)
            for val in vals
                new_dict[val] = get(new_dict, val, 0) + count
            end
        end
        loop_counts = new_dict
    end
    return sum([loop_counts[x] for x in keys(loop_counts)])
end
println(my_loop(input, 25))
```

```         
55312
```

### Part Two

The second part was identical to the first, only we needed to find the number of stones after 75 blinks. This rendered essentially any other memory allocation scheme other than the above impossible (which I learned the hard way)!

``` julia
println(my_loop(input, 75))
```

```         
65601038650482
```

## [Day 12: Garden Groups](https://adventofcode.com/2024/day/12)

``` julia
include("Utils.jl")
using .Utils
input = get_example(2024,12)
input = split.(input,"")
input = mapreduce(permutedims, vcat, input)
input = [string(x) for x in input]
```

```         
4×4 Matrix{String}:
 "A"  "A"  "A"  "A"
 "B"  "B"  "C"  "D"
 "B"  "B"  "C"  "C"
 "E"  "E"  "E"  "C"
```

### Part One

This problem involves plots of vegetables--each vegetable represented by a letter like `A` and stored in a grid. Each plot of connected vegetables needs a fence around it, and the cost of the fence is equivalent to the number of fence piences needed to surround a fence times the number of plants within the fence. In the example input, the fences around each group of plants look like this:

```         
+-+-+-+-+
|A A A A|
+-+-+-+-+     +-+
              |D|
+-+-+   +-+   +-+
|B B|   |C|
+   +   + +-+
|B B|   |C C|
+-+-+   +-+ +
          |C|
+-+-+-+   +-+
|E E E|
+-+-+-+
```

So `A` has 10 fence pieces and contains 4 plants, so the cost of the fence around `A` is 40. `E` has 8 fence pieces and contains 3 plants, so its fence costs 24. For the first part, we need to calculate the cost of fencing for all plants.

For the first part, I went through each space and, with a [breadth-first search](https://en.wikipedia.org/wiki/Breadth-first_search), identified each unique grouping of plants and the locations of each.

``` julia
inbounds(G, v) = (v[1] > 0) & (v[1] <= size(G)[1]) & (v[2] > 0) & (v[2] <= size(G)[2])
function find_all_in_group(G, root)
    Q = [root]
    explored = [root]
    valid = [root]
    while Q != []
        v = pop!(Q)
        for (i,j) in [(-1,0),(1,0),(0,-1,),(0,1)]
            if inbounds(G, (v[1] + i,v[2] + j)) && 
                G[CartesianIndex(v[1] + i, v[2] + j)] == G[root] && 
                !in(CartesianIndex(v[1] + i, v[2] + j), explored)
                push!.([Q,valid,explored], [CartesianIndex(v[1] + i, v[2] + j)])
            end
        end
    end
    return valid
end
groups = unique(collect(Iterators.flatten([[sort(find_all_in_group(input, CartesianIndex(i,j))) 
for i in 1:size(input)[1]] for j in 1:size(input)[2]])))
```

From there, I needed to calculate the number of fence units needed to surround each square. I did this with some simple logic--for each plant within the group, if any of the adjacent squares to that plant (up, down, left, and right) were *not* in the group, then the plant required a fence unit.

``` julia
function calc_value_1(group)
    return sum([(4 - sum(in.(CartesianIndex.(
            [(g[1] + 1, g[2]),(g[1] - 1, g[2]),(g[1], g[2] + 1),(g[1], g[2] - 1)]
        ),[group]))) for g in group]) * length(group)
end
println(sum(calc_value_1.(groups)))
```

```         
140
```

### Part Two

For the second part, rather than determine the price based on the number of fence units, we just needed to determine the price based on the total number of **sides** needed to fence in each unit. So for the example above, `A` has just 4 sides, but `C` has 8 sides, so the cost of `A` is 16 and the cost of `C` is 32.

This problem got much easier when I stopped looking for sides and started looking for corners--a basic rule of geometry is that a closed shape must have as many corners as it does sides. This here looks quite messy, but at a high level, it creates a 2x2 window and iterates over the matrix, looking for instances where just one element of the group exists within the window (which must be a corner, by definition) or instances where two elements of the group exist diagonal to each other with the other two elements are not a part of the group (which is *two* corners).

``` julia
function calc_value_2(group)
    return length(group) * 
    sum(sum([[
        ifelse(sum(in.(CartesianIndex.([(i,j),(i,j+1),(i+1,j),(i+1,j+1)]), [group])) in [1,3],1,
        ifelse(((CartesianIndex(i,j) in group) & (CartesianIndex(i+1,j+1) in group) & !(CartesianIndex(i,j+1) in group) & !(CartesianIndex(i+1,j) in group)) | 
        ((CartesianIndex(i,j+1) in group) & (CartesianIndex(i+1,j) in group) & !(CartesianIndex(i+1,j+1) in group) & !(CartesianIndex(i,j) in group)),2,0)) 
        for i in 1:size(new_input)[1]-1] 
            for j in 1:size(new_input)[2]-1]))
end
sum(calc_value_2.(groups))
```

```         
80
```

## [Day 13: Claw Contraption](https://adventofcode.com/2024/day/13)

``` julia
include("Utils.jl")
using .Utils
input = get_example(2024,13)
input = [Dict("button_a" => (parse(Int64,input[x-2][3][2:end-1]),parse(Int64,input[x-2][4][2:end])),
              "button_b" => (parse(Int64,input[x-1][3][2:end-1]),parse(Int64,input[x-1][4][2:end])),
              "prize" => (parse(Int64,input[x][2][3:end-1]),parse(Int64,input[x][3][3:end]))) 
        for x in 3:3:length(input)]
```

```         
4-element Vector{Dict{String, Tuple{Int64, Int64}}}:
 Dict("button_a" => (94, 34), "button_b" => (22, 67), "prize" => (8400, 5400))
 Dict("button_a" => (26, 66), "button_b" => (67, 21), "prize" => (12748, 12176))
 Dict("button_a" => (17, 86), "button_b" => (84, 37), "prize" => (7870, 6450))
 Dict("button_a" => (69, 23), "button_b" => (27, 71), "prize" => (18641, 10279))
```

This problem involves a series of claw machines. Each machine has two buttons, `A` and `B`. Pushing button `A` costs 3 tokens, pushing button `B` costs 1 token. For the first claw machine above, pushing button `A` will move the claw from the `x,y` coordinates of `0,0` to `94,34`, and pushing it again will move the claw unit another 94 units over and another 34 units up to `188,68`. Pushing button `B` will move the claw 22 units over and 67 units up. Eventually, you want to combine `A` pushes and `B` pushes to reach the location of the `prize`, `8400,5400`, exactly.

Not every machine is feasible--for the first machine, pushing button `A` 80 times and pushing button `B` 40 times gets you to exactly `8400,5400`, but there is not combination of pushes for the second machine that will allow you to reach `12748,12176`.

For part one, we need to find the minimum number of tokens to spend to get all feasible prizes from every machine, given that we can only push `A` and `B` at most 100 times each.

I recognized this as a [Mixed-Integer Linear Programming](https://www.mathworks.com/help/optim/ug/mixed-integer-linear-programming-algorithms.html) problem. Beyond simply solving each matrix for a solution, we specifically need to find `3 * button A presses + 1 * button B presses`, and `A` and `B` presses can only be integers. Additionally, because we need to know the *fewest* tokens we can spend, we have a cost function that we must seek to optimize. Suppose the `prize` was at `90,90` and `A` moved `10,10` and `B` moved `9,9`. We could press button `A` nine times as a feasible solution, but because it costs 3 times as much to push button `A` as it does `B`, it's cheaper to push button `B` 10 times (27 tokens for pushing `A` compared to 10 tokens for pushing `B`).

I opted to use the [`JuMP.jl` library](https://jump.dev/JuMP.jl/stable/) for specifying this problem, and the [`COPT.jl` library](https://github.com/COPT-Public/COPT.jl) for optimization. I chose the `COPT` optimizer because it handles large values for MILP well.

I defined `A` and `B` as integers, bounded by 0 (inclusive) and an upper bound `100` (inclusive). I defined the objective as minimizing `3A + B`, then constrained `A` times the x-movement of `A` plus `B` times the x-movement of `B` to be equal to the x-coordinate of the `prize` (then added the same constraint in the y-dimension as well).

After that, I asked the optimizer to find a solution. If no solution could be found, then I returned `0`, otherwise I returned `3A + B`.

``` julia
using JuMP
using COPT
function solve_machine(machine, upper_bound = 100, value_offset = 0)
    model = Model(COPT.Optimizer)
    if !ismissing(upper_bound)
        @variable(model, upper_bound >= A >= 0, Int)
        @variable(model, upper_bound >= B >= 0, Int)
    else
        @variable(model, A >= 0, Int)
        @variable(model, B >= 0, Int)
    end
    @objective(model, Min, 3A + B)
    @constraint(model, A * machine["button_a"][1] + 
                       B * machine["button_b"][1] == BigInt(machine["prize"][1] + value_offset))
    @constraint(model, A * machine["button_a"][2] + 
                       B * machine["button_b"][2] == BigInt(machine["prize"][2] + value_offset))
    optimize!(model);
    if !is_solved_and_feasible(model) 
        return 0
    else 
        return 3 * Int(value(A)) + Int(value(B))
    end
end
println(Int(sum(solve_machine.(input))))
```

```         
480
```

### Part Two

The second part threw a monkey wrench--the locations of all of the prizes were off by `10000000000000` (so the new prize location in the first machine above was **actually** `10000000008400,10000000005400`) though the buttons still only moved you a small amount. There was, however, no longer a 100 button-press limit.

I modified the above function (whose modifications are already reflected above) to handle both the removal of the upper boundaries of button pushing and the new offset. `COPT.jl` was able to handle the large constraints easily, though other optimizers like `HiGHS.jl` struggled to keep the values as actual integers.

``` julia
println(Int(sum(solve_machine.(input, missing, 10000000000000))))
```

```         
875318608908
```

## [Day 14: Restroom Redoubt](https://adventofcode.com/2024/day/14)

``` julia
include("Utils.jl")
using .Utils
input = get_example(2024,14)
dims = (7,11)
input = [Dict("p" => reverse(parse.(Int64,split(split(i[1],"=")[2],","))),
              "v" => reverse(parse.(Int64,split(split(i[2],"=")[2],","))))
        for i in input]
```

```         
12-element Vector{Dict{String, Vector{Int64}}}:
 Dict("v" => [-3, 3], "p" => [4, 0])
 Dict("v" => [-3, -1], "p" => [3, 6])
 Dict("v" => [2, -1], "p" => [3, 10])
 Dict("v" => [-1, 2], "p" => [0, 2])
 Dict("v" => [3, 1], "p" => [0, 0])
 Dict("v" => [-2, -2], "p" => [0, 3])
 Dict("v" => [-3, -1], "p" => [6, 7])
 Dict("v" => [-2, -1], "p" => [0, 3])
 Dict("v" => [3, 2], "p" => [3, 9])
 Dict("v" => [2, -1], "p" => [3, 7])
 Dict("v" => [-3, 2], "p" => [4, 2])
 Dict("v" => [-3, -3], "p" => [5, 9])
```

This problem concerns a series of robots cleaning a bathroom. Each robot (represented above as unique dictionaries) has a starting position `p` and a velocity `v` that reflects how many tiles it moves in the bathroom each second. The bathroom has some dimensions (in this example, 11 columns and 7 rows)--if a robot were to travel into a wall, it would teleport to the other side of the bathroom. Stealing another example from the website, this robot with `p=2,4 and v=2,-3` travels through the top wall in the first two seconds, and emerges from the bottom.

```         
Initial state:
...........
...........
...........
...........
..1........
...........
...........

After 1 second:
...........
....1......
...........
...........
...........
...........
...........

After 2 seconds:
...........
...........
...........
...........
...........
......1....
...........
```

For the first part of the problem, we need to find how many robots are in each quadrant (the upper-left tiles, the upper-right tiles, the bottom-left tiles, and the bottom-right tiles, excluding any robots in exactly the middle row or middle column), and multiply those four numbers together.

For this problem, I created a function that would find the coordinates of a robot after a given number of seconds and plotted it onto our grid. I was able to handle the robots teleporting through walls using modular division. Then, I split up the matrix into quadrants and found the number of robots in each.

``` julia
using Statistics
function plot_grid(dims, steps, input)
    grid = zeros(dims...)
    for robot in input
        new_cords = [(1 + robot["p"][x] + robot["v"][x] * steps) % dims[x] for x in 1:2]
        if (new_cords[1] < 1) new_cords[1] = dims[1] + new_cords[1] end
        if (new_cords[2] < 1) new_cords[2] = dims[2] + new_cords[2] end
        grid[new_cords...] = grid[new_cords...] + 1
    end
    return grid
end
p1_grid = plot_grid(dims, 100, input)
println(Int(
    sum(p1_grid[Int.(1:median(1:dims[1])-1), Int.(1:median(1:dims[2])-1)]) *
    sum(p1_grid[Int.(median(1:dims[1])+1:dims[1]), Int.(1:median(1:dims[2])-1)]) *
    sum(p1_grid[Int.(1:median(1:dims[1])-1), Int.(median(1:dims[2])+1:dims[2])]) *
    sum(p1_grid[Int.(median(1:dims[1])+1:dims[1]), Int.(median(1:dims[2])+1:dims[2])])
))
```

```         
12
```

### Part Two

Part two was quite cryptic--it suggested that after a number of seconds, the robots would align themselves into the picture of a Christmas tree on the tile grid.

I took a wild guess (which turned out to be correct!) that this would only be possible if all of the robots were each on their own tile, with no robots overlapping. I simply iterated with the grid until I reached a point where all robots had their own space.

``` julia
input = get_input(2024,14) # we need the input, not the example here!
dims = (103,101)
global counter = 1
while any(plot_grid(dims, counter, input) .> 1)
    global counter = counter + 1
end
println(counter)
```

```         
7709
```

## Looking ahead

This concludes two weeks of Advent of Code problems! I have been moving through them a little more slowly thanks to work + life + travel than I was able to in week one, but I'm feeling good being all caught up. I'm excited to tackle the next week (and hopefully reclaim my top spot on my work leaderboard\... coming for you Jeff)!
