---
layout:     post
title:      "Secret Sharing, Part 2"
subtitle:   "Efficient Sharing with the Fast Fourier Transform"
date:       2017-06-05 12:00:00
header-img: "img/post-bg-03.jpg"
author:           "Morten Dahl"
twitter_username: "mortendahlcs"
github_username:  "mortendahl"
---

<em><strong>TL;DR:</strong> efficient secret sharing requires fast polynomial evaluation and interpolation; here we show what it takes to use the Fast Fourier Transform for this.</em>

In the [first part](/2017/06/04/secret-sharing-part1/) on secret sharing we looked at Shamir's scheme and its packed variant where several secrets are shared together. Polynomials lie at the core of both schemes

do we use the values of a polynomial 

There is a Python notebook containing [the code samples](https://github.com/mortendahl/privateml/blob/master/secret-sharing/Fast%20Fourier%20Transform.ipynb), yet for better performance our [open source Rust library](https://crates.io/crates/threshold-secret-sharing) is recommended.

<em>
Parts of this blog post are derived from work done at [Snips](https://snips.ai/) and [originally appearing in another blog post](https://medium.com/snips-ai/high-volume-secret-sharing-2e7dc5b41e9a). That work also included parts of the Rust implementation.
</em>


**TODO TODO TODO motivation for using a prime field (as opposed to an extension field): it allows us to naturally use modular arithmetic to simulate bounded integer arithmetic, as useful for instance in secure computation. in the previous post we used prime fields, but all the algorithms there would carry directly over to an extension field (they will here as well, but again we focus on prime fields). binary extension fields would make the computations more efficient but are less suitable since it is less obvious which encoding/embedding to use in order to simulate integer arithmetic**

The algorithms in this post are significantly more complex than [what is typically used for Shamir sharing](https://github.com/mortendahl/privateml/blob/master/secret-sharing/Schemes.ipynb), and as such it's worth considering when there's an actual benefit to be had from using them. [Our Rust implementation](https://crates.io/crates/threshold-secret-sharing) perhaps provides a more realistic foundation for comparing the different algorithms and parameter choice; in particular, the library unsurprisingly shows that when either the number of secrets or number of shares is high then there a orders of magnitude efficiency improvements to be gained.


# Polynomials

If we [look back](/2017/06/04/secret-sharing-part1/) at Shamir's scheme we see that it's all about polynomials: a random polynomial embedding the secret is sampled, and the shares are taken as its value at certain points.

```python
def shamir_share(secret):
    polynomial = sample_shamir_polynomial(secret)
    shares = [ evaluate_at_point(polynomial, p) for p in SHARE_POINTS ]
    return shares
```

The same goes for the packed variant, where several secrets are embedded in the sampled polynomial.


```python
def packed_share(secrets):
    polynomial = sample_packed_polynomial(secrets)
    shares = [ interpolate_at_point(polynomial, p) for p in SHARE_POINTS ]
    return shares
```

Notice however that it differs in how the shares (as values of the polynomial) are found: Shamir's scheme uses `evaluate_at_point` while the packed uses `interpolate_at_point`. The reason is that the sampled polynomial in the former case is in *coefficient representation* while in the latter it is in *point-value representation*.

Specifically, we often represent a polynomial `f` of degree `N` by a list of `N+1` coefficients `a0, ..., aN` such that `f(x) = (a0) + (a1 * x) + (a2 * x^2) + ... + (aN * x^N)`. This representation is convenient for many things, including efficiently evaluating the polynomial at a given point using e.g. [Horner's method](https://en.wikipedia.org/wiki/Horner%27s_method).

However, every such polynomial may also be represented by a set of `N+1` point-value pairs `(p0, v0), ..., (pN, vN)` where `vi = f(pi)` and all the `pi` are distinct. And evaluating the polynomial at a given point is still possible, yet now requires a more involved *interpolation* procedure that is typically less efficient, at least unless some preprocessing is performed. 

At the same time, the point-value has several advantages over the coefficient representation, one of them being that the same `N+1` degree polynomial may be represented by *more than* `N+1` pairs... **TODO**
Note that same `N+1` degree polynomial may also be represented using *more than* `N+1` pairs in the point-value representation. In this case there is some redundancy in the representation, that we may for instance take advantage of in secret sharing (to reconstruct even if some shares are lost) and in coding theory (to decode correctly even if some errors occur during transmission). This would not be possible in the coefficient representation, since adding more (non-zero) coefficients increases the degree. However, in point-value representation it is, since the pairs are technically speaking defining the *least degree* polynomial `g` such that `g(pi) == vi`. The reason we can use to two representations somewhat interchangible is that if the pairs where generated by a `N+1` degree polynomial `f` then we are guaranteed that `g == f`.




# Fast Fourier Transform

Indeed, our current implementation of the packed scheme relies on the [Fast Fourier Transform](https://en.wikipedia.org/wiki/Fast_Fourier_transform) (FFT) over finite fields (also known as a [Number Theoretic Transform](https://en.wikipedia.org/wiki/Discrete_Fourier_transform_(general)#Number-theoretic_transform)) whereas the typical implementation of Shamir’s scheme only needs a simple evaluation of polynomials.


## Walk-through example

**todo swap w and v throughout**

Recall that all our computations happen in a prime field `Z_Q` for some prime `Q`; in this example we'll use `Q = 433`, which order `Q-1` is divisible by both `4` and `9`: `Q-1 == 432 == 4 * 9 * 12`.

Assume then that we have a polynomial `A(x) = 1 + 2x + 3x^2 + 4x^3` of degree `N-1 == 3`. For readability we will write this as `A(x) = 1 + 2x + 3x2 + 4x3` instead, and represent it as a list of `N` coefficients.

```python
A_coeffs = [ 1, 2, 3, 4 ]
```

Our goal is essentially to turn this list of coefficients into a list of equal length of values `[ A(w0), A(w1), A(w2), A(w3) ]` for some points `w = [w0, w1, w2, w3]`. The standard way of evaluating polynomials is of course one way of during this, which using Horner's rule can be done in a total of `Oh(N * N)` operations.

```python
def horner_evaluate(coeffs, point):
    result = 0
    for coef in reversed(coefs):
        result = (coef + point * result) % Q
    return result
  
A = lambda x: horner_evaluate(A_coeffs, x)

assert([ A(wi) for wi in w ] 
    == [ 12, 12, 12, 12 ])
```

As we will see though, the FFT will allow us to do so more efficiently when the length is sufficiently large and the points are chosen carefully; asymptotically it allows us to compute the values in `Oh(N * log N)` operations.

The first insight we need is that there is an alternative evaluation strategy that breaks `A(x)` into two smaller polynomials; in particular, if we define polynomials `B(y) = 1 + 3y` and `C(y) = 2 + 4y` by taking every other coefficient from `A(x)`, then we have `A(x) = B(x * x) + x * C(x * x)`.

This means that if we know values of `B(y)` and `C(y)` at *the squares* `v` of the `w` points then we can use these to compute the values of `A(x)` at the `w` points using table look-ups: `A_values[i] = B_values[i] + w[i] * C_values[i]`.

```python
# split A into B and C
B_coeffs = [ 1,    3,   ] # = A_coeffs[0::2]
C_coeffs = [    2,    4 ] # = A_coeffs[1::2]
B = lambda y: horner_evaluate(B_coeffs, y)
C = lambda y: horner_evaluate(C_coeffs, y)

# compute values for B and C
v = [ wi * wi % Q for wi in w ]
B_values = [ B(vi) for vi in v ]
C_values = [ C(vi) for vi in v ]

# combine results into values for A
A_values = [ ( B_values[i] + w[i] * C_values[i] ) % Q for i,_ in enumerate(w) ]

assert(A_values == [ A(wi) for wi in w ] )
```




So far we haven't saved much, but the second insight fixes that. Namely, by picking the points `w` to be the elements of a subgroup of order 4, the `v` points used for `B` and `C` will form a subgroup of order 2; in particular, we have `v[0] == v[2]` and `v[1] == v[3]` so we only need the first halves of `B_values` and `C_values`, and have hence cut the problem in half!

To do this we need a generator `OMEGA` of the subgroup of order `N`. We shall return to how this is found later, and for now simply state that `179` is one such when `Q = 433`.

```python
OMEGA = 179

assert( len(set( pow(OMEGA, e, Q) for e in range(Q) )) == N )

w = [ pow(OMEGA, e, Q) for e in range(N) ]

v = [ w[0] * w[0], w[1] * w[1] ]
assert(v == [ cds, cds ])

B_values = [ B(v[0]), B(v[1]) ]
C_values = [ C(v[0]), C(v[1]) ]
```
now with `A_values[i] = B_values[i % 2] + w[i] * C_values[i % 2]`.

The third and final insight we need is then that we can continue this process of diving the polynomial to evaluate in half: to compute e.g. `B_values` we can break `B` into two polynomials `D` and `E` of degree 0 (i.e. only a constant), and using `A_values` we could find the values of a larger degree 8 polynomial. The only requirement for this process is to make sure we always have a subgroup of the right order. 

## Dump


Why this is interesting will become clear below, but we see that if we already have lists `B_values` and `C_values` then we may now use these to compute our desired `A_values` by look-ups and a multiplication and addition: `A_values[i] = B_values[i] + w[i] * C_values[i]`.

```python
w = [ 1, 2, 3, 4 ]
v = [ wi * wi % Q for wi in w ]

A_values  = [ A(w[0]), A(w[1]), A(w[2]), A(w[3]) ]

B_values  = [ B(v[0]), B(v[1]), B(v[2]), B(v[3]) ]
C_values  = [ C(v[0]), C(v[1]), C(v[2]), C(v[3]) ]
BC_values = [ B_values[i] + w[i] * C_values[i] for i,_ in enumerate(w) ]

for i in enumerate(A_values):
  assert(  )
```
then we may use these to compute our desired
```python
A_values = [ A(w[0]), A(w[1]), A(w[2]), A(w[3]) ]
```


```python

w = [ 1, 2, 3, 4 ]

assert([ A(w[0]), A(w[1]), A(w[2]), A(w[3]) ] 
    == [       1,      10,        ,         ])
    
assert([ BC(w[0]), BC(w[1]), BC(w[2]), BC(w[3]) ]
    == [        1,       10,         ,          ])
```


One way of looking at this is that if we already have lists 
```python
B_values = [ B(w[0] * w[0]), B(w[1] * w[1]), B(w[2] * w[2]), B(w[3] * w[3]) ]
C_values = [ C(w[0] * w[0]), C(w[1] * w[1]), C(w[2] * w[2]), C(w[3] * w[3]) ]
```
then we may use these to compute our desired
```python
A_values = [ A(w[0]), A(w[1]), A(w[2]), A(w[3]) ]
```
by look-ups and a multiplication and addition: `A_values[i] = B_values[i] + w[i] * C_values[i]`.





`A(w0) = B(w0 * w0) + w0 * C(w0 * w0) = B(w0) + w0 * C(w0)`

`A(w1) = B(w1 * w1) + w1 * C(w1 * w1) = B(w2) + w1 * C(w2)`

`A(w2) = B(w2 * w2) + w2 * C(w2 * w2) = B(w4) + w2 * C(w4)`

`A(w3) = B(w3 * w3) + w3 * C(w3 * w3) = B(w6) + w3 * C(w6)`

`A(w^2) = B(w^4) + w^2 * C(w^4)`, but since `w^4 = w^(4 mod 4) = w^0` then we actually get `A(w^2) = B(w^0) + w^2 * C(w^0)`

`A(w^3) = B(w^6) + w^3 * C(w^6)` but since `w^6 = w^(6 mod 4) = w^2` then we actually get `A(w^3) = B(w^2) + w^3 * C(w^2)`


`A(x) = 1 + 2x + 3x^2 + 4x^3 + 5x^4 + 6x^5 + 7x^6 + 8x^7 + 9x^8`

If `B(y) = 1 + 4y + 7y^2`, `C(y) = 2 + 5y + 8y^2`, and `D(y) = 3 + 6y + 9y^2` then `A(x) = B(x^3) + x * C(x^3) + x^2 * D(x^3)`. Then, for similar reasons to above, we may compute `A(w^0), ..., A(w^8)` from `B(w^0), B(w^3), B(w^6)`




## Algorithm for power-of-2

```python
# len(aX) must be a power of 2
def fft2_forward(aX, omega=OMEGA2):
    if len(aX) == 1:
        return aX

    # split A(x) into B(x) and C(x) -- A(x) = B(x^2) + x C(x^2) -- and recurse
    bX = aX[0::2]
    cX = aX[1::2]
    B = fft2_forward(bX, (omega**2) % Q)
    C = fft2_forward(cX, (omega**2) % Q)
        
    # combine subresults
    A = [0] * len(aX)
    Nhalf = len(aX) // 2
    for i in range(Nhalf):
        
        j = i
        x = pow(omega, j, Q)
        A[j] = (B[i] + x * C[i]) % Q
        
        j = i + Nhalf
        x = pow(omega, j, Q)
        A[j] = (B[i] + x * C[i]) % Q
      
    return A
```

```python
def fft2_backward(A):
    N_inv = inverse(len(A))
    return [ (a * N_inv) % Q for a in fft2_forward(A, inverse(OMEGA2)) ]
```

Note that some optimizations omitted to clarity are possible and typically used; they are shown in the Python notebook.

## Algorithm for power-of-3

```python
# len(aX) must be a power of 3
def fft3_forward(aX, omega=OMEGA3):
    if len(aX) == 1:
        return aX

    # split A(x) into B(x), C(x), and D(x): A(x) = B(x^3) + x C(x^3) + x^2 D(x^3)
    bX = aX[0::3]
    cX = aX[1::3]
    dX = aX[2::3]
    
    # apply recursively
    omega_cubed = (omega**3) % Q
    B = fft3_forward(bX, omega_cubed)
    C = fft3_forward(cX, omega_cubed)
    D = fft3_forward(dX, omega_cubed)
        
    # combine subresults
    A = [0] * len(aX)
    Nthird = len(aX) // 3
    for i in range(Nthird):
        
        j = i
        x = (omega**j) % Q
        xx = (x * x) % Q
        A[j] = (B[i] + x * C[i] + xx * D[i]) % Q
        
        j = i + Nthird
        x = (omega**j) % Q
        xx = (x * x) % Q
        A[j] = (B[i] + x * C[i] + xx * D[i]) % Q
        
        j = i + Nthird + Nthird
        x = (omega**j) % Q
        xx = (x * x) % Q
        A[j] = (B[i] + x * C[i] + xx * D[i]) % Q

    return A
```

```python
def fft3_backward(A):
    N_inv = inverse(len(A))
    return [ (a * N_inv) % Q for a in fft3_forward(A, inverse(OMEGA3)) ]
```

# Application to Secret Sharing

## Shamir's scheme

```python
def shamir_share(secret):
    small_coeffs = [secret] + [random.randrange(Q) for _ in range(T)]
    large_coeffs = small_coeffs + [0] * (ORDER3 - len(small_coeffs))
    large_values = fft3_forward(large_coeffs)
    shares = large_values
    return shares
```


## Packed scheme

```python
def packed_share(secrets):
    small_values = [0] + secrets + [random.randrange(Q) for _ in range(T)]
    small_coeffs = fft2_backward(small_values)
    large_coeffs = small_coeffs + [0] * (ORDER3 - ORDER2)
    large_values = fft3_forward(large_coeffs)
    shares = large_values[1:]
    return shares
```

```python
def packed_reconstruct(shares):
    large_values = [0] + shares
    large_coeffs = fft3_backward(large_values)
    small_coeffs = large_coeffs[:ORDER2]
    small_values = fft2_forward(small_coeffs)
    secrets = small_values[1:K+1]
    return secrets
```

## Performance evaluation

# Parameter Generation

```python
def generate_parameters(min_bitsize, k, t, n):
    order2 = k + t + 1
    order3 = n + 1
    
    order_divisor = order2 * order3
    p, g = find_prime_field(min_bitsize, order_divisor)
    
    order = p - 1
    omega2 = pow(g, order // order2, p)
    omega3 = pow(g, order // order3, p)
    
    return p, omega2, omega3
```

```python
def find_prime_field(min_bitsize, order_divisor):
    p, order_prime_factors = find_prime(min_bitsize, order_divisor)
    g = find_generator(p, order_prime_factors)
    return p, g
```

```python
def find_generator(prime, order_prime_factors):
    order = prime - 1
    for candidate in range(2, Q):
        for factor in order_prime_factors:
            exponent = order // factor
            if pow(candidate, exponent, Q) == 1:
                break
        else:
            return candidate
```

## Finding primes

```python
def find_prime(min_bitsize, order_divisor):
    while True:
        k1 = sample_prime(min_bitsize)
        for k2 in range(128):
            p = k1 * k2 * order_divisor + 1
            if is_prime(p):
                order_prime_factors  = [k1]
                order_prime_factors += prime_factor(k2)
                order_prime_factors += prime_factor(order_divisor)
                return p, order_prime_factors
```

```python
def sample_prime(bitsize):
    lower = 1 << (bitsize-1)
    upper = 1 << (bitsize)
    while True:
        candidate = random.randrange(lower, upper)
        if is_prime(candidate):
            return candidate
```

```python
def prime_factor(x):
    factors = []
    for prime in SMALL_PRIMES:
        if prime > x: break
        if x % prime == 0:
            factors.append(prime)
            x = remove_factor(x, prime)
    assert(x == 1)
    return factors
```


TODO

# Conclusion
