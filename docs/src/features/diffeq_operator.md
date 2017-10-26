# DiffEqOperators

The `AbstractDiffEqOperator` interface is an interface for declaring parts of
a differential equation as linear or affine. This then allows the solvers to
exploit linearity to achieve maximal performance.

## Using DiffEqOperators

`AbstractDiffEqOperator`s act like functions. When defined, `A` has function
calls `A(t,u)` and `A(t,u,du)` that act like `A*u`. These operators update
via a function `update_coefficients!(A,t,u)`.

## Constructors

### Wrapping an Array: DiffEqArrayOperator

`DiffEqArrayOperator` is for defining an operator directly from an array. The
operator is of the form

```math
\alpha(t)A(t,u)
```

for some scalar `α` and time plus possibly state dependent `A`. The
constructor is:

```julia
DiffEqArrayOperator(A::AbstractMatrix{T},α=1.0,
                             update_func = DEFAULT_UPDATE_FUNC)
```

`A` is the operator array. `α` is the scalar coefficient. If `α` is a function
`α(t)`, then it will update the coefficient as necessary. `update_func` is the
function called by `update_coefficients!(A,t,u)` (along with `α` if it's a
function). If left as its default, then `update_func` is trivial which signifies
`A` is a constant.

### AffineDiffEqOperator

For `As = (A1,A2,...,An)` and `Bs = (B1,B2,...,Bn)` where each of the `Ai` and
`Bi` are `DiffEqLinearOperator`s, the following constructor:

```julia
function AffineDiffEqOperator{T}(As,Bs,u_cache=nothing)
```

builds an operator `L = (A1 + A2 + ... An)*u + B1 + B2 + ... + Bn`. `u_cache`
is for designating a type of internal cache for non-allocating evaluation of
`L(t,u,du)`. If not given, the function `L(t,u,du)` is not available. Note
that in solves which exploit this structure, this function call is not necessary.
It's only used as the fallback in ODE solvers which were not developed for this
structure.

## Formal Properties of DiffEqOperators

These are the formal properties that an `AbstractDiffEqOperator` should obey
for it to work in the solvers.

### AbstractDiffEqOperator Interface Description

1. Function call and multiplication: L(t,u,du) for inplace and du = L(t,u) for
   out-of-place, meaning L*u and A_mul_B!.
2. If the operator is not a constant, update it with (t,u). A mutating form, i.e.
   update_coefficients!(A,t,u) that changes the internal coefficients, and a
   out-of-place form B = update_coefficients(A,t,u).
3. is_constant(A) trait for whether the operator is constant or not.

### AbstractDiffEqLinearOpeartor Interface Description

1. AbstractDiffEqLinearOperator <: AbstractDiffEqOperator
2. Can absorb under multiplication by a scalar. In all algorithms things like
   dt*L show up all the time, so the linear operator must be able to absorb
   such constants.
4. is_constant(A) trait for whether the operator is constant or not.
5. Optional: diagonal, symmetric, etc traits from LinearMaps.jl.
6. Optional: expm(A). Required for simple exponential integration.
7. Optional: expmv(A,t,u) = expm(t*A)*u and expmv!(v,A::DiffEqOperator,t,u)
   Required for sparse-saving exponential integration.
8. Optional: factorizations. A_ldiv_B, factorize et. al. This is only required
   for algorithms which use the factorization of the operator (Crank-Nicholson),
   and only for when the default linear solve is used.