---
date: "2024-12-07"
description: My write-up of the first week of Advent of Code problems
draft: false
keywords:
- julia
slug: aoc_2024_week_one
tags:
- julia
title: 2024 Advent of Code Week 1
toc: false
bsky_thread: https://bsky.app/profile/johnbedwards.io/post/3lcr3aqnnhc2y
---

## Introduction

The [Advent of Code](https://adventofcode.com/) (AOC) is a series of programming problems that are released daily from December 1st to December 25th, each problem more challenging than the last. As a means of practicing my Julia skills, I decided to try to tackle the AOC this year using just Julia! This is my write-up of the first week of AOC problems that I completed--as these problems progressively get more difficult, I anticipate splitting these posts into chunks that cover fewer days but are more comprehensive.

## Data input

A sneaky tricky problem for those doing AOC for the first time is, "How do I read in the inputs?" Typically, the AOC website provides both a toy example and a solution to the example in addition to a large input text file, which much be parsed to find a solution.

To make this a little easier on myself, I put together a [module](https://github.com/john-b-edwards/aoc-2024/blob/main/Utils.jl) to grab the example and the input. Pulling in the example was a little tricky, I had to do some intelligent parsing of the problem description to figure out where the example might live. Pulling in the actual input was much easier, as it lives as a raw .txt file at an easy to query URL (adventofcode.com/{year}/day/{day}/input).

I used the `HTTP.jl` package to query each resource, then used some string splitting utilities to parse inputs. I return the inputs as a nested vector, where each element of the first-level vector is each line of the input, and the second-level vector is each element in each line. This makes it pretty easy to iterate through both structured inputs (say, the input represents a matrix of numbers or characters) or unstructured inputs (say, the input represents a series of instructions of various lengths). I also tried to include some logic to do sensible type parsing (so if all of the values *could* be converted to integers, the module will perform this conversion automatically).

## [Day One: Historian Hysteria](https://adventofcode.com/2024/day/1)

``` julia
# input
include("Utils.jl")
using .Utils
input = get_example(2024,1)
```

```         
6-element Vector{Vector{Int64}}:
 [3, 4]
 [4, 3]
 [2, 5]
 [1, 3]
 [3, 9]
 [3, 3]
```

### Part One

For part one, I had to treat all of left numbers as one list, and all of the right numbers as a separate list. From there, I needed to pair the smallest number on the left with the smallest number on the right, then second-smallest in each each list, and so on, then find and sum the differences between each pair.

This was fairly straightforward--sorting both lists allowed me to broadcast the subtraction operator across both lists. From there, I just needed to sum up the resulting vector. No sweat!

``` julia
lhs = [i[1] for i in input]
rhs = [i[2] for i in input]
sort!(lhs)
sort!(rhs)
println(sum(abs.(lhs .- rhs)))
```

```         
11
```

### Part Two

The second part was a little trickier--I needed to find the "similarity score" for all of the elements in the left list, defined as "how often each element in the left list appears in the right list, times the value of each element in the left list". So 3 appears three times in the right list, so for the first element in the left list, the similarity score is 9. I then needed to take all of those values and sum them.

Again, broadcasting made this arbitrarily easily--I wrote a list comprehension to go over all the elements in the left-handed list which then summed up all of the corresponding elements in the right handed list. From there, I summarized the similarity scores. Oneliner!

``` julia
println(sum([i * sum(i .== rhs) for i in lhs])) 
```

```         
31
```

## [Day Two: Red-Nosed Reports](https://adventofcode.com/2024/day/2)

``` julia
input = get_example(2024,2)
```

```         
6-element Vector{Vector{Int64}}:
 [7, 6, 4, 2, 1]
 [1, 2, 7, 8, 9]
 [9, 7, 6, 2, 1]
 [1, 3, 2, 4, 5]
 [8, 6, 4, 4, 1]
 [1, 3, 6, 7, 9]
```

### Part One

The input represents a series of reports (each row), and within each report are a series of levels (each value in each row) that correspond to measurements from a power plant. Going from left to right within each report, if all levels are all increasing or all decreasing AND each level is at least one level different and no more than three levels from the previous level, the report is considered safe. The task is to find out how many reports are safe.

To handle this, I wrote a function with several logical tests to see if a report is safe, and then broadcasted that function across all the reports. H/t to [John on Bluesky](https://bsky.app/profile/arbitrandomuser.bsky.social/post/3lcdmvrc4sc2d) who pointed out that Julia has a built-in `diff()` function that accomplishes the same as the list comprehension I originally wrote, `[my_vec[j] - my_vec[j-1] for j in 2:length(my_vec)]`.

``` julia
test(diffs) = (all(diffs .> 0) | all(diffs .< 0)) & all(abs.(diffs) .!= 0) & all(abs.(diffs) .< 4)
println(sum(test.(diff.(input))))
```

```         
2
```

### Part Two

The second part introduced a wrinkle to the first--if a report could be made safe by removing at most one level from the report, then we could consider the report safe.

I wrote a second function that wrapped the first test function, and checked if the line was safe as is. If not, then I went through each level in each report and removed it, then re-tested. If any of the tests came back as safe, then I returned that value as safe as well.

``` julia
function parse_two(line)
    safe = test(diff(line))
    if !safe
        new_diffs = [diff(my_new_vec) for my_new_vec in [line[1:end .!= k] for k in eachindex(line)]]
        safe = any(test.(new_diffs))
    end
    return safe
end
println(sum(parse_two.(input)))
```

```         
4
```

## [Day Three: Mull It Over](https://adventofcode.com/2024/day/3)

``` julia
input = replace(join(get_example(2024,3)),r"(<em>)|(</em>)"=>"")
```

```         
"xmul(2,4)%&mul[3,7]!@^do_not_mul(5,5)+mul(32,64]then(mul(11,8)mul(8,5))"
```

### Part One

The first part gave us a long string with a series of instructions. If we found a value that looked like `mul(X,Y)`, we needed to multiply `X` by `Y`, then return the sum of those values. Anything that wasn't formatted like `mul(X,Y)` was to be thrown out.

This was a good chance for me to stretch my regex muscles in Julia, something I haven't used very frequently. I used `eachmatch()` to create an iterator over any matches, then parsed and multiplied `X` and `Y` within each match, then finally summed the values.

``` julia
function parse_input(input)
    return sum([prod([parse(Int64,y.match) for y in eachmatch(r"[0-9]*",x.match) if y.match != ""]) 
                    for x in eachmatch(r"mul\([0-9]*,[0-9]*\)",input)])
end
println(parse_input(input))
```

```         
161
```

### Part Two

The second part then specified that in parsing the string from left to right, if we came across a function that said `don't()`, we needed to ignore any `mul(X,Y)` operations until we found a `do()` function.

This again was a regex problem, though wrangling the regex proved a little tricky. Eventually, I settled on the following regex, which looked for all characters between a `don't()` and either a `do()` or the end of the string (h/t to [Aurélien on Bluesky](https://bsky.app/profile/atrotier.bsky.social/post/3lcgjsttpsc26) for noticing that my regex would fail if there was a `don't()` operation, then a `mul(X,Y)`, then an end of string). I then replaced all these matches with empty strings, and re-parsed the input to find the sum.

``` julia
function clean_input(input)
    return replace(input, r"don't\(\)((.|\n)*?)(do\(\)|$)" => "")
end
println(parse_input(clean_input(input)))
```

```         
48
```

## Day Four: Ceres Search

``` julia
# needed to provide a manual override to the parsing function here...
input = get_example(override="MMMSXXMASM
MSAMXMSMSA
AMXSXMAAMM
MSAMASMSMX
XMASAMXAMM
XXAMMXXAMA
SMSMSASXSS
SAXAMASAAA
MAMMMXMMMM
MXMXAXMASX")
input = split.(input,"")
```

```         
10-element Vector{Vector{SubString{String}}}:
 ["M", "M", "M", "S", "X", "X", "M", "A", "S", "M"]
 ["M", "S", "A", "M", "X", "M", "S", "M", "S", "A"]
 ["A", "M", "X", "S", "X", "M", "A", "A", "M", "M"]
 ["M", "S", "A", "M", "A", "S", "M", "S", "M", "X"]
 ["X", "M", "A", "S", "A", "M", "X", "A", "M", "M"]
 ["X", "X", "A", "M", "M", "X", "X", "A", "M", "A"]
 ["S", "M", "S", "M", "S", "A", "S", "X", "S", "S"]
 ["S", "A", "X", "A", "M", "A", "S", "A", "A", "A"]
 ["M", "A", "M", "M", "M", "X", "M", "M", "M", "M"]
 ["M", "X", "M", "X", "A", "X", "M", "A", "S", "X"]
```

For this puzzle, we were provided with a grid of characters `["X","M","A","S"]` that composed a word search. For the search, we needed to find out how many times the word `"XMAS"` appeared with those characters in order, be it up, down, left, right, diagonal, backwards, etc.

To start, I wrote a function that would search for `"XMAS"` in a string.

``` julia
search_xmas(x) = sum(length.(findall.(r"XMAS",x)))
```

Then, I checked left by going over each row of the input, then checked by reversing each row then iterating through it.

``` julia
n_left = search_xmas(join.(eachrow(mat)))
n_right = search_xmas(reverse.(join.(eachrow(mat))))
```

Next, I wanted to look up and down, so I transposed my word search matrix, then checked left and right again.

``` julia
transpose_str(x) = permutedims(x, (2,1))
n_up = search_xmas(join.(eachrow(transpose_str(mat))))
n_down = search_xmas(reverse.(join.(eachrow(transpose_str(mat)))))
```

Now I needed to check diagonally, in four different directions--left-up, left-down, right-up, right-down. The `diag()` function from `LinearAlgebra.jl` turned out to be really useful here--I could generate diagonal slices of a matrix and search over them.

``` julia
using LinearAlgebra
dim = size(mat)[1]
diags_1 = diag.([mat],(1-dim):(dim-1))
rev_mat = mapreduce(permutedims, vcat, reverse.(input))
diags_2 = diag.([rev_mat],(1-dim):(dim-1))
n_diag_1 = search_xmas(join.(diags_1))
n_diag_2 = search_xmas(reverse.(join.(diags_1)))
n_diag_3 = search_xmas(join.(diags_2))
n_diag_4 = search_xmas(reverse.(join.(diags_2)))
```

Finally, I just summed up the results of all of my searches.

``` julia
println(n_left + n_right + n_up + n_down + n_diag_1 + n_diag_2 + n_diag_3 + n_diag_4)
```

```         
18
```

### Part Two

The second part requires us to find how frequently we find two `"MAS"` strings together in an "X" form, with the "A" as the midpoint. For example:

```         
M.S
.A.
M.S
```

Is a valid input, as it contains `"MAS"` going left-down and going left-up with `"A"` as the midpoint.

This was a little bit more straightforward--I divided the word search matrix into 3x3 slices, then checked if "MAS" appeared in at least two of the four diagonal directions.

``` julia
global counter = 0
for i in 1:size(mat)[1]-2, j in 1:size(mat)[2]-2
    slice = mat[i:(i+2),j:(j+2)]
    slice_diag_1 = slice[1,1] * slice[2,2] * slice[3,3]
    slice_diag_2 = slice[1,3] * slice[2,2] * slice[3,1]
    if ((slice_diag_1== "MAS") | (slice_diag_1 == "SAM")) & ((slice_diag_2 == "MAS") | (slice_diag_2 == "SAM"))
        global counter = counter + 1
    end
end
println(counter)
```

```         
9
```

## [Day Five: Print Queue](https://adventofcode.com/2024/day/5)

``` julia
input = get_example(2024,5)
```

```         
27-element Vector{String}:
 "47|53"
 "97|13"
 "97|61"
 "97|47"
 "75|29"
 ⋮
 "97,61,53,29,13"
 "75,29,13"
 "75,97,47,61,53"
 "61,13,29"
 "97,13,75,29,47"
```

### Part One

The input for this problem consisted of two parts: 1) a series rules for how to arrange page numbers (ex. `"47|53"`, indicates that page 47 must come before page 53), and 2) a series of pages to print (ex. `"97,61,53,29,13"` is an instruction to print pages 97, 61, 53, 29, and then 13 in that order).

For the first part, we needed to identify which print orders followed the rules above, then grab and sum the middle numbers from the correct print orders. To do this, I went through each rule and each print order in conjunction, and search for any orders that didn't follow the rules. If I couldn't find any, I returned the print order, grabbed the median value, and summed that up.

``` julia
find_sum_med(vec) = sum(parse.(Int64,getindex.(vec,Int.(floor.(length.(vec) ./ 2) .+ 1))))
rules = split.(input[occursin.("|",input)],"|")
orders = split.(input[occursin.(",",input)],",")
correct_orders = [order for order in orders 
                    if !any([(rule[1] in order) & (rule[2] in order) && 
                        findfirst(x -> x == rule[1],order) > findfirst(x -> x == rule[2],order) 
                            for rule in rules])]
println(find_sum_med(correct_orders))
```

```         
143
```

### Part Two

For the second part of our problem, we needed to put incorrect print orders in the proper arrangement by the rules. To do this, I filtered to incorrect print orders, then ran a bubble-sort using the ordering rules. A bubble-sort is hardly the fasted algorithm, but it was the fastest to write and ran arbitrarily quickly thanks to Julia, so I went with it to get a solution out quickly for AOC leaderboard purposes.

``` julia
wrong_orders = orders[.!in.(orders,[correct_orders])]
for order in wrong_orders, i in 1:length(order)-1, j in 2:length(order)
    if !in([order[j-1],order[j]],rules)
        tmp = order[j-1]
        order[j-1] = order[j]
        order[j] = tmp
    end
end
println(find_sum_med(wrong_orders))
```

```         
123
```

## [Day Six: Guard Gallivant](https://adventofcode.com/2024/day/6)

``` julia
input = get_example(2024,6)
input = split.(input,"")
mat = mapreduce(permutedims, vcat, input)
```

```         
10×10 Matrix{SubString{String}}:
 "."  "."  "."  "."  "#"  "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "."  "."  "#"
 "."  "."  "."  "."  "."  "."  "."  "."  "."  "."
 "."  "."  "#"  "."  "."  "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "#"  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "."  "."  "."
 "."  "#"  "."  "."  "^"  "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "."  "."  "#"  "."
 "#"  "."  "."  "."  "."  "."  "."  "."  "."  "."
 "."  "."  "."  "."  "."  "."  "#"  "."  "."  "."
```

### Part One

In the input above, the matrix represents a laboratory, the arrow `"^"` represents a guard, and the hash `"#"` represents an obstacle. The guard moves in the direction that he's facing until he reaches an obstacle, at which point he rotates 90 degrees and starts moving in that direction. Eventually, the guard leaves the space.

The first part of the problem asked how many distinct spaces will the guard visit until they leave the space, including the starting space.

To approach this problem, I used a `while` loop to handle movement of the guard until they left the matrix space. For each iteration, the guard moves in the direction of the arrow, and then I used an `OrderedDict()` with modular division to handle changing directions when the guard ran into an obstacle. For each space the guard passed over, I changed that value in the matrix to a `"X"`. Finally, once the guard left the boundaries of the matrix, I counted how many values were `"X"` in the matrix to find my answer.

``` julia
using OrderedCollections
MOVES = OrderedDict("^" => [-1,0],">" => [0,1],"v" => [1,0],"<" => [0,-1])
make_move(cord) = cord .+ MOVES[mat[cord...]]
start = findall(x->x in ["^",">","v","<"],mat)
global inbounds, cord = true, Tuple.(start)[1]
while inbounds
    new_cord = make_move(cord)
    if (new_cord[1] > size(mat)[1]) | (new_cord[1] < 1) | (new_cord[2] > size(mat)[2]) | (new_cord[2] < 1)
        mat[cord...] = "X"
        global inbounds = false
    elseif mat[new_cord...] == "#"
        mat[cord...] = String.(keys(MOVES))[mod(findall(x->x == mat[cord...],String.(keys(MOVES)))[1],4) + 1]
    else
        mat[new_cord...] = mat[cord...]
        mat[cord...] = "X"
        global cord = new_cord
    end
end
println(length(findall(x->x == "X",mat)))
```

```         
41
```

### Part Two

The second part of the problem involved a little problem solving--we wanted to place an obstacle (`"#"`) in the guard's path such that the guard would become stuck in an infinite loop--the answer to the second part is the number of possible spaces where an infinite loop would become possible.

The second part of the problem was by far the most complicated to solve of any of the AOC problems that I have done, and I'm certainly the least proud of my (ultimately correct, horribly inefficient) solution.

To solve, I first iterated over all of the spaces that the guard passed through (no point in putting an obstacle down that the guard will never encounter). From there, I placed an obstacle in each space, then ran the same logic to move the guard. In a second matrix of the same size as the original matrix, I tracked the guards' movement and direction. I allowed the `while` loop to run with the terminating condition of the guard leaving the space, but with a second terminating condition--if the guard passed through the same space going the same direction as it had before, it was safe to assume that the guard was stuck in an infinite loop.

``` julia
ref_mat = copy(mat)
global counter = 0
mat = mapreduce(permutedims, vcat, input)
stored_mat = copy(mat)
for i in findall(x->x == "X",ref_mat)
    global mat = copy(stored_mat)
    if !(mat[i] in keys(MOVES)) & (mat[i] != "#")
        movement_mat = fill(".",size(mat))
        mat[i] = "#"
        local inbounds = true
        local cord = Tuple.(start)[1]
        while inbounds
            new_cord = make_move(cord)
            if (new_cord[1] > size(mat)[1]) | (new_cord[1] < 1) | (new_cord[2] > size(mat)[2]) | (new_cord[2] < 1)
                inbounds = false
            elseif mat[new_cord...] == "#"
                mat[cord...] = String.(keys(MOVES))[mod(findall(x->x == mat[cord...],String.(keys(MOVES)))[1],4) + 1]
            else
                mat[new_cord...] = mat[cord...]
                movement_mat[cord...] = mat[cord...]
                cord = new_cord
            end
            if inbounds && mat[cord...] == movement_mat[cord...]
                inbounds = false
                global counter += 1
            end
        end
    end
end
println(counter)
```

```         
6
```

My solution turned out to be quite slow--it took about 4 minutes to run on my machine with the final input I had. I imagine this is because rather than doing any kind of in-place modification of the matrix, with each iteration of the matrix I re-allocated a new copy of the matrix entirely just to reset it. I imagine there's some logic I could to do reset the matrix intelligently, but ultimately I ended up leaving my solution as is.

## [Day Seven: Bridge Repair](https://adventofcode.com/2024/day/7)

``` julia
input = get_example(2024,7)
```

```         
9-element Vector{Vector{String}}:
 ["190:", "10", "19"]
 ["3267:", "81", "40", "27"]
 ["83:", "17", "5"]
 ["156:", "15", "6"]
 ["7290:", "6", "8", "6", "15"]
 ["161011:", "16", "10", "13"]
 ["192:", "17", "8", "14"]
 ["21037:", "9", "7", "18", "13"]
 ["292:", "11", "6", "16", "20"]
```

### Part One

For this problem, the left-most values in each vector represent a potential answer; the right values reflect numbers that could possibly be combined using either addition or multiplication to produce the left-most answer. So `190` could be produced by multiplying `10` by `19`, and `292` could be produced by adding `11` to `6`, then multiplying that by `16`, then adding `20` to that figure (the operations always move left to right). However, there's no way you could add or multiply `17` and `5` to get `83`, so there's no way any set of operators could produce a valid equation. The answer is the sum of all of the left-most values for equations that *could* be valid.

I wrote a function to test each equation `i` using a vector of potential operations `ops`. After parsing each equation, I generated a list of all possible combinations of `ops` (so if there are three values that could be combined, the potential combinations of operations could be `+,+`, `+,*`, `*,+`, or `*,*`), and then tested to see if any combination resulted in the first value in the equation. In an effort to avoid any extremely large numbers, I broke the loop if at any point in applying the operations, the operated-upon numbers became larger than the test number. If the equation could be valid, I returned the first value in the equation, otherwise I returned zero.

``` julia
function test_line(i, ops)
    test = parse(Int64, replace(i[1], r":"=>""))
    is_correct = false
    for op in Base.Iterators.product(Base.Iterators.repeated(ops, length(i)-2)...)
        test_num = parse(Int64, i[2])
        for k in 3:length(i)
            if test_num > test_num
                break
            elseif op[k-2] == "+"
                test_num = test_num + parse(Int64, i[k])
            elseif op[k-2] == "*"
                test_num = test_num * parse(Int64, i[k])
            end
            if test == test_num
                is_correct = true
                break
            end
        end
        if is_correct
            break
        end
    end
    return ifelse(is_correct, test, 0)
end
println(sum(test_line.(input, [["+","*"]])))
```

```         
3749
```

### Part Two

The second part introduced a third valid operator--the concatenation `||` operator (so for example, `81 + 40` would be `121`, which could then be concatenated with `27` to produce `12127`).

I modified my existing `test_line` function to handle this new operation and ran it again to find my answer. With my full input, this took roughly a minute to run, a product of there being so many possible combinations to handle given three different operations, but I was pleased that it was able to run that quickly given how large the numbers were that it was working with.

``` julia
function test_line_two(i, ops)
    test = parse(Int64, replace(i[1], r":"=>""))
    is_correct = false
    for op in Base.Iterators.product(Base.Iterators.repeated(ops, length(i)-2)...)
        test_num = parse(Int64, i[2])
        for k in 3:length(i)
            if test_num > test_num
                break
            elseif op[k-2] == "+"
                test_num = test_num + parse(Int64, i[k])
            elseif op[k-2] == "*"
                test_num = test_num * parse(Int64, i[k])
            else
                test_num = parse(Int64, string(test_num) * i[k])
            end
            if test == test_num
                is_correct = true
                break
            end
        end
        if is_correct
            break
        end
    end
    return ifelse(is_correct, test, 0)
end
println(sum(test_line.(input, [["+","*","||"]])))
```

```         
11387
```

## Looking ahead

With the first week down, I know these problems are only going to get harder (and probably involve a lot more code to solve)\... so we'll see if I'm able to keep pace. If I can fit my solutions for week two into a sensible amount of space, I'll be sure to write them up, or maybe split them into multiple posts, so keep an eye out!
