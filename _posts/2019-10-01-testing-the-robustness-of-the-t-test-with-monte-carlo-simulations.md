Summary
-------

Data arising from biological studies are often clustered, that is, the
design of the study means that some subsets of observations are likely
to have errors that are correlated with each other. For example, study
participants might be sampled in families, or mice housed together in
cages, or patients recruited from different centres. In each of these
cases there is likely to be a factor that influences outcomes all of the
members of one cluster but not members of other clusters, and this
violates the assumption of independence of residual variance that is
required for many statistical procedures.

In these three posts I explore the effect of this on the accuracy of
statistical tests using the t-test as a simple example. First, I
introduce a simulation study to show that the t-test ‘works’, by
calculating its power and Type I error rate in a range of scenarios
where the assumptions underlying the test are met. I then show the
effect of violating the assumption of independence of observations on
Type 1 error, and demonstrate alternative analyses that are valid when
this assumption is not met. Finally I explore the implications for
clustering on power, using a common mouse study design as an example.

The key results are:

-   Observations from units that are not independent must not be
    analysed as if they are independent.
-   Ignoring this can severely inflate error rates.
-   A correct analysis, such as a linear mixed model or simply averaging
    data over clusters before analysis, is often not very difficult.
-   For a mouse experiment: co-housing your mice means that you will
    need more mice than you would if you’d been able to treat them
    randomly and house them individually, with the size of this effext
    depending on the specific experimental design.
-   So this must be considered in your design and power calculation

Statistics by simulation
========================

The real world is complex, but statistical models are usually very
simple. Parsiomy aside, we simplify the world when making statistical
models partly because our results about the validity of statistical
procedures are derived analytically, and these derivations are only
feasible in these simplified cases.

Hence the ‘assumptions’ that accompany, eg, the t-test, with very little
guidance available regarding how well the test works or does not work
when these assumptions are not met.

But with modern computing power there is no need to restrict ourselves
to solving problems analytically; we can investigate how well a test
procedure is likely to work for a given type of dataset by a process
called Monte Carlo simulation.

In short what this means is that we’ll generate thousands of simulated
datasets from a model of our real world data generating process (making
that model as complex as we like) and then we’ll test how well any
proposed analytical method works to estimate the known parameters of the
process.

Testing the t-test
==================

For this example I’m going to test the t-test. My aim is to understand
how well it works when I violate the ‘assumptions’ that accompany it, in
particular the assumption of independence of observations, to know how
well I can compensate with alternative analysis methods.

There are three parts to this exercise.

1.  Here I’ll set up a simulation of a t-test working properly, and test
    its power and Type-I error rate empirically.
2.  In the second part I’ll use data including some shared variance
    between points, to quantify how badly wrong our t-test is.
3.  Finally I’ll test two alternative approaches to correctly analysing
    such data.

Motivating example
==================

The motivation for this example is the analysis of data that comes from
animal studies, particularly studies of mice, where individual mice are
co-housed in cages of say 3-5 animals, and treatment groups might
include 2 or 3 cages.

It is widely known that if treatments are allocated per cage rather than
per mouse, then the experimental unit is the cage and results must be
analysed on that basis. But in practice this is rarely done, analyses
are typically conducted with each mouse considered as an independent
experimental unit.

My intention here is to test the implications for using this incorrect
analysis. The implications are wider than just those conducting simple
t-tests; many more complex analyses (from regression models to
microarray analyses to differential abundance microbiome studies) make
this same assumption that experimental units are independent within
groups, and will suffer the same potential inflated error rates or
biases if they are not.

The data generating process and simulation outcomes
===================================================

First though we’ll focus on the simple t-test (in fact Welch’s t-test as
this is the R default and why not).

I’m going to simulate a mouse experiment looking at the effect of a
treatment (compared to control) on mouse weight, with two independent
groups of ten mice per group. Error variance will be i.i.d. normal with
a standard deviation of 10g in both groups.

In some cases I will simulate data with a real difference of 10g between
the groups. In other cases I will simulate data with no real difference
between the groups. In each case I will be interested in the number of
datasets for which a t-test (or other method) returns a statistically
significant difference between the groups.

In the cases that there is a real difference between the group means, a
‘significant’ t-test is a good thing, and the proportion of datasets for
which the test is statisticially significant estimates the power of the
test.

In the cases where there is no real difference, any ‘significant’ t-test
represents a type-1 error, and so the proportion of these estimates the
type-1 error rate of the test.

Simulating data
===============

First I set the random seed (so these simulations can be exactly
reproduced), and make a function to simulate one dataset under my
assmptions, allowing the sample size, true effect and sd to be set.

``` r
set.seed(06102019) # set at todays date.
simData1 <- function(N=10, b=10, sd=10, a=100){ 
  # 'a' is the control mean, it doesn't matter at all
  x <- rep(c(0,1), each=N) #  x is the treatment group
  y <- rnorm(2*N, a+b*x, sd) #  y is the response
  data.frame(x,y)
}

data1 <- simData1()
```

Now lets plot the data, to see how they look:

