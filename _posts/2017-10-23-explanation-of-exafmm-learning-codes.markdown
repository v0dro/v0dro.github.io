---
layout: post
title: "Explanation of ExaFMM learning codes."
date: 2017-10-23 21:33:24 +0900
comments: true
categories:
---

In this file I will write descriptions of the exafmm 'learning' codes and my understanding of them. I have been tasked with understanding the code and porting it to Ruby, my favorite language.
We shall start from the first tutorial, i.e. [0_tree](). You can find the full Ruby code here.

<!-- MarkdownTOC style="round" autolink="true" depth="4" -->

- 0_tree
  - step1.cxx
  - step2.cxx
  - step3.cxx
  - step04.cxx
- 1_traversal
  - step1.cxx
  - step2.cxx
- 2_kernels
  - kernel.h
  - vector.h
  - exafmm.h
  - exafmm2d.h and step1.cxx
  - step2.cxx

<!-- /MarkdownTOC -->


# 0_tree

## step1.cxx

This program simply populates some bodies with random numbers, creates a hypothetical X and Y axes and figures out the quadrant of each of the bodies.

Each of the nodes of the tree have a maximum of [4 immediate children](). We first initialize 100 `struct Body` objects, and then set the X and Y co-ordinates of each of them to a random number between 0 and 1.

In order to actually build the tree we follow the following steps:

  * First [get the bounds]() between which the random numbers lie. That is, we figure out the min and max random number that is present in the bodies.
  * We then get a ['center' and a 'radius'](). This is useful for creating 'quadrants' and partitioning points into different quandrants in later steps. The center is calculated by adding the min and max numbers (which we treat as the diameter) and dividing by 2. This step is necessary since there is no 'square' space that can be partitioned into multiple spaces like there was in the lecture series. The way of calculating the radius `r0` is a little peculiar. It does not use the distance formula, its main purpose is....
  * And then simply count the bodies in each quadrant and display them.

Ruby code:
The body is represented as the Ruby class `Body`:
``` ruby
class Body
  attr_reader :x

  def initialize
    @x = [0.0, 0.0]
  end
end
```

There is an interesting way of knowing the quadrant in this code. It goes like this:
``` ruby
a = body.x[0] > x0[0] ? 1 : 0
b = body.x[1] > x0[1] ? 1 : 0
quadrant = a + (b << 1)
```
Above code basically plays with 0 and 1 and returns a number between 0 and 3 as the correct quadrant number.

## step2.cxx

This code basically takes the bodies created in the previous step, counts the number of bodies in each quadrant and sorts them by quadrant.

The new steps introduced in this program can be summarized as follows:
  * Count the bodies in each quadrant and store the count in an array. The `size` array in case of the Ruby implementation.
  * In the next step we successively add the number of elements in each quadrant so that it gives us the offset value at which elements from a new quadrant will start in the `bodies` Array \(of course, after it is sorted\).
  * We then sort the bodies according to the quadrant that they belong to. Something peculiar that I notice about this part is that `counter[quadrant]` also gets incremented after each iteration for sorting. Why is this the case even though the counters have been set to the correct offsets previously?

## step3.cxx

This program introduces a new method called `buildTree`, inside of which we will actually build the tree. It removes some of the sorting logic from `main` and puts it inside `buildTree`. The `buildTree` function performs the following functions:
  * Most of the functions relating to sorting etc are same. Only difference is that there is in-place sorting of the `bodies` array and the `buffer` array does not store elements anymore.
  * A new function introduced is that we re-calculate the center and the radius based on sorted co-ordinates. This is done because we want new center and radii for the children.
  * The `buildTree` function is called recursively such that the quadrants are divided until a point is reached where the inner most quadrant in the hierarchy does not contain more than 4 elements.

Implementation:

There is an interesting piece of code in the part for calculating new center and radius:
``` ruby
 # i is quadrant number
center[d] = x0[d] + radius * (((i & 1 << d) >> d) * 2 - 1)
```

In the above code, there is some bit shifting and interleaving taking place whose prime purpose is to split the quadrant number into X and Y dimension and then using this to calculate the center of the child cell.

Another piece of code is this:
``` ruby
counter = Array.new 4, start
1.upto(3) do |i|
  counter[i] = size[i-1] + counter[i-1]
end

# sort bodies and store them in buffer
buffer = bodies.dup
start.upto(finish-1) do |n|
  quadrant = quadrant_of x0, buffer[n]
  bodies[counter[quadrant]] = buffer[n]
  counter[quadrant] += 1
end
```

