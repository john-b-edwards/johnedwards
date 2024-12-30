---
date: "2024-12-30"
description: My write-up of the third week of Advent of Code problems
draft: false
keywords:
- julia
slug: aoc_2024_week_three
tags:
- julia
title: 2024 Advent of Code Week 3
toc: false
bsky_thread: 
---

## Introduction

The [Advent of Code](https://adventofcode.com/) (AOC) is a series of programming problems that are released daily from December 1st to December 25th, each problem more challenging than the last. As a means of practicing my Julia skills, I decided to try to tackle the AOC this year using just Julia! If you'd like to see my work in completing the first week of AOC problems, click [here](https://johnbedwards.io/blog/aoc_2024_week_one/), and if you'd like to see my second week, click [here](https://johnbedwards.io/blog/aoc_2024_week_two/). Things get real tough this week, so let's dive in!

## [Day Fifteen: Warehouse Woes](https://adventofcode.com/2024/day/15)

``` julia
# input
include("Utils.jl")
using .Utils
input = get_example(2024,15)
input_1 = [x for x in input if x[1] == '#']
input_1 = split.(input_1,"")
input_1 = mapreduce(permutedims, vcat, input_1)
input_1 = [string(i) for i in input_1]
```

```         
10×10 Matrix{String}:
 "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"
 "#"  "."  "."  "O"  "."  "."  "O"  "."  "O"  "#"
 "#"  "."  "."  "."  "."  "."  "."  "O"  "."  "#"
 "#"  "."  "O"  "O"  "."  "."  "O"  "."  "O"  "#"
 "#"  "."  "."  "O"  "@"  "."  "."  "O"  "."  "#"
 "#"  "O"  "#"  "."  "."  "O"  "."  "."  "."  "#"
 "#"  "O"  "."  "."  "O"  "."  "."  "O"  "."  "#"
 "#"  "."  "O"  "O"  "."  "O"  "."  "O"  "O"  "#"
 "#"  "."  "."  "."  "."  "O"  "."  "."  "."  "#"
 "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"
```

``` julia
input_1_stored = copy(input_1)
input_2 = split(join([x for x in input if x[1] != '#']),"")
```

```         
700-element Vector{SubString{String}}:
 "<"
 "v"
 "v"
 ⋮
 "<"
 "<"
 "^"
```

### Part One

The first part of our input represents the layout of a warehouse as seen from above. Boxes in the warehouse are represented with an `O`, walls represented with a `#`, and a robot for moving the boxes represented with a `@`.

The second part of our input represents a series of instructions passed to the robot. The robot will move around according to the instructions, moving any boxes it runs into, but it will skip instructions that would cause it to run into a wall (or cause boxes to run into walls).

Our task is to determine the position of all of the boxes after the robot follows all of the instructions. From there, we take the coordinates of each of the boxes and sum them together to find our answer.

My initial pass at part one involved moving slices of the array defined in part one around--from the position of the robot, I checked along the direction that the robot was supposed to move. For example, if the robot was supposed to move up, I checked the space directly above the robot. If it was an empty space, I moved the robot. If there was a wall, I did not move the robot. If there was a box, I continued moving up until I encountered either an empty space or a wall.

This approach turned out to not be scaleable for part two however (as you'll see shortly!) so I wound up re-writing it to form a breadth-first search--it accomplished the same task, but allowed me greater flexibility in solving part two.

``` julia
move_dict = Dict("^"=>CartesianIndex(-1,0),">"=>CartesianIndex(0,1),"v"=>CartesianIndex(1,0),"<"=>CartesianIndex(0,-1))
function process_move(mat, point, direction)
    to_move, to_check, checked = [point], [point + move_dict[direction]], []
    new_mat = copy(mat)
    while to_check != []
        v = pop!(to_check)
        if mat[v] == "#"
            return mat
        elseif (mat[v] == "]") & !in(v, checked)
            push!.([to_move], [v, v + CartesianIndex(0,-1)])
            push!.([to_check], [v, v + CartesianIndex(0,-1), v + move_dict[direction]])
        elseif (mat[v] == "[") & !in(v, checked)
            push!.([to_move], [v, v + CartesianIndex(0,1)])
            push!.([to_check], [v, v + CartesianIndex(0,1), v + move_dict[direction]])
        elseif (mat[v] != ".") & !in(v, checked)
            push!(to_move, v)
            push!(to_check, v + move_dict[direction])
        end
        push!(checked, v)
    end
    for p in unique(to_move)
        new_mat[p + move_dict[direction]] = mat[p]
        if !in(p - move_dict[direction], to_move) new_mat[p] = "." end
    end
    new_mat[point] = "."
    return new_mat
end
```

Then I wrote a quick function to calculate the "GPS score" from the position of the boxes.

``` julia
function calc_gps_score(mat)
    return sum([100 * (x[1]-1) + (x[2]-1) for x in findall(x->x in ["O","["], mat)])
end
```

From there, I went through and iterated over all of the second part of the input until the robot was done moving, and calculated the GPS score.

``` julia
robot = findfirst(x->x=="@", input_1)
for move in input_2
    global input_1 = process_move(input_1, robot, move)
    global robot = findfirst(x->x=="@", input_1)
end
println(calc_gps_score(input_1))
```

```         
10092
```

### Part Two

The second part was almost identical to the first, except now the boxes and walls of the warehouse were twice as large (though the robot was still the same size). The example map above from part one looked like this for part two:

```         
####################
##....[]....[]..[]##
##............[]..##
##..[][]....[]..[]##
##....[]@.....[]..##
##[]##....[]......##
##[]....[]....[]..##
##..[][]..[]..[][]##
##........[]......##
####################
```

You can see now the impact of the breadth-first search: rather than moving individual vectors, I would need to track the position of whole boxes that spanned multiple columns.

The first step I took was adjusting the first part of the input to match the problem as defined in part two:

``` julia
input_1 = copy(input_1_stored)
newmat = fill(" ", size(input_1)[1], size(input_1)[2] .* 2)
for i in 1:size(input_1)[2], j in 1:size(input_1)[1]
    if input_1[i,j] == "#" newmat[i,(2j-1):(2j)] = ["#","#"]
    elseif input_1[i,j] == "O" newmat[i,(2j-1):(2j)] = ["[","]"]
    elseif input_1[i,j] == "." newmat[i,(2j-1):(2j)] = [".","."]
    elseif input_1[i,j] == "@" newmat[i,(2j-1):(2j)] = ["@","."] end
end
```

Then, I simply needed to re-run my processing of the input with the new map.

``` julia
robot = findfirst(x->x=="@", newmat)
for move in input_2
    global newmat = process_move(newmat, robot, move)
    global robot = findfirst(x->x=="@", newmat)
end
println(calc_gps_score(newmat))
```

```         
9021
```

## [Day Sixteen: Reindeer Maze](https://adventofcode.com/2024/day/16)

``` julia
include("Utils.jl")
using .Utils
input = [string(i) for i in mapreduce(permutedims, vcat, split.(get_example(2024,16),""))]
```

```         
15×15 Matrix{String}:
 "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"
 "#"  "."  "."  "."  "."  "."  "."  "."  "#"  "."  "."  "."  "."  "E"  "#"
 "#"  "."  "#"  "."  "#"  "#"  "#"  "."  "#"  "."  "#"  "#"  "#"  "."  "#"
 "#"  "."  "."  "."  "."  "."  "#"  "."  "#"  "."  "."  "."  "#"  "."  "#"
 "#"  "."  "#"  "#"  "#"  "."  "#"  "#"  "#"  "#"  "#"  "."  "#"  "."  "#"
 "#"  "."  "#"  "."  "#"  "."  "."  "."  "."  "."  "."  "."  "#"  "."  "#"
 "#"  "."  "#"  "."  "#"  "#"  "#"  "#"  "#"  "."  "#"  "#"  "#"  "."  "#"
 "#"  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "."  "#"  "."  "#"
 "#"  "#"  "#"  "."  "#"  "."  "#"  "#"  "#"  "#"  "#"  "."  "#"  "."  "#"
 "#"  "."  "."  "."  "#"  "."  "."  "."  "."  "."  "#"  "."  "#"  "."  "#"
 "#"  "."  "#"  "."  "#"  "."  "#"  "#"  "#"  "."  "#"  "."  "#"  "."  "#"
 "#"  "."  "."  "."  "."  "."  "#"  "."  "."  "."  "#"  "."  "#"  "."  "#"
 "#"  "."  "#"  "#"  "#"  "."  "#"  "."  "#"  "."  "#"  "."  "#"  "."  "#"
 "#"  "S"  "."  "."  "#"  "."  "."  "."  "."  "."  "#"  "."  "."  "."  "#"
 "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"
```

### Part One

Our input represents a maze that reindeer are attempting to navigate. A `#` indicates a wall, `S` indicates the start of the maze, and `E` represents the exit from the maze.

Reindeer can take any path through the maze, but rotating clockwise or counter-clockwise 90 degrees will cost them 1000 points, and moving forward will cost them one point.

Our task is to find the lowest possible score for our input.

This turned out to be a job for Dijkstra's, annoying as it may have been. In previous examples, we have used Dijkstra's to find the shortest path through a maze, but it's important to remember that Dijkstra's doesn't neccesarily seek to minimize distance, rather, it seeks to minimize a cost function. This function canbe flat distance in some cases, but in this instance, we could use our own cost function (turns + movement) to find the lowest score path.

I began with building our dictionary of available paths, as well as our start and end nodes.

``` julia
directions = [(0,1),(-1,0),(0,-1),(1,0)]
prev = Dict()
start, target_node = (Tuple(findfirst(x->x==y, input)) for y in ["S","E"])
vertices = Dict((x,y) => Inf for x in Tuple.(findall(x->x!="#", input)), y in directions)
vertices[(start,(0,1))] = 0
unvisited = copy(vertices)
```

From there, I went through and calculated the *score* required to visit each node using Dijkstra's. I included scores for all possible directions that one could enter a node from.

``` julia
while length(unvisited) > 0
    current_node, current_direction = findmin(unvisited)[2]
    neighbors = [(current_node .+ j,j) for j in directions
        if (input[current_node[1] + j[1], current_node[2] + j[2]] != "#") & 
            in((current_node .+ j,j), keys(unvisited))
    ]
    for n in neighbors
        old_dist = vertices[n]
        if current_direction == n[2]
            new_dist = vertices[(current_node, current_direction)] + 1
        else
            new_dist = vertices[(current_node, current_direction)] + 1001
        end
        if old_dist >= new_dist
            vertices[n], unvisited[n] = new_dist, new_dist
            if n in keys(prev)
                push!(prev[n], (current_node, current_direction))
            else
                prev[n] = [(current_node, current_direction)]
            end
        end
    end
    delete!(unvisited, (current_node, current_direction))
end
```

From there, I found the lowest score (for any direction) to enter the end node from.

``` julia
println(minimum([vertices[(target_node,x)] for x in directions]))
```

```         
7036.0
```

### Part Two

For the second part, we want to watch the reindeer run the mazes. To see the reindeer, we want to find out how many different spaces along the map where we could sit and plausibly see a reindeer running the maze--to do this, we need to find *all possible paths* that have the lowest score possible.

To do this, I used reverse iteration to find all possible paths that satisfied the lowest score condiiton--note above that in our Dijkstra's algorithm, for each node we check, we store any nodes that led us to that node in a dictionary. To find all paths, we simply iterate over that dictionary and store all nodes in a list, then find the length of the set of that list.

``` julia
path = []
target_dir = directions[argmin([vertices[(target_node,x)] for x in directions])]
tocheck = [(target_node,target_dir)]
while tocheck != []
    u = pop!(tocheck)
    if in(u, keys(prev))
        push!(path, u)
        append!(tocheck, prev[u])
    end
end
println(length(unique([x[1] for x in path])) + 1)
```

```         
45
```

## [Day Seventeen: Chronospatial Computer](https://adventofcode.com/2024/day/17)

``` julia
include("Utils.jl")
using .Utils
input = get_example(2024,17)
```

```         
4-element Vector{Vector{String}}:
 ["Register", "A:", "729"]
 ["Register", "B:", "0"]
 ["Register", "C:", "0"]
 ["Program:", "0,1,5,4,3,0"]
```

``` julia
A = parse(Int64, input[1][3])
B = parse(Int64, input[2][3])
C = parse(Int64, input[3][3])
prog = parse.(Int64, split(input[4][2],","))
```

### Part One

This one was positively devilish! Here, we were provided with a computer of sorts. The computer knows eight instructions--a function that takes in an argument. The argument can refer to either one of the values stored in the register or it can refer to the value itself. Our input consists of initial values to the register as well as a "program" consisting of instructions for executing certain functions in order with these arguments.

The first part simply asked us to determine the output of the program. For this, I wrote a function that reflected literally the functions and modified the values stored in the register.

``` julia
function computer(A, prog)
    combo = Dict(0=>0,1=>1,2=>2,3=>3,4=>A,5=>B,6=>C)
    pointer = 0
    vals = []
    while true
        if pointer + 1 > length(prog) break end
        jump_pointer = true
        opcode,operand = prog[pointer+1:pointer+2]
        if opcode == 0 combo[4] = Int(floor(combo[4] / 2 ^ combo[operand]))
        elseif opcode == 1 combo[5] = combo[5] ⊻ operand
        elseif opcode == 2 combo[5] = combo[operand] % 8
        elseif (opcode == 3) && (combo[4] != 0)
            pointer = operand
            jump_pointer = false
        elseif opcode == 4 combo[5] = combo[5] ⊻ combo[6]
        elseif opcode == 5 append!(vals,combo[operand] % 8)
        elseif opcode == 6 combo[5] = Int(floor(combo[4] / 2 ^ combo[operand]))
        elseif opcode == 7 combo[6] = Int(floor(combo[4] / 2 ^ combo[operand])) end
        pointer = pointer + 2 * jump_pointer
    end
    return vals
end
println(computer(A, prog))
```

```         
Any[4, 6, 3, 5, 6, 3, 5, 2, 1, 0]
```

### Part Two

This is where the program got *exceptionally* tricky. We had to find a value for register "A" that would cause our program to return *itself* (something akin to a [Quine](https://en.wikipedia.org/wiki/Quine_(computing))).

I first tried brute forcing, starting with my initial value of A and going up. I quickly realized that it would be impossible. For my actual input, the program I had was 16 digits long, and for a given value of A of length n, the output would typically be around n digits long--so I would need to iterate through roughly all possible 16 digit numbers to brute force my value!

I tried inspecting my output closer, poking and prodding at how the program worked. I realized that for smaller digit numbers, I could get the output to replicate a small part of the program, and as I went up in terms of the number of digits for my input, I could produce more and more of the output. I also realized that the value stored in the `A` register was subject to some big changes in value--specifically, it was repeatedly divided by `2^3`, or `8`. Finally, I realized that the program would only terminate when the value of `A` was `0`. I set `A` to `1` and my input to `0`. From there, I added `1` to my input until I was able to replicate part of the program as my output. Once I did that, I multiplied by input by `8`, and repeated searching until I replicated a longer section of the program as my output. I did this until the entire output matched the program.

This wasn't my most elegant solution! It ended up working just fine for me and resolved quite quickly (much more quickly than a *raw* brute force search), but I didn't test my solution with other inputs. Still, my main takeaway was the importance of poking and prodding at the program to understand it, which was required for me to solve it.

``` julia
function find_quine(prog)
    combo = Dict(0=>0,1=>1,2=>2,3=>3,4=>A,5=>B,6=>C)
    adv_ndx = findfirst(x->(prog[x]==0) & (x % 2 == 1), eachindex(prog))
    multiplier = 2 ^ combo[prog[adv_ndx + 1]]
    counter, prog_counter = 0,0
    while prog_counter < length(prog)
        if computer(counter, prog) == prog break
        elseif computer(counter, prog) == prog[end-prog_counter:end]
            counter = counter * multiplier
            prog_counter = prog_counter + 1
        else counter = counter + 1 end
    end
    return counter
end
println(find_quine(prog))
# not shown as this is kind of a hard-coded solution!
```

## [Day Eighteen: RAM Run](https://adventofcode.com/2024/day/18)

``` julia
include("Utils.jl")
using .Utils
input = get_example(2024,18,override= "")
input_parsed = [CartesianIndex(parse(Int64, x[2]) + 1, parse(Int64,x[1]) + 1) 
                for x in split.(input,",")]
```

```         
25-element Vector{CartesianIndex{2}}:
 CartesianIndex(5, 6)
 CartesianIndex(3, 5)
 CartesianIndex(6, 5)
 ⋮
 CartesianIndex(6, 1)
 CartesianIndex(7, 2)
 CartesianIndex(1, 3)
```

### Part One

For this problem, we are in an `NxN` grid (`N=7` for the example, but for the actual input, `N=71`), beginning at the top-left corner of the grid. Our goal is to reach the bottom-right corner of the grid. Obstructing us, however, are a series of "bits" that sequentially fall into the grid and obstruct our path (effectively becoming walls). Each second, a new "bit" appears on our grid, its position indicated by our input.

For part one, we first need to simulate the first 1024 bits (in the example, 12 bits) falling into the grid, then find the length of the shortest path to escape.

This was fairly straightforward--we just needed to process the coordinates we were given into a grid-space, then run a Dijkstra's algorithm to find the shortest path.

``` julia
DIMS = 7
S = CartesianIndex(1,1)
E = CartesianIndex(DIMS, DIMS)
DIRECTIONS = CartesianIndex.([(0,1),(-1,0),(0,-1),(1,0)])
# part one
inbounds(G, v) = (v[1] > 0) & (v[1] <= size(G)[1]) & (v[2] > 0) & (v[2] <= size(G)[2])
function plot_input(input, n_bytes)
    mat = fill(".",DIMS, DIMS)
    for x in input[1:minimum([n_bytes,length(input)])] mat[x] = "#" end
    return mat
end
function dist_to_exit(input, n)
    mat = plot_input(input, n)
    vertices = Dict(x => Inf for x in findall(x->x!="#", mat))
    vertices[S] = 0
    unvisited = copy(vertices)
    while unvisited != Dict()
        u = findmin(unvisited)[2]
        delete!(unvisited, u)
        neighbors = [u + d for d in DIRECTIONS 
                     if inbounds(mat, u+d) && 
                     in(u+d, keys(unvisited)) & 
                     (mat[u + d] != "#")]
        for n in neighbors
            vertices[n] = minimum([1 + vertices[u], vertices[n]])
            unvisited[n] = minimum([1 + vertices[u], vertices[n]])
        end
    end
    return vertices[E]
end
println(Int(dist_to_exit(input_parsed, 12)))
```

```         
22
```

### Part Two

The second part was not much trickier. We are provided with well more than 1024 bits (or 12 bits for the example)\--our task is to determine at which point the bits completely block the exit and make escaping impossible, and find the position of the block that made our escape impossible.

For this, the function I wrote to simulate the bits falling into place as well as my Dijkstra's algorithm were fast enough that I could simply continue to simulate over the coordinates provided and check at which point the distance to the exit became `Inf`.

``` julia
global counter = 12
while !isinf(dist_to_exit(input_parsed, counter)) counter = counter + 1 end
println(input[counter])
```

```         
6,1
```

I was curious if there was a faster way to solve this problem (this technically counts as brute forcing, I think!) and put out a quiet call on Bluesky if anyone else had thoughts--and got a great response!

<blockquote class="bluesky-embed" data-bluesky-uri="at://did:plc:3xxpcqzw7hnm7cwgpmxcntkg/app.bsky.feed.post/3ldmvoz2bqc2e" data-bluesky-cid="bafyreicn7obsbn3pzaiwbgvcubl553lfdd6uturjekhgp7ugigw5t33t5e"><p lang="en">In Part 2 you can check if the next coordinate is on your current best path. If not, skip ahead. If so, recalculate the new best path, if any.</p>&mdash; Jeff Standen (<a href="https://bsky.app/profile/did:plc:3xxpcqzw7hnm7cwgpmxcntkg?ref_src=embed">@jstanden.bsky.social</a>) <a href="https://bsky.app/profile/did:plc:3xxpcqzw7hnm7cwgpmxcntkg/post/3ldmvoz2bqc2e?ref_src=embed">December 18, 2024 at 10:13 PM</a></blockquote>

<script async src="https://embed.bsky.app/static/embed.js" charset="utf-8"></script>

This would have required some significant re-factoring on my part that I opted out of, but I agree that it would be significantly faster!

## [Day Nineteen: Linen Layout](https://adventofcode.com/2024/day/19)

``` julia
include("Utils.jl")
using .Utils
input = get_example(2024,19)
```

```         
9-element Vector{Vector{String}}:
 ["r,", "wr,", "b,", "g,", "bwu,", "rb,", "gb,", "br"]
 ["brwrr"]
 ["bggr"]
 ["gbbr"]
 ["rrbgbr"]
 ["ubwu"]
 ["bwurrg"]
 ["brgr"]
 ["bbrgwb"]
```

### Part One

For this problem, we are provided with a series of towels with different striped patterns (the first line of our input). Each letter corresponds to a particular color (r for red, b for blue, etc.). Our task is to determine if a pattern provided in the second part of our input is possible with the towels we're provided. For example, `brwrr` is possible by lining up `br`, `wr`, and `r` towels, but there is no combination of towels that would allow us to create the lineup `ubwu`.

This required a recursive approach. After parsing the input, I wrote a function that took a string representing a sequence of towels and checked if any of the valid patterns from the first part of the input appeared in the start of the string. If the answer was yes, then I removed that from the start of the string and called the function again on the new string. If the answer was no, I returned `0`. If I passed in an empty string (indicating that each sequence could have been formed from the towels in the first part of the input), then I returned a value of 1. This function returned how many different ways one could combine the towels to form the desired pattern. I then ran this on the input and returned how many times this value was greater than one to find the number of possible combinations.

I sped up this function using [memoization](https://en.wikipedia.org/wiki/Memoization)--I used a dictionary to store all calls to the `is_valid()` function, so if I had previously called `is_valid()` with an exact argument before, rather than go through every pattern and do some `Regex`ing, I just returned what the dictionary had.

``` julia
patterns = [string(x) for x in split(join(input[1]),",")]
designs = [x[1] for x in input[2:end]]
stored = Dict()
function is_valid(s)
    if s in keys(stored) return stored[s] end
    ans = 0
    if s == "" ans = 1 end
    for p in patterns
        if occursin(Regex("^" * p), s)
            ans = ans + is_valid(replace(s, p=>"",count=1))
        end
    end
    stored[s] = ans
    return ans
end
println(sum(is_valid.(designs) .> 0))
```

```         
6
```

### Part Two

For the second part, we needed to determine how many possible ways we could construct the patterns using the towels. For this, I re-ran the function, but rather than count how many times the function returned a value greater than one, I simply returned the output of the function and summed those values.

``` julia
println(sum(is_valid.(designs)))
```

```         
16
```

## [Day Twenty: Race Condition](https://adventofcode.com/2024/day/20)

``` julia
# input
include("Utils.jl")
using .Utils
input = get_example(2024,20)
mat = [string(i) for i in mapreduce(permutedims, vcat, split.(input,""))]
```

```         
15×15 Matrix{String}:
 "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"
 "#"  "."  "."  "."  "#"  "."  "."  "."  "#"  "."  "."  "."  "."  "."  "#"
 "#"  "."  "#"  "."  "#"  "."  "#"  "."  "#"  "."  "#"  "#"  "#"  "."  "#"
 "#"  "S"  "#"  "."  "."  "."  "#"  "."  "#"  "."  "#"  "."  "."  "."  "#"
 "#"  "#"  "#"  "#"  "#"  "#"  "#"  "."  "#"  "."  "#"  "."  "#"  "#"  "#"
 "#"  "#"  "#"  "#"  "#"  "#"  "#"  "."  "#"  "."  "#"  "."  "."  "."  "#"
 "#"  "#"  "#"  "#"  "#"  "#"  "#"  "."  "#"  "."  "#"  "#"  "#"  "."  "#"
 "#"  "#"  "#"  "."  "."  "E"  "#"  "."  "."  "."  "#"  "."  "."  "."  "#"
 "#"  "#"  "#"  "."  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "."  "#"  "#"  "#"
 "#"  "."  "."  "."  "#"  "#"  "#"  "."  "."  "."  "#"  "."  "."  "."  "#"
 "#"  "."  "#"  "#"  "#"  "#"  "#"  "."  "#"  "."  "#"  "#"  "#"  "."  "#"
 "#"  "."  "#"  "."  "."  "."  "#"  "."  "#"  "."  "#"  "."  "."  "."  "#"
 "#"  "."  "#"  "."  "#"  "."  "#"  "."  "#"  "."  "#"  "."  "#"  "#"  "#"
 "#"  "."  "."  "."  "#"  "."  "."  "."  "#"  "."  "."  "."  "#"  "#"  "#"
 "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"  "#"
```

### Part One

This is another maze problem, where `#` represent walls, `S` represents the start of the maze, and `E` represents the end of the maze. However, we have a twist--we can cheat at the maze! Moving a space takes one picosecond, but we can enable a cheat that allows us to move into walls for up to two picoseconds. At the end of two picoseconds, we have to be back within the bounds of the maze. Our task is to identify how many cheats could save us at least 100 picoseconds.

I first started by running a simple Dijkstra's to determine the distance between the start and end of the maze, along with all of the nodes along the way.

``` julia
DIRECTIONS = CartesianIndex.([(0,1),(-1,0),(0,-1),(1,0)])
inbounds(G, v) = (v[1] > 0) & (v[1] <= size(G)[1]) & (v[2] > 0) & (v[2] <= size(G)[2])
calc_dist(x, y) = abs(y[1]-x[1]) + abs(y[2]-x[2])
S = findfirst(x->x=="S",mat)
E = findfirst(x->x=="E",mat)
vertices = Dict(x => Inf for x in findall(x->x!="#", mat))
vertices[S] = 0
unvisited = copy(vertices)
while unvisited != Dict()
    u = findmin(unvisited)[2]
    delete!(unvisited, u)
    neighbors = [u + d for d in DIRECTIONS 
                 if inbounds(mat, u+d) && 
                 in(u+d, keys(unvisited)) & 
                 (mat[u + d] != "#")]
    for n in neighbors
        vertices[n] = minimum([1 + vertices[u], vertices[n]])
        unvisited[n] = minimum([1 + vertices[u], vertices[n]])
    end
end
```

From there, I used a *second* Dijkstra's to identify any possible cheats by comparing the time taken to reach a node versus how much time it would take to reach the node with a possible cheat. For a given amount of time `n` that we're allowed to cheat for, the function returns a dictionary of every cheat and how much time it saves. From there, I simply `filter`ed to cheats that would save 100 or more seconds.

``` julia
function find_cheats(max_length)
    cheats = Dict()
    possible_cheat_starts, possible_cheat_ends = keys(vertices), keys(vertices)
    for c in possible_cheat_starts
        cheat_ends = [i for i in possible_cheat_ends if 
                      (calc_dist(i, c) <= max_length) &&
                      ((vertices[i] - vertices[c] - calc_dist(i,c)) >= calc_dist(i,c))]
        for d in cheat_ends
            cheats[(c,d)] = vertices[d] - vertices[c] - calc_dist(d,c)
        end
    end
    return cheats
end
my_cheats = find_cheats(2)
println(length(filter(((k,v),) -> v >= 100, my_cheats))) # doesn't work for the example maze!
```

### Part Two

The second example was identical to the first, only now, we could cheat for 20 seconds. The part one solution can be seamlessly modified to solve part two:

``` julia
my_cheats = find_cheats(20)
println(length(filter(((k,v),) -> v >= 100, my_cheats))) # again, doesn't really work for the example maze!
```

## [Day Twenty-One: Keypad Conundrum](https://adventofcode.com/2024/day/21)

``` julia
# input
include("Utils.jl")
using .Utils
input = get_example(2024,21)
```

```         
5-element Vector{String}:
 "029A"
 "980A"
 "179A"
 "456A"
 "379A"
```

### Part One

I **hated** this problem! We were given a series of inputs representing numbers to be input on a numerical keypad with this layout:

```         
+---+---+---+
| 7 | 8 | 9 |
+---+---+---+
| 4 | 5 | 6 |
+---+---+---+
| 1 | 2 | 3 |
+---+---+---+
    | 0 | A |
    +---+---+
```

However, we cannot input these numbers ourselves--we have to control a robot to do it for us, using a directional keypad with this layout:

```         
    +---+---+
    | ^ | A |
+---+---+---+
| < | v | > |
+---+---+---+
```

Starting on the numerical keypad at the `A` key, we can direct a pointer to move up, down, left or right, then press `A` on the directional keypad to cause the pointer to input the number it is hovering over.

However, we cannot even directly control this robot--we have to use this same keypad to control a robot *who in turn uses this keypad to control a second robot* to put the inputs on the numerical keypad. Our task is to determine the length of the shortest path to input a given numerical command, then multiply that by the numerical part of the input (so `780` in the first input above), then sum these values for each command.

This was a PITA, specifically because the problem stipulates that the pointer must remain in the bounds of the keypad: so in order to move from the `<` key to the `A` key, we cannot move `up` then `right`, but we ***must*** move `right` then `up`.

My initial solution involved generating all possible paths to visit each needed key in order at each level, then calculating the keys needed to input those keys at the level directly above it, and so on, growing out with each successive level. This was feasible for the first part of the problem, but infeasible for the second part. For that, I had to develop a consistent set of rules for manipulating the keypad to have both the shortest possible path *and* one that stayed within the boundaries of the path--testing for some annoying and hard to replicate edge cases. That became my function `find_paths()`, which determined the shortest possible optimal path from one part of a keypad to another part of a keypad.

``` julia
NUM_KEYPAD = ['7' '8' '9';'4' '5' '6';'1' '2' '3';' ' '0' 'A']
DIR_KEYPAD = [' ' '^' 'A';'<' 'v' '>']
DIRECTIONS = CartesianIndex.([(0,1),(-1,0),(0,-1),(1,0)])
inbounds(G, v) = (v[1] > 0) & (v[1] <= size(G)[1]) & (v[2] > 0) & (v[2] <= size(G)[2])
# part one
function find_paths(d1, d2, pad)
    d1_ndx, d2_ndx = findfirst(x->x==d1,pad),findfirst(x->x==d2,pad)
    dist = d2_ndx - d1_ndx
    x, y = dist[2], dist[1]
    horz = cat(repeat('>',abs(x) * (x > 0)),repeat('<',abs(x) * (x < 0)),dims = 1)
    vert = cat(repeat('^',abs(y) * (y < 0)),repeat('v',abs(y) * (y > 0)),dims = 1)
    if (x > 0) & (pad[CartesianIndex(d2_ndx[1],d1_ndx[2])] != ' ')
        return [join(vert) * join(horz) * "A"]
    elseif pad[CartesianIndex(d1_ndx[1],d2_ndx[2])] != ' '
        return [join(horz) * join(vert) * "A"]
    else
        return [join(vert) * join(horz) * "A"]
    end
end
```

Next, I needed to find the path required to visit all parts of a given input. I used memoization again to keep my memory down (a necessity for part two).

``` julia
function tally_paths(path, pad)
    routes = Dict()
    start = 'A'
    for x in path
        for y in find_paths(start, x, pad)
            routes[y] = get(routes, y, 0) + 1
        end
        start = x
    end
    return routes
end
```

Finally, I wrote a function that found the length of a path for a given number of robots operating on all levels and multiplied it by the numerical value of the input.

``` julia
function find_complexity(key, robots = 3)
    paths = tally_paths(key,NUM_KEYPAD)
    for n in 1:(robots-1)
        new_routes = Dict()
        for (x, y) in paths
            for (w, z) in tally_paths(x, DIR_KEYPAD)
                new_routes[w] = get(new_routes, w, 0) + z * y
            end
        end
        paths = new_routes
    end
    return parse(Int64,join(filter(!isnothing, tryparse.(Int64, split(key,""))))) * 
        sum([length(x) * y for (x,y) in paths])
end
println(sum(find_complexity.(input, [3])))
```

```         
126384
```

### Part Two

The second part of this problem was almost the same as the first, the only difference being that instead of needing to direct three robots in sequence, we now needed to direct **twenty six** robots in sequence! Fortunately the memoization helped a good bit for this solution:

``` julia
println(sum(find_complexity.(input, [26])))
```

```         
154115708116294
```

## Looking ahead

That's three weeks of the Advent of Code down! I've actually [already finished the AOC](https://bsky.app/profile/johnbedwards.io/post/3leg3fp3kbc2o), so I know how difficult the final week will be (spoiler alert--really tough!) but I'm in the process of writing it up now. I've been really wearing our Dijkstra's here, and noticing that I'm using a lot of the same functions and definitions--things I could probably have baked into my `Utils.jl` module. For example, there are a lot of mazes, so some kind of generic 2D Dijkstra's function would probably be useful. I'd also consider it useful to store the `inbounds()` function I wrote in the module, along with the `DIRECTIONS` dictionary I use a lot of. Lessons to be learned for next year I suppose!