``` r
ggplot(data=data1, aes(x=factor(x), y=y)) + 
  geom_boxplot(fill="red") + 
  geom_point(size=3) + 
  theme_bw() + labs(x="Group", y="Weight (g)")
```

![](2019-10-01-testing-the-robustness-of-the-t-test-with-monte-carlo-simulations_files/figure-markdown_github/unnamed-chunk-3-1.png)

Hopefully this looks like we expect. We can run a t-test now, to test
whether the difference between these groups is statistically
significant:

``` r
t.test(data=data1, y~x)
```

    ## 
    ##  Welch Two Sample t-test
    ## 
    ## data:  y by x
    ## t = -3.829, df = 17.061, p-value = 0.001336
    ## alternative hypothesis: true difference in means is not equal to 0
    ## 95 percent confidence interval:
    ##  -27.547271  -7.977848
    ## sample estimates:
    ## mean in group 0 mean in group 1 
    ##        96.92645       114.68901

So in this case the t-test has correctly identified that there is a
significant difference between the two groups.

Let’s try again, this time we’ll just get the plot and the p-value:

``` r
data2 <- simData1()

ggplot(data=data2, aes(x=factor(x), y=y)) + geom_boxplot(fill="red") + geom_point(size=3) + theme_bw() + xlab("Group") + ylab("Weight (g)")
```

![](2019-10-01-testing-the-robustness-of-the-t-test-with-monte-carlo-simulations_files/figure-markdown_github/unnamed-chunk-5-1.png)

``` r
t.test(data=data2, y~x)$p.value
```

    ## [1] 0.04214639

Power and coverage
==================

So from our two reps so far the t-test detects the difference on both
occasions, but how well would we expect it to work on average?

Next we’ll simulate ten thousand of these datasets, and test in how many
experiments we correctly reject the null hypothesis.

This should give us a good idea of how likely the t-test is to work in
our one-off real world experiment, if the difference is between groups
is of the magnitude that we have assumed here (10g).

First, we’ll make another function that that simulates a dataset and
returns a p-value:

``` r
simPvalue <- function(N=10, b=10, sd=10){
  dat <- simData1(N=N, b=b, sd=sd) # get the data
  ttest1 <- t.test(data=dat, y~x) # do the t-test
  ttest1$p.value # return the p-value
}
```

Now we can call our experiment 10000 times under both H1 (true
difference of 10g between groups) and under H0 (true difference of 0
between groups), and store the results:

``` r
repsH1_p <- replicate(10000, simPvalue(b=10) )
repsH0_p <- replicate(10000, simPvalue(b=0) )
```

Power
-----

If we plot a histogram of p-values, we see:

``` r
par(mfrow=c(1,2))

hist(repsH1_p, col="red", breaks=20, main="b=10", xlab="P-value ", ylab="Frequency", freq=TRUE)

hist(repsH0_p, col="blue", breaks=20, main="b=0", xlab="P-value under H0", ylab="Frequency", freq=TRUE)
abline(a=500,b=0)
```

![](2019-10-01-testing-the-robustness-of-the-t-test-with-monte-carlo-simulations_files/figure-markdown_github/unnamed-chunk-8-1.png)

When there is a true difference of 10g between groups then approximately
half of the p-values are less than 0.05, while half are greater than
0.05. We can calculate this more precisely:

``` r
sum(repsH1_p<0.05)
```

    ## [1] 5664

So our null hypothesis would be rejected in 5664 of the 10000
simulations, or about `round(r sum(repsH1_p<0.05)/10000,2)` of the time.

So this experimental design has a 56% chance of detecting a
statistically significant effect, if the true effect size is b=1.

This is the power of the test.

This was such a simple case that we could have used the built-in R
function to calculate the power directly, as follows:

``` r
power.t.test( d=10, sd=10, n=10, power=NULL)
```

    ## 
    ##      Two-sample t test power calculation 
    ## 
    ##               n = 10
    ##           delta = 10
    ##              sd = 10
    ##       sig.level = 0.05
    ##           power = 0.5619846
    ##     alternative = two.sided
    ## 
    ## NOTE: n is number in *each* group

Which hopefully gives us a similar finding!

Type 1 error
------------

We can also ask in how many simulations under H0 we got a ‘significant’
result.

``` r
sum(repsH0_p<0.05)
```

    ## [1] 465

So in 465 of the 10000 studies, or around 5% of the time, there is a
false positive result at p\<0.05. This is reassuring, as is the uniform
distribution of the p-values over \[0,1\], which is the expected
distribution of a p-value under the null hypothesis.

Conclusion
==========

We have shown the t-test ‘works’ as expected for hypothesis testing in
this one simple case, and we have empirically shown that our experiment
was underpowered with N=10 in both groups for detecting an effect size
of 10g, since the power was only 56%.

Next time we’ll use some datasets that do not meet the assumptions under
which the t-test is supposed to operate, to compare how the type 1 error
is affected. Then we’ll look at some alternative analyses that should be
valid using these datasets and compare the power and type-1 error of
these.