In the above code, the `counter` variable is first used to store offsets of the elements in different quadrants. In the next loop it is in fact a counter for that stores in the index of the body that is currently under consideration.

## step04.cxx

In this step we use the code written in the previous steps and actually build the tree.
The tree is built recursively by splitting into quadrants and then assigning them to cells
based on the quadrant. The 'tree' is actually stored in an array.

The cells are stored in a C++ vector called `cells`.

In the `Cell` struct, I wonder why the body is stored as a pointer and not a variable.

Implementation in the Ruby code, like saving the size of an Array during a recursive call
is slightly different since Ruby does not support pointers, but the data structures and
overall code is more or less a direct port.

# 1_traversal

These codes are for traversal of the tree that was created in the previous step. The full code can be found in [1_traversal.rb]() file.

## step1.cxx

This step implements the P2M and M2M passes of the FMM.

One major difference between the C++ and Ruby implementation is that since Ruby does not have pointers, I
have used the array indices of the elements instead. For this purpose there are two attributes in the
`Cell` class called `first_child_index` that is responsible for holding the index in the `cells` array
about the location of the first child of this cell, and the second `first_body_index` which is responsible for holding the index of the body in the `bodies` array.

This step does this by introducing a method called `upwardPass` which iterates through nodes and thier children and computes the P2M and M2M kernels.

## step2.cxx

This step implements the rest of the kernels i.e. M2L, L2L, L2P and P2P. It also introduces two new methods `downward_pass` that calculates the local forces from other local forces and L2P interactions and `horizontal_pass` that calculates the inter-particle interactions and m2l.

No special code as such over here, its just the regular FMM stuff.

# 2_kernels

This code is quite different from the previous two. While the previous programs were mostly retricted to a single file, this program substantially increases complexity and spreads the implementation across several files. We start using 3 dimensional co-ordinates too.

In this code, we start to make a move towards spherical co-ordinate system to represent the particles in 3D. A few notable algorithms taken from some research papers have been implemented in this code.

Lets describe each file and see what implementation lies inside

## kernel.h

