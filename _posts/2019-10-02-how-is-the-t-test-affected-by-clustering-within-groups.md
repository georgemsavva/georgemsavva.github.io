In the last entry we tested the power and type 1 error rate of the
t-test for a simple two group experiment, using datasets simulated under
conditions that perfectly matched the assumptions of the test.

We showed that the power of our experiment to detect an effect of 10g
was 56%, and that under H0 we got a false positive result 5% of the time
(defining statistical significance at p\<0.05).

Here we will violate of the key assumptions of the t-test to look at how
it affects its performance.

Clustering
==========

Most researchers are aware knows that the t-test depends on data being
normally distributed. Some people know that this means the data are
required to be normally distributed within groups. Fewer people know
that the variances within each group should also be equal. And fewer
still think about the most important assumption, that is that the
observations within each group must be independent of one another. (This
is also true for simple ANOVA which can be thought of as a
generalisation of the t-test to more than two groups).

This assumption of independence means that, conditional on the group
assignment there should be no correlation between any pair of
observations within each group.

However this assumption is frequently not met. For example, if we are
conducting a mouse experiment then mice within the same cage are
unlikely to be completely independent of each other in terms of their
response. Either contamination within the cage (mice affecting each
other in some way) or cage-level factors (such as the position,
temparature, noise, lighting, feed etc) might cause a pair of mice in
the same cage to respond more similarly than a pair of mice from
different cages. This clustering in cages, which is very often clearly
seen in data, would be a direct violation of the assumption of
independence of observations.

It’s also often the case that treatments need to be allocated per cage
rather than per mouse, eg if they are delivered in feed, if or if there
is some ethical or practical reason why mice with different treatments
should not be housed together.

It is accepted that experimental unit in this case is the cage not the
mouse, although most intro stats courses and intro stats textbooks do
not deal with this scenario, considering it an advanced topic despite
complex error structures like this being present more often than not in
biological experiments. Consequently we do not tend to see analyses
conducted in that respect the clustering; many animal studies treat
co-housed and co-treated mice as independent, or do not include any
information on how mice were housed at all.

This is certainly wrong in theory, but how bad is it in practice? We’ll
use a simulation study to find out.

A simulation study of cage effects
==================================

To adapt our previous simulation we need to make two additions. First,
we need to decide on a cage layout. Second we need to say how much
correlation there is likely to be within the cages.

It is common to have 3 or 4 mice per cage, so we’ll split our 10 mice
per group into 2 sets of 3 and 1 of 4.

We’ll introduce a small amount of covariance, such that of our variance
of 10^2, 20% comes from the cage effect and 80% from the individual.
That is the intra-class correlation coefficient is 0.2. (In practice I
typically see intra-class correlation coefficients from mouse
experiments between 0 and 0.5).

Lets adapt our simulation function first:

``` r
simPvalue2 <- function( b=1, sd=1, icc=0.2, cageN = c(3,3,4)){
  sd1 <- sd*sqrt(1-icc) # get the between group and within group sd
  sd2 <- sd*sqrt(icc)
  
  x <- rep(c(0,1), each=sum(cageN)) #  x is the treatment group
  cageID <- rep(1:(length(cageN)*2), c(cageN, cageN))
  dat <- data.frame(x, cageID)
  cageeffect <- data.frame(cageID=1:(length(cageN)*2), ce=rnorm(length(cageN)*2, 0, sd2))
  dat <- merge(dat, cageeffect)
  
  dat$y <- rnorm(2*sum(cageN), b*dat$x + dat$ce, sd1) #  y is the response
  ttest1 <- t.test(data=dat, y~x) # do the t-test
  ttest1$p.value # return
}
```

Now we’ll run 1000 reps to check the distribution of the p-value when
b=0.

``` r
reps2 <- replicate( n=1000, simPvalue2(b=0))

sum(reps2<0.05)
```

    ## [1] 105

This is worrying - we’ve gone from 5% of false positives in the last
simulation to around 11% false positives in this simulation.

How does it work if we use an alternative layout of 2 cages per group
and 5 mice per cage?

``` r
reps3 <- replicate( n=1000, simPvalue2(b=0,cageN=c(5,5)))

sum(reps3<0.05)
```

    ## [1] 167

Now 15% of experiments go wrong at a nominally 5% level.

This is bad because if there was no true effect you’d be seeing a false
positive 15% of the time instead of 5%.

If we use 5 cages of 2 animals each?

``` r
reps4 <- replicate( n=1000, simPvalue2(b=0,cageN=c(2,2,2,2,2)))

sum(reps4<0.05) / 1000
```

    ## [1] 0.078

So with design we are back to 7%, which is better but not perfect.
Looking at the p-value distributions they are no longer uniform:

``` r
par(mfrow=c(1,3))
hist(reps2, col="red", breaks=20, main="Two cages of three\nand one of four", xlab="P-value under H0", ylab="Frequency", freq=TRUE, ylim=c(0,1500))
abline(a=500,b=0)
hist(reps3, col="blue", breaks=20, main="Two cages\nof five", xlab="P-value under H0", ylab="Frequency",freq=TRUE, ylim=c(0,1500))
abline(a=500,b=0)
hist(reps4, col="green", breaks=20, main="Five cages\nof two", xlab="P-value under H0", ylab="Frequency",freq=TRUE, ylim=c(0,1500))
abline(a=500,b=0)
```

![](2019-10-02-how-is-the-t-test-affected-by-clustering-within-groups_files/figure-markdown_github/unnamed-chunk-5-1.png)

Finally, as we vary the amount of within-cage correlation:

``` r
N=1000
errorrate <- function(icc=0.2){
  reps1 <- replicate( n=N, simPvalue2(b=0,icc=icc,cageN=10))
  reps2 <- replicate( n=N, simPvalue2(b=0,icc=icc,cageN=c(5,5)))
  reps3 <- replicate( n=N, simPvalue2(b=0,icc=icc,cageN=c(3,3,4)))
  reps4 <- replicate( n=N, simPvalue2(b=0,icc=icc,cageN=c(2,2,2,2,2)))

c(sum(reps1<0.05) / N,
  sum(reps2<0.05) / N,
  sum(reps3<0.05) / N,
  sum(reps4<0.05) / N)
  }
  
type1 <- as.data.frame(sapply(c(0.1,0.2,0.3,0.4,0.5), FUN=errorrate))
type1 <- as.data.frame(type1)
names(type1) <- c("ICC=0.1", "ICC=0.2","ICC=0.3","ICC=0.4","ICC=0.5")
type1dat <- data.frame(design=c("One cage of 10", "Two cages of 5", "Two cages of 3 and one of 4", "Five cages of 2"),type1)
knitr::kable(type1dat, type="html")
```

| design                      |  ICC.0.1|  ICC.0.2|  ICC.0.3|  ICC.0.4|  ICC.0.5|
|:----------------------------|--------:|--------:|--------:|--------:|--------:|
| One cage of 10              |    0.165|    0.269|    0.359|    0.458|    0.543|
| Two cages of 5              |    0.099|    0.178|    0.206|    0.274|    0.294|
| Two cages of 3 and one of 4 |    0.066|    0.115|    0.141|    0.158|    0.204|
| Five cages of 2             |    0.066|    0.060|    0.086|    0.113|    0.106|

We see that the more variance is shared within cages, and the fewer
cages the mice are divided between, the higher the type 1 error.

Next we’ll consider how to analyse this data correctly and the
implications of clustering for statistical power.