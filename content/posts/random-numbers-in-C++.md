---
title: "Random Numbers in C++"
date: 2021-04-07T19:38:28-04:00
draft: false
---

# Issues of  `std::rand()`
`std::rand()` is an abomination inherited from C. If you want to be correct, the use case of it so limited you might as well use some other library that does the right thing.

## The Pigeonhole Principle
>  for natural numbers k and m, if n = km + 1 objects are distributed among m sets, then the pigeonhole principle asserts that at least one of the sets will contain at least k + 1 objects.
>
> -- <cite>Wikipedia</cite>

If I were to give you 12 graphic cards and tell you to evenly distribute them to 5 people. You can't,  partially because
nobody can get 12 graphics cards in 2021 and partially due to the pigeonhole principle. 5 does not evenly divide 12,
some people will be extra lucky and manage to get more than 2 cards.


## Broken Library, Blame C
What does `std::rand()` return? From [cppreference.org](https://en.cppreference.com/w/cpp/numeric/random/rand)
> Returns a pseudo-random integral value between 0 and RAND_MAX (0 and RAND_MAX included).

We already have a problem, `RAND_MAX` is implementation-defined, but guaranteed to be at least 32767. If it is actually just
32767, then for the common use case of implementing a dice `std::rand() % 6` will not have a uniform distribution due
to the pigeonhole principle. However, there is more trouble coming. `std::rand()` is usually implemented horribly, on many
platforms the lower bits have terrible randomness. So when you modulo it with a relatively small number, the distribution is
even more skewed.

So, unknown upper range, pigeonhole principle working against us and the low-quality low bits, a perfect trifecta! This is
exactly why `std::rand()` should die in a painful death, sadly it's in C and that means it will never die regardless how
awful it is.[^1]



# What is Randomness?
If you want high-quality random numbers, you must first have a clear understanding of what randomness is in a sufficiently
rigorous context. It's time to cue the mandatory xkcd meme ![comics](https://imgs.xkcd.com/comics/random_number.png). In
some context, this is random. It merely happens that most of the time that's not what you want.

## Probability Distributions
To better understand what we actually want, we must understand the concept of probability distributions. In probability
distributions, we quantify how likely a variable will take on certain values. Imagine two black magic boxes, both spits out
random numbers, one spits 1 and 0 in an almost 50/50 fashion while the other while spits out 1 almost constantly, but sometimes it
spits out 0. Both are random, knowing the previous number that it spits out gives you little to no indication of what the next number will be, yet clearly, they are different. We say that they have two different underlying probability distributions.

In practice, we almost never come up with custom probability distributions, which is the domain of real statisticians. Here I
would introduce just a few common ones that are offered in the C++ standard library.

### Uniform Distribution
This is the distribution most people think of. In a uniform distribution with lower range l and upper range u, all the numbers
in [l, u] are equally likely to occur. Note that C++ both offers `std::uniform_int_distribution` and `std::uniform_real_distribution`
, which limits the output as the names suggest.

### Normal Distribution
In normal distributions, the output has a central tendency and is symmetric. It's almost a crime to explain the normal distribution
without some graphs, so here it is.

![normal distribution](https://cdn.pixabay.com/photo/2013/07/13/12/19/distribution-159626_1280.png)
You can think of the height at a given x value as to how likely it is to occur[^2]. As you can see, there are some values that are
more likely than others, namely the center region; numbers outside of those regions are symmetric. Normal distribution only makes sense
in the infinite context, hence there is no version just for ints. [^3] You can control how much the central tendency happens
by controlling the variance and where the center region lies by setting the mean. If you find yourself needing the normal
distribution, you need to have good working knowledge about basic statistics or consult a real statistician. Don't take
any advice from me other than the advice to talk to a statistician.

### Bernoulli Distribution
The Bernoulli distribution can be seen as a fancier version of a coin toss. Instead of just 50/50, the Bernoulli distribution
allows you to set a custom probability. This is the ideal model for simulating weighted coins, where the odds of the head may
be higher than tails. In C++, the `std::bernoulli_distribution` returns a `bool` indicating a binary event.

There are more distributions available, if you're interested, go to [cppreference](https://en.cppreference.com/w/cpp/numeric/random).


## Source of Randomness
Computers are deterministic machines, so how do non-deterministic events occur from deterministic machines? They don't,
computers use pseudo-random number generators. Which given an initial state, will spit out deterministic results. However,
similar to a hash function, a slight change in input will result in very different outputs. So what we actually need to do,
is figure out how to set up the initial states in a random fashion.

### True Randomness Exists on Modern Hardware
When I said that computers are deterministic machines, that was partially a lie. Modern hardware actually has a built-in
source of randomness that is truly random, based on the environment of the hardware. In C++, this is represented using
`std::random_device`. It has a function call operator that will ask the source of randomness to produce a random value. So
in theory, you could just use it with a probability distribution and you will have true randomness[^4]. However, in practice,
this is really slow so we only use it to set up our desired generator.

### Generators
In the context of C++, given an initial state, a generator will continue to produce random bit patterns[^5]. Most people
are familiar with the linear congruential algorithm, however, the Mersenne twister is preferable in most scenarios. In C++,
both `std::linear_congruential_engine` and `std::mersenne_twister_engine` are provided. Note, you should almost never use them directly,
there are additional parameters than just initial states, understanding them all with their implications requires years of mathematical
training.

# Example
By now, you actually have all the knowledge to correctly generate random numbers in C++! Here's an example generating high
quality dice rolls
```C++
#include <random> // the header where all the magic happens

int main() {
    // true source of randomness, used to seed
    std::random_device rd;
    
    // 64 bit std::mersenne_twister_engine with a 
    // good setting, note the size is big, consider 
    // making it static or put it on heap to avoid 
    // frequent construction 
    std::mt19937_64 engine{rd()};
    
    // 1 to 6 inclusive, producing int
    std::uniform_int_distribution<int> dis{1, 6}; 
    
    // produces one random value in 1 to 6 in an 
    // uniform fashion, call this repeatedly to 
    // get more values
    dis(engine);
}
```

# Tl;Dr
Don't ever use `std::rand()`.


[^1]: Just like c-style strings, perhaps for a future post.
[^2]: This is not strictly correct, technically it's called the probability density.
[^3]: Don't assume you can just round it and call it a day, your choice of rounding will affect the final distribution.
[^4]: Don't assume you can implement cryptography with this, proper cryptography requires lots of training and additional requirements on the generator.
[^5]: Don't assume you can just cast the bit patterns to int or floats, you have no control of its distribution and you will be
all kinds of pain if you cast it to floats since floats contain denormalized numbers, NaN, two 0s, and two infinities.