The `kernel.h` header file implemenets all the FMM kernels. It also implements two special functions called `evalMultipole` and `evalLocal` that evaluate the multipoles and local expansion for spherical co-ordinates using the actual algorithm that is actually used in exafmm. An implementation of this algorithm can be found on page 16 of the paper ["Treecode and fast multipole method for N-body simulation with CUDA"](https://arxiv.org/pdf/1010.1482.pdf%20) by Yokota sensei. A preliminary implementation of this algorithm can be found in ["A Fast Adaptive Multipole Algorithm in Three Dimensions"](http://www.sciencedirect.com/science/article/pii/S0021999199963556) by Cheng.

The Ruby implementation of this file is in `kernel.rb`.

I will now describe this algorithm here best I can:

### Preliminaries

#### Ynm vector

This is a vector that defines the [spherical harmonics](https://en.wikipedia.org/wiki/Spherical_harmonics) of degree _n_ and order _m_. A primitive version for computing this exists in the paper by Cheng and a newer, faster version in the paper by Yokota.

Spherical harmonics allow us to define series of a function in 3D rather in 1D that is usually the case for things like the expansion of _sin(x)_. They are representations of functions on the surface of a sphere instead of on a circle, which is usually the case with other 2D expansion functions. They are like the Fourier series of the sphere. This [article](http://mathworld.wolfram.com/SphericalHarmonic.html) explains the notations used nicely.

The order (_n_) and degree (_m_) correspond to the order and degree of the [Legendre polynomial](http://mathworld.wolfram.com/LegendrePolynomial.html) that is used for obtaining the spherical harmonic. _n_ is an integer and _m_ goes from _0..n_.

For causes of optimization, the values stored inside `ynm` are not the ones that correspond to the spherical harmonic, but are values that yield optimized results when the actual computation happens.

#### Historical origins of kernel.h

This file is a new and improved version of the laplace.h file from the exafmm-alpha repo. Due to the enhacements made, the code in this file performs calculations that are significantly more accurate than those in laplace.h.

laplace.h consists of a C++ class inside which all the functions reside, along with a constructor that computes pre-determined values for subsequent computation of the kernels. For example, in the constructor of the `Kernel` class, there is a line like so:
``` cpp
Anm[nm] = oddOrEven(n)/std::sqrt(fnmm*fnpm);
```
This line is computing the value of $$ A^{m}_{n} $$ as is given by Cheng's paper (equation 14). This value is used in M2L and L2L kernels later. However, this value is never directly computed in the new and optimized `kernel.h` file. Instead, it modifies the computation of the `Ynm` vector such that it no longer becomes necessary to involve the `Anm` term in any kernel computation.

### Functions

#### cart2sph

This function converts cartesian co-ordinates in (X,Y,Z) to spherical co-ordinates involving `radius`, `theta` and `phi`. `radius` is simply the square root of the norm of the co-ordinates (norm is defined as the sum of squares of the co-ordinates in `vec.h`).

#### evalMultipole simple implementation

This algorithm calculates the multipole of a cell. It uses spherical harmonics so that net force of the forces inside a sphere and can be estimated on the surface of the sphere, which can then be treated as a single body for estimating forces.

The optimizations that are presented in the `kernel.h` version of this file are quite complex to understand since they look quite different from the original equation.

For code that is still sane and easier to read, head over to the [laplace.h](https://github.com/exafmm/exafmm-alpha/blob/develop/kernels/laplace.h#L48) file in exafmm-alpha. The explanations that follow for now are from this file. We will see how the same functions in `kernel.h` have been modified to make computation faster and less dependent on large number divisions which reduce the accuracy of the system.

The `evalMultipole` function basically tries to populate the `Ynm` array with data that is computed with the following equation:

$$
\rho^{n}Y_{n}^{m}=\sum_{m=0}^{P-1}\sum_{n=m+1}^{P-1}\rho^{n}P_{n}^{m}(x)\sqrt{\frac{(n-m)!}{(n+m)!}}e^{im\beta}
$$

It starts with evaluating terms that need not be computed for every iteration of `n`, and computes those terms in the outer loop itself. The terms in the outer loop correspond to the condition `m=n`. The first of these is the exponential term $$ e^{im\beta} $$.

After this is a curious case of computation of some indexes called `npn` and `nmn`. These are computed as follows:
``` ruby
npn = m * m + 2 * m # case Y n  n
nmn = m * m         # case Y n -n
```

The corresponding index calculation for the inner loop is like this:
``` ruby
npm = n * n + n + m # case Y n  m
nmm = n * n + n - m # case Y n -m
```

This indexes the `Ynm` array. This is done because we are visualizing the Ynm array as a pyramid whose base spans from `-m` to `m` and who height is `n`. A rough visualization of this pyramid would be like so:
```
   -m ---------- m
n  10 11 12 13  14
|    6  7  8  9
|     3  4   5  
|      1   2
V        0
```

The above formulas will give the indexes for each half of the pyramid. Since the values of one half of the pyramid are conjugates of the other half, we can only iterate from `m=0` to `m<P` and use this indexing method for gaining the index of the other half of the pyramid.

Now let us talk about the evaluation of the [Associated Legendre Polynomial](http://mathworld.wolfram.com/AssociatedLegendrePolynomial.html) $$ P^m_{n}(cos(\theta)) $$, where _m_ is the order of the differential equation and _n_ is the degree. The Associated Legendre Polynomial is the solution to the [Associated Legendre Equation](http://mathworld.wolfram.com/AssociatedLegendreDifferentialEquation.html). The Legendre polynomial can be expressed in terms of the [Rodrigues form](https://en.wikipedia.org/wiki/Associated_Legendre_polynomials#Definition_for_non-negative_integer_parameters_.E2.84.93_and_m) for computation without dependence on the simple Legendre Polynomial $$ P_{n} $$. However, due to the factorials and rather large divisions that need to be performed to compute the Associated Legendre polynomial in this form, computing this equation for large values of _m_ and _n_ quickly becomes unstable. Therefore, we use a recurrence relation of the Polynomial in order to compute different values.

The recurrence relation looks like so:

$$
(n-m+1)P^m_{n+1}(x)=x(2n+1)P^m_n(x)-(n+m)P^m_{n-1}(x)
$$

This is expressed in the code with the following line:
``` ruby
p = (x * (2 * n + 1) * p1 - (n + m) * p2) / (n - m + 1)
```
It can be seen that `p` is equivalent to $$ P^{m}_{n+1} $$, `p1` is equivalent to $$ P^{m}_{n} $$ and `p2` is equivalent to $$ P^{m}_{n-1} $$. This convention is followed everywhere in the code.

Observe that the above equation requires the value of _P_ for _n-1_ and _n+1_ to be computed so that the value for _P_ at _n_ can be computed. Therefore, we first set _m=m+1_ and then compute $$ P^m_{m+1} $$ which can be expressed like this:
$$
P^{m}_{m+1}(x)=x(2m+1)P^{m}_{m}(x)
$$

The above equation is expressed by the following line in the code:
``` ruby
p = x * (2 * m + 1) * p1
```

If you read the code closely, you will see that just at the beginning of the `evalMultipole` function, we initialize `p1 = 1` the first time the looping is done. This is because when `p1` at the first instance is identified with `m = 0`, and we substitute `m=0` in this equation:

$$
P^{m}_{m} = (-1)^{m}(2m-1)!(1-x^{2})^{\frac{m}{2}}
$$

We will get $$ P^{m}_{m}(x)=1 $$.

When you look at the code initially, there might be some confusion regarding the significance of having to `rho` terms, `rhom` and `rhon`. This is written because each term of `Ynm` depends on a particular power of `rho` raised to `n`. So just before the inner loop, you can see the line `rhon = rhom`, which basically reduces the number of times that `rho` needs to be multiplied since the outer loop's value of `rho` is already set to what it should be for that particular iteration.

Finally, see that there is a line right after the inner loop which reads like this:
``` ruby
pn = -pn * fact * y
```
This line is for calculating the value of `p1` or $$ P^{m}_{m} $$ after the first iteration of the loop. Since the second factorial term in the equation basically just deals with odd numbers, the calculation of this term can be simplified by simply incrementing by `2` with `fact += 2`. The `y` term in the above equation is in fact `sin(alpha)` (defined at the top of this function). This is because, if you see the original equation, you will see that the third term is $$ (1-x^{2}) $$, and _x_ is in fact `cos(alpha)`. Therefore, using the trigonometric equation, we can say simply substitute the entire term with `y`.

#### evalMultipole optimized implementation

Now that a background of the basic implementation of `evalMultipole` has been established, we can move over to understanding the code that is placed inside the [kernel.h](https://github.com/exafmm/exafmm/blob/learning/2_kernels/kernel.h) file of the `exafmm/learning` branch. This code is more optimized and can compute results with much higher accuracy than the code that is present in the `exafmm-alpha` repo that we previously saw. The main insipiration for this code come's from the Treecode paper posted above.

In this code, most of the stuff relating to indexing and calculation of the powers of `rho` is pretty much the same. However, there are some important changes with regards to the computation of the values that go inside the `Ynm` array. This change is also reflected in the subsequent kernels.

The simplication in computation is basically based on the notion that a P2M kernel will eventually be expanded to M2M and therefore it makes sense to compute some terms that are required for M2M inside P2M itself. In order to see how exactly this will work, consider the line in laplace.h that is used for computing the M2M:
```
M += Cj->M[jnkms] * std::pow(I,real_t(m-abs(m))) * Ynm[nm] * real_t(oddOrEven(n) * Anm[nm] * Anm[jnkm] / Anm[jk]);
```
The above line computes the M2M as given by eq.13 in [Cheng's paper](https://ac.els-cdn.com/S0021999199963556/1-s2.0-S0021999199963556-main.pdf?_tid=262a8f4c-d58d-11e7-82f6-00000aacb360&acdnat=1512018967_7cd88d8da2a5a747344fe9c0619e5563). Now, division and multiplication of such large numbers makes the M2M calculation very unstable if the order and/or degree of the equations is large. Therefore, the new `evalMultipole` simplifies this computation by computing some terms in the P2M stage itself.

In order to understand this, let us see the equation given by Cheng. Let us call this equation `1`:

$$
M^{k}_{j}=\sum_{n=0}^{j}\sum_{m=-n}^{m=n} \frac{O_{j-n}^{k-m}\cdot i^{|k-m|-|k|-|m|}\cdot A^{m}_{n}\cdot A^{k-m}_{j-n}\cdot \rho^{n}\cdot Y^{-m}_{n}(\alpha,\beta)}{A_{j}^{k}}
$$

In the new M2M kernel, the _A_ terms are clubbed together with other terms such that no actual division or multiplication involving these terms takes place the way it does in the laplace.h code. In this regard, we club together $$ O^{k-m}_{j-n} $$ and $$ A^{k-m}_{j-n} $$. You will notice that _O_ is the actually the multipole that is computed in the P2M stage (the `Cj->M[jnkms]` term in the code sample above). Therefore, the actual equation of the P2M kernel using the new evalMultipole method becomes like this:

$$
M^{m}_{n}=\sum^{P-1}_{n=0}\sum^{n}_{m=-n} q_{j}\cdot \rho^{n} \cdot Y^{-m}_{n}(\alpha, \beta)\cdot A^{m}_{n}
$$

The $$ A^{m}_{n} $$ part in eq. (1) will be clubbed with the spherical harmonic _Y_ and will be calculated inside the `evalMultipole` method for every particle in case of P2M and every multipole in case of M2M. Thus since the P2M and M2M have similar behaviour (i.e. grouping of many particles to lesser particles) we can use the same function for both.

In retrospect, inside the evalMultipole method, the part of the above P2M equation after the _q_ is calculated. This equation, upon expansion of spherical harmonics into its consitituents and cancellation of terms with $$ A^{m}_{n} $$, can be simplified as the following equation. Note that the computed value of this equation is what gets stored inside the `Ynm` array.

$$
array^{n}_{m}=\sum^{P-1}_{m=0}\sum^{P-1}_{n=m+1} \frac{\rho^{n} \cdot P^{n}_{m}(x) \cdot e^{im\beta}}{-(n+m)!}
$$

The implementation of this equation inside evalMultipole is a little funny. It has been optimized in such a way that the division operations never happen between numbers that are too big. The factorials are calculated on the fly while the loop is in progress. This can get a little confusing at first since it is not very obvious. The factorial is mainly calculated in two lines of code, [here](https://github.com/exafmm/exafmm/blob/learning/2_kernels/kernel.h#L56) and [here](https://github.com/exafmm/exafmm/blob/learning/2_kernels/kernel.h#L65).

The first line of code reads `rhon /= -(n + m)`. The origin of this is obvious as can be seen from the above equation that is being calculated inside evalMultipole. The division happens after each iteration and there is no stored factorial that is used the way it was in `laplace.h` for reducing the number that needs to be used in the division.

The second line of code reads like `rhom /= -(2 * m + 2) * (2 * m + 1)`. In this case, notice that the LHS has the variable `rhom`. This variable is used only in the outer loop for computation of Ynm (i.e. the case of `n=m`). In order to explain this, consider that the inner loop starts from `n=m+1` and can be rewritten as `rhon /= -(2*m+1)` (at least for the first iteration). When the Ynm value needs to be calculated for the outer loop, we must have the current value of `m` and the value for the next iteration in the `rhom`, therefore we use `2*m+2` as well.

To understand in somewhat more detail, see the following Ruby code that recreates the values of the above indices:
``` ruby
P = 5
m = 0
n = m+1

prod1 = 1
0.upto(P-1) do |m|
  prod = prod1
  (m+1).upto(P-1) do |n|
    puts "m: #{m} n: #{n} 2m+1: #{m+n}"
    prod *= (m + n)
  end
  prod1 *= (2*m + 1)*(2*m + 2)
  puts "P1: #{prod1} 2m+1: #{2*m+1} 2m+2: #{2*m+2}"
end
```

The above code produces the follwoing output:
```
m: 0 n: 1 2m+1: 1
P: 1
m: 0 n: 2 2m+1: 2
P: 2
m: 0 n: 3 2m+1: 3
P: 6
m: 0 n: 4 2m+1: 4
P: 24
P1: 2 2m+1: 1 2m+2: 2
m: 1 n: 2 2m+1: 3
P: 6
m: 1 n: 3 2m+1: 4
P: 24
m: 1 n: 4 2m+1: 5
P: 120
P1: 24 2m+1: 3 2m+2: 4
m: 2 n: 3 2m+1: 5
P: 120
m: 2 n: 4 2m+1: 6
P: 720
P1: 720 2m+1: 5 2m+2: 6
m: 3 n: 4 2m+1: 7
P: 5040
P1: 40320 2m+1: 7 2m+2: 8
P1: 3628800 2m+1: 9 2m+2: 10
```
If you observe the output of code, you can see that the indices being calculated during P1 phase are exactly one greater than the previous phase, which indicates that the factorial value is being calculated properly, incrementally.

## vector.h

This file defines a new custom type for storing 1D vectors called `vec` as a  C++ class. It also defines various functions that can be used on vectors like `norm`, `exp` and other simple arithmetic.

The Ruby implementation of this file is in `vector.rb`.

## exafmm.h

## exafmm2d.h and step1.cxx

Shows a very simple preliminary implementation of the actuall exafmm code. Mostly useful for understanding purpose only.

## step2.cxx
