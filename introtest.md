Hello!
------

This blog will include reflections on my work as a statistician.

This first post is a test that I’ve got github pages working properly.

Let’s add a graph to make sure we can get graphics files working OK.

``` r
library(ggplot2)
x <- rnorm(100)
y <- rep(1:4, 25)
col <- rep(c("red","green", "blue","yellow"),25)
ggplot(data=data.frame(x,y), aes(x=x)) + geom_histogram(aes(fill=col)) + theme_void() + theme(plot.background = element_rect(color="black",fill="black"), legend.position = "none") + facet_wrap(~floor(y))
```

![](introtest_files/figure-markdown_github/testgraph-1.png)
