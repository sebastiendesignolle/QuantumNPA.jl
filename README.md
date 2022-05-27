# QuantumNPA

Code to do NPA in Julia. In development - names of important functions or
even the entire project could change.

Prerequisites:
```julia
using Pkg; Pkg.add(["Combinatorics", "JuMP", "SCS"])
```

Then to use or try out:
```julia
include("QuantumNPA.jl");
using .QuantumNPA
```
(The dot in the second line isn't a typo.)



## Working examples

Maximise CHSH at level 2 of the hierarchy:
```julia
julia> @dichotomic A1 A2 B1 B2;

julia> S = A1*(B1 + B2) + A2*(B1 - B2)
A1 B1 + A1 B2 + A2 B1 - A2 B2

julia> npa_max(S, 2)
2.828427124718305
```

Maximise Svetlichny at level 1 + A B + A C + B C:
```julia
julia> @dichotomic A[1:2] B[1:2] C[1:2];

julia> E(x,y,z) = A[x]*B[y]*C[z]
E (generic function with 1 method)

julia> S = -E(1,1,1) + E(1,1,2) + E(1,2,1) + E(1,2,2) + E(2,1,1) + E(2,1,2) + E(2,2,1) - E(2,2,2)
-A1 B1 C1 + A1 B1 C2 + A1 B2 C1 + A1 B2 C2 + A2 B1 C1 + A2 B1 C2 + A2 B2 C1 - A2 B2 C2

julia> npa_max(S, "1 + A B + A C + B C")
5.656854248886011
```
(note that the spaces e.g. between A and B are necessary in the string, since
party labels go from A to Z then AA to ZZ then AAA to ZZZ...)

Maximise a modified CHSH at level 1 + A B + A^2 B:
```julia
julia> npa_max(0.3 * A1 + 0.6 * A1*(B1 + B2) + A2*(B1 - B2), "1 + A B + A^2 B")
2.3584742798682132
```

Maximise `<A1>` subject to `<A1*(B1 + B2)> = 1.4` and `<A2*(B1 - B2)> = 1.4`,
assuming the operator variables are already defined:
```julia
julia> npa_max(A1, [A1*(B1 + B2) - 1.4, A2*(B1 - B2) - 1.4], 2)
0.19802950752624165
```

The above examples all use dichotomic variables, but projectors are also
supported. Here we maximise the CH74 form of CHSH:
```julia
julia> PA11, PA12 = projector(1,1,1:2);

julia> PB11, PB12 = projector(2,1,1:2);

julia> npa_max(-PA11 - PB11 + PA11*(PB11 + PB12) + PA12*(PB11 - PB12), 1)
0.20710681094471445

julia> (sqrt(2) - 1)/2
0.20710678118654757
```

Maximise CGLMP with d=3 at level 1 + A B:
```julia
julia> npa_max(cglmp(3), "1 + A B")
2.914855484110488

julia> 1 + sqrt(11/3)
2.914854215512676
```
This uses a function `cglmp()` already defined in `qnpa.jl` to construct the
CGLMP operator.

Maximise the global guessing probability Pg(A1,B1|E) in the CHSH setting
using full statistics:
```julia
# Create projectors. The keyword argument 'full=true' means that the
# operator corresponding to the highest-numbered output is directly set to
# the identity minus the other ones. For example,
#
#   PA[2,1] = Id - PA[1,1]
#
# and
#
#   PE[4] = Id - PE[1] - PE[2] - PE[3]
#
# This is meant to make working in the Collins-Gisin projection (which the
# NPA code uses) more convenient.
PA = projector(1, 1:2, 1:2, full=true)
PB = projector(2, 1:2, 1:2, full=true)
PE = projector(5, 1:4, 1, full=true)

# CHSH = 2*sqrt(2) * p
p = 0.9

# Expectation value of G is the probability that Eve correctly guesses
# Alice's and Bob's joint outcome.
G = sum(PA[a,1] * PB[b,1] * PE[2*(a-1) + b]
        for a in 1:2 for b in 1:2)

# Ideal CHSH-violating correlations mixed with noise. N.B., the actual
# constraints imposed are on the expectation values of the operators
# in the array.
constraints = [PA[1,1] - 0.5,
               PA[1,2] - 0.5,
               PB[1,1] - 0.5,
               PB[1,2] - 0.5,
               PA[1,1]*PB[1,1] - 0.25*(1 + p/sqrt(2)),
               PA[1,1]*PB[1,2] - 0.25*(1 + p/sqrt(2)),
               PA[1,2]*PB[1,1] - 0.25*(1 + p/sqrt(2)),
               PA[1,2]*PB[1,2] - 0.25*(1 - p/sqrt(2))]

# This returns about 0.74618 for p = 0.9 at level 2 using the default SCS
# solver.
npa_max(G, constraints, 2)
```

QuantumNPA calls the SCS solver by default (since it doesn't require a
license) to solve the NPA relaxation of a quantum optimisation problem, but a
keyword argument lets you specify a different one. E.g., solve a problem
using Mosek (requires a license):
```julia
julia> using Mosek, MosekTools

julia> npa_max(S, 2, solver=Mosek.Optimizer)
2.82842711211242
```
You can also change the default solver if you don't want to specify it every
time, e.g.,
```julia
julia> QuantumNPA.set_solver(Mosek.Optimizer)
```

If you want to construct a JuMP model and solve it separately:
```julia
julia> model = npa2jump(S, "1 + A B", solver=SCS.Optimizer)
A JuMP Model
Maximization problem with:
Variables: 16
Objective function type: GenericAffExpr{Float64,VariableRef}
`Array{GenericAffExpr{Float64,VariableRef},1}`-in-`MathOptInterface.PositiveSemidefiniteConeSquare`: 1 constraint
Model mode: AUTOMATIC
CachingOptimizer state: EMPTY_OPTIMIZER
Solver name: SCS
Names registered in the model: v

julia> optimize!(model)
----------------------------------------------------------------------------
        SCS v2.1.4 - Splitting Conic Solver
        (c) Brendan O'Donoghue, Stanford University, 2012
----------------------------------------------------------------------------
Lin-sys: sparse-direct, nnz in A = 36
eps = 1.00e-05, alpha = 1.50, max_iters = 5000, normalize = 1, scale = 1.00
acceleration_lookback = 10, rho_x = 1.00e-03
Variables n = 16, constraints m = 45
Cones:  sd vars: 45, sd blks: 1
Setup time: 3.08e-04s
SCS using variable warm-starting
----------------------------------------------------------------------------
 Iter | pri res | dua res | rel gap | pri obj | dua obj | kap/tau | time (s)
----------------------------------------------------------------------------
     0| 2.54e+19  0.00e+00  1.00e+00 -2.56e+19 -0.00e+00  2.06e+19  1.76e-04 
    20| 1.54e-09  2.26e-09  1.05e-09 -2.83e+00 -2.83e+00  2.38e-17  2.25e-03 
----------------------------------------------------------------------------
Status: Solved
Timing: Solve time: 2.27e-03s
        Lin-sys: nnz in L factor: 97, avg solve time: 1.99e-06s
        Cones: avg projection time: 7.35e-05s
        Acceleration: avg step time: 2.23e-05s
----------------------------------------------------------------------------
Error metrics:
dist(s, K) = 1.9436e-09, dist(y, K*) = 2.9284e-09, s'y/|s||y| = 7.9234e-12
primal res: |Ax + s - b|_2 / (1 + |b|_2) = 1.5397e-09
dual res:   |A'y + c|_2 / (1 + |c|_2) = 2.2622e-09
rel gap:    |c'x + b'y| / (1 + |c'x| + |b'y|) = 1.0502e-09
----------------------------------------------------------------------------
c'x = -2.8284, -b'y = -2.8284
============================================================================

julia> objective_value(model)
2.828427121779378

julia> model[:v]
1-dimensional DenseAxisArray{VariableRef,1,...} with index sets:
    Dimension 1, Monomial[A1 A2, A1, A1 A2 B1, A1 A2 B2, A1 A2 B1 B2, A2 B1, B2, B1, B1 B2, A1 B2, A1 A2 B2 B1, A1 B1, A2 B2, A2 B1 B2, A1 B1 B2, A2]
And data, a 16-element Array{VariableRef,1}:
 v[A1 A2]
 v[A1]
 v[A1 A2 B1]
 v[A1 A2 B2]
 v[A1 A2 B1 B2]
 v[A2 B1]
 v[B2]
 v[B1]
 v[B1 B2]
 v[A1 B2]
 v[A1 A2 B2 B1]
 v[A1 B1]
 v[A2 B2]
 v[A2 B1 B2]
 v[A1 B1 B2]
 v[A2]
```



## Basic features

We can do arithmetic with and take conjugates of some different types of
operators that we associate to different parties. At the moment:
- dichotomic,
- fourier,
- projector,
- unitary,
- zbff (operators for Brown-Fawzi-Fawzi method).

The identity is represented by a variable `Id` that is predefined.
```julia
julia> Id
Id

julia> projector(1, 2, 3)
PA2|3

julia> PA = projector(1, 1:2, 1:2);

julia> PB = projector(2, 1:2, 1:2);

julia> PA[1,1]
PA1|1

julia> PA[1,1]*PB[1,1]
PA1|1 PB1|1

julia> PB[1,1]*PA[1,1]
PA1|1 PB1|1

julia> PA[1,1]*PA[2,1]
0

julia> PA[1,1]*PA[1,1]
PA1|1

julia> UA = unitary(1, 1:3);

julia> V = UA[1]*conj(UA[2])*UA[3]
UA1 UA*2 UA3

julia> P = PA[1,1] + V
PA1|1 + UA1 UA*2 UA3

julia> conj(P)
PA1|1 + UA*3 UA2 UA*1

julia> P*P
PA1|1 + PA1|1 UA1 UA*2 UA3 + UA1 UA*2 UA3 PA1|1 + UA1 UA*2 UA3 UA1 UA*2 UA3

julia> conj(P)*P
Id + PA1|1 + PA1|1 UA1 UA*2 UA3 + UA*3 UA2 UA*1 PA1|1

julia> Q = Id + V*PA[1,1]
Id + UA1 UA*2 UA3 PA1|1

julia> Q*Q
Id + 2 UA1 UA*2 UA3 PA1|1 + UA1 UA*2 UA3 PA1|1 UA1 UA*2 UA3 PA1|1

julia> conj(Q)*Q
Id + PA1|1 + PA1|1 UA*3 UA2 UA*1 + UA1 UA*2 UA3 PA1|1

julia> ZE = zbff(5, 1:2);

julia> R = PA[1,1]*PB[2,2]*ZE[1]*ZE[2]
PA1|1 PB2|2 ZE1 ZE2

julia> conj(R)
PA1|1 PB2|2 ZE*2 ZE*1

julia> FA = fourier(1, 9, 1, 5)
A9^1

julia> FA^0
Id

julia> FA^3
A9^3

julia> conj(FA^3)
A9^2

julia> FA^5
Id

julia> FA^6
A9^1

julia> conj(FA^3)*FA^3
Id

julia> FA*FA
A9^2

julia> conj((FA*FA)^4)
A9^2

julia> A1, A2 = dichotomic(1, 1:2);

julia> B1, B2 = dichotomic(2, 1:2);

julia> S = A1*(B1 + B2) + A2*(B1 - B2)
A1 B1 + A1 B2 + A2 B1 - A2 B2

julia> S^2
4 Id - A1 A2 B1 B2 + A1 A2 B2 B1 + A2 A1 B1 B2 - A2 A1 B2 B1
```

The functions that create monomials and their parameters are
```julia
dichotomic(party, input)
fourier(party, input, power, d)
projector(party, output, input, full=false)
unitary(party, index, conj=false)
zbff(party, index, conj=false)
```
In these:
- Party numbers start from 1.
- The parameters called `input`, `output`, and `index` can be either integers
  or arrays or ranges of integers.
- The parameter `conj` is optional and defaults to `false` if it is omitted.
- For projectors, if you give a range of inputs you can also give a value for
  a fourth parameter `full`, which defaults to `false`. Setting it to `true`
  indicates that you indend for the range of outputs to represent the full
  set of measurement outcomes. In that case, in place of the last projector
  you are given the identity minus the sum of all the preceding projectors.

Couple of examples:
```julia
julia> projector(1, 1:2, 1:2)
2×2 Array{Monomial,2}:
 PA1|1  PA1|2
 PA2|1  PA2|2

julia> projector(1, 1:3, 1:2, full=true)
3×2 Array{Any,2}:
 PA1|1               PA1|2
 PA2|1               PA2|2
 Id - PA1|1 - PA2|1  Id - PA1|2 - PA2|2

julia> julia> zbff(1, 1:3)
3-element Array{Monomial,1}:
 ZA1
 ZA2
 ZA3
```

I am working on writing macros to automatically create variables using
"standard" names. At the moment you can do, e.g., this to create some
dichotomic variables:
```julia
@dichotomic A1 A2 B1 B2 C[1:3]
```
The above macro invocation does essentially the same as running the
following:
```julia
A1 = dichotomic(1, 1)
A2 = dichotomic(1, 2)
B1 = dichotomic(2, 1)
B2 = dichotomic(2, 2)
C = dichotomic(3, 1:3)
```

There are no special relations (at least, at the moment) between the
different types of operators, so you shouldn't, for example, mix projectors
and dichotomic operators unless you consider them to be unrelated to each
other:
```julia
julia> dichotomic(1, 1) * projector(1, 1, 1)
A1 PA1|1

julia> dichotomic(1, 1) - (2*projector(1, 1, 1) - Id)
Id + A1 - 2 PA1|1
```


## Analysing, modifying, and deconstructing operators

Monomials and polynomials are objects of different types, although a
polynomial consisting of a single monomial multiplied by 1 is printed the
same as a monomial:
```julia
julia> P = projector(1, 1, 1)
PA1|1

julia> typeof(P)
Monomial

julia> Q = 1*P
PA1|1

julia> typeof(Q)
Polynomial

julia> S
A1 B1 + A1 B2 + A2 B1 - A2 B2

julia> typeof(S)
Polynomial
```
If you need to ensure a given object is a polynomial you can "promote" it by
calling `Polynomial()` on it. This does nothing if the argument is already a
polynomial:
```julia
julia> x = Polynomial(1)
Id

julia> y = Polynomial(Id)
Id

julia> z = Polynomial(S)
A1 B1 + A1 B2 + A2 B1 - A2 B2

julia> typeof.([x, y, z])
3-element Array{DataType,1}:
 Polynomial
 Polynomial
 Polynomial
```
Note that, in the last case, the polynomial returned is the same as (and not
a copy of) the original, which means that modifying `z` here will modify `S`
since they are the same object:
```julia
julia> z === S
true

julia> z[A1] = 7
7

julia> S
7 A1 + A1 B1 + A1 B2 + A2 B1 - A2 B2
```
If you want to create a copy of a polynomial that you can safely modify
without changing the original you can call the `copy()` function to do this:
```julia
julia> S = A1*(B1 + B2) + A2*(B1 - B2)
A1 B1 + A1 B2 + A2 B1 - A2 B2

julia> z = copy(S)
A1 B1 + A1 B2 + A2 B1 - A2 B2

julia> z === S
false

julia> z[A1] = 7
7

julia> z
7 A1 + A1 B1 + A1 B2 + A2 B1 - A2 B2

julia> S
A1 B1 + A1 B2 + A2 B1 - A2 B2
```

As the two above examples suggest, you can access and/or modify the
coefficient associated to a given monomial using `[]`:
```julia
julia> S[A1*B1]
1

julia> S[A1]
0
```
You can get all the monomials in a polynomial by calling the `monomials()`
function on it:
```julia
julia> monomials(S)
Base.KeySet for a Dict{Monomial,Number} with 4 entries. Keys:
  A1 B2
  A1 B1
  A2 B1
  A2 B2
```
Polynomials will also act as iterators over pairs of their nonzero
coefficients and monomials in contexts where an iterator is expected:
```julia
julia> collect(S)
4-element Array{Any,1}:
4-element Array{Any,1}:
 Pair{Number,Monomial}(1, A1 B2)
 Pair{Number,Monomial}(1, A2 B1)
 Pair{Number,Monomial}(1, A1 B1)
 Pair{Number,Monomial}(-1, A2 B2)

julia> for (c, m) in S
           @printf "%s  =>  %2d\n" m c
       end
A2 B1  =>   1
A1 B2  =>   1
A2 B2  =>  -1
A1 B1  =>   1
```
If you want to iterate over the monomials in lexicographical order you can
just call `sort()` on the polynomial first:
```julia
julia> for (c, m) in sort(S)
           @printf "%s  =>  %2d\n" m c
       end
A1 B1  =>   1
A1 B2  =>   1
A2 B1  =>   1
A2 B2  =>  -1
```

In order to help analyse a problem, there is a function `operators()` that
can find and return all the individual (order 1) operators in one or more
monomials and polynomials or collections of such operators. For instance, if
we wanted to maximise the local guessing probability in the CHSH setting
using full statistics we might represent the problem by an objective
polynomial and a list of constraint polynomials whose expectation values we
want to set to zero:
```julia
A1, A2 = dichotomic(1, 1:2)
B1, B2 = dichotomic(2, 1:2)
E1 = dichotomic(5, 1)

objective = (1 + A1*E1)/2

constraints = [A1, A2, B1, B2,
               A1*B1 - 0.7,
               A1*B2 - 0.7,
               A2*B1 - 0.7,
               A2*B2 + 0.7]
```
Assuming these variable definitions, we can use the `operators()` function to
immediately find all the level-one operators in the problem:
```julia
julia> operators(objective, constraints)
Set{Monomial} with 5 elements:
  A2
  A1
  B1
  B2
  E1
```
`operators()` can optionally take a keyword argument `by_parties` which is
set to `false` by default. Setting it to `true` groups the level-one
operators by party and returns a dictionary of the parties and operators
associated to those parties:
```julia
julia> operators(objective, constraints, by_party=true)
Dict{Integer,Set{Monomial}} with 3 entries:
  2 => Set(Monomial[B1, B2])
  5 => Set(Monomial[E1])
  1 => Set(Monomial[A2, A1])
```
This should be useful if we want to determine all the monomials in a problem
at NPA levels like "1 + A B + A E + B E"...


## NPA example

This short example finds what operators appear and where in the NPA moment
matrix at level 2 for the CHSH problem. It covers only the upper triangular
part and treats monomials and their conjugates as the same.
```julia
A1, A2 = dichotomic(1, 1:2)
B1, B2 = dichotomic(2, 1:2)
ops1 = [Id, A1, A2, B1, B2]
ops2 = sort(Set(O1*O2 for O1 in ops1 for O2 in ops1))

indices = Dict()

indexed_ops = collect(enumerate(ops2))

for (i, x) in indexed_ops
    for (j, y) in indexed_ops[i:end]
        m = conj(x)*y
        m = min(m, conj(m))

        if haskey(indices, m)
            push!(indices[m], (i, j))
        else
            indices[m] = [(i, j)]
        end
    end
end
```
This gives:
```julia
julia> for (m, l) in sort!(collect(indices), by=first)
           @printf "%11s  =>  %s\n" m l
       end
         Id  =>  [(1, 1), (2, 2), (3, 3), (4, 4), (5, 5), (6, 6), (7, 7), (8, 8), (9, 9), (10, 10), (11, 11), (12, 12), (13, 13)]
         A1  =>  [(1, 2), (3, 7), (4, 8), (5, 9)]
         A2  =>  [(1, 3), (2, 6), (4, 10), (5, 11)]
         B1  =>  [(1, 4), (2, 8), (3, 10), (5, 13)]
         B2  =>  [(1, 5), (2, 9), (3, 11), (4, 12)]
      A1 A2  =>  [(1, 6), (1, 7), (2, 3), (8, 10), (9, 11)]
      A1 B1  =>  [(1, 8), (2, 4), (7, 10), (9, 13)]
      A1 B2  =>  [(1, 9), (2, 5), (7, 11), (8, 12)]
      A2 B1  =>  [(1, 10), (3, 4), (6, 8), (11, 13)]
      A2 B2  =>  [(1, 11), (3, 5), (6, 9), (10, 12)]
      B1 B2  =>  [(1, 12), (1, 13), (4, 5), (8, 9), (10, 11)]
   A1 A2 A1  =>  [(2, 7)]
   A2 A1 A2  =>  [(3, 6)]
   A1 A2 B1  =>  [(2, 10), (3, 8), (4, 6), (4, 7)]
   A1 A2 B2  =>  [(2, 11), (3, 9), (5, 6), (5, 7)]
   A1 B1 B2  =>  [(2, 12), (2, 13), (4, 9), (5, 8)]
   A2 B1 B2  =>  [(3, 12), (3, 13), (4, 11), (5, 10)]
   B1 B2 B1  =>  [(4, 13)]
   B2 B1 B2  =>  [(5, 12)]
A1 A2 A1 A2  =>  [(6, 7)]
A1 A2 A1 B1  =>  [(7, 8)]
A1 A2 A1 B2  =>  [(7, 9)]
A2 A1 A2 B1  =>  [(6, 10)]
A2 A1 A2 B2  =>  [(6, 11)]
A1 A2 B1 B2  =>  [(6, 13), (7, 12), (8, 11)]
A1 A2 B2 B1  =>  [(6, 12), (7, 13), (9, 10)]
A1 B1 B2 B1  =>  [(8, 13)]
A1 B2 B1 B2  =>  [(9, 12)]
A2 B1 B2 B1  =>  [(10, 13)]
A2 B2 B1 B2  =>  [(11, 12)]
B1 B2 B1 B2  =>  [(12, 13)]
```

The example above uses `min(m, conj(m))` to find which of `m` or its
conjugate comes first lexicographically. It works because comparisons between
monomials are defined:
```julia
julia> A1 == A1
true

julia> A1 < A1
false

julia> A1 < A2
true

julia> A2 < A1*A2
true
```
`sort` used above works for the same reason. `==` and `!=` (but not the
inequalities) can also be used to compare monomials with polynomials or
polynomials with each other:
```julia
julia> A1 == 1*A1
true

julia> A1 < 1*A1
ERROR: MethodError: no method matching isless(::Monomial, ::Polynomial)
[...]
```



## Internal details

The way a list of operators are joined to multiply them is determined at the
moment by a function `join_ops` near the beginning of the file `qnpa.jl`. It
is not super general at the moment but is general enough to handle the
different types of operators already defined.

Associating operators in groups to parties is handled by the `Monomial`
type. At the moment it just contains a list
```julia
word = [(p1, ops1), (p2, ops2), ...]
```
of parties `p1`, `p2`, etc. and lists of operators `ops1`, `ops2`,
etc. associated to those parties. For example,
```julia
julia> R
PA1|1 PB2|2 ZE1 ZE2

julia> R.word
3-element Array{Tuple{Integer,Array{Operator,1}},1}:
 (1, [P1|1])
 (2, [P2|2])
 (5, [Z1, Z2])
```
It is assumed that:

1. parties are numbered starting from 1,
2. the party numbers are in strictly increasing order `p1 < p2 < p3 ...`, and
3. only parties that have at least one operator associated with them appear
   in the list.

This (mainly the definition of `Monomial` and the function
`Base.:*(x::Monomial, y::Monomial)`) is the part of the code that would have
to be modified if we wanted to support, e.g., operators acting on the
subystems of more than one party.
