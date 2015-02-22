---
layout: post
title: R version of tsfill and xfill combined
tags:
- R
---

Suppose you have an unbalanced panel with daily data observed at the zip code level, with some zip codes (labeled `zip.cd`) not having any records on some days (labeled `cal.dt`). Suppose also that these zip codes are clustered in [DMA's](http://en.wikipedia.org/wiki/Media_market) (labeled `dma.cd`).

Suppose now that you want it balanced so all zip code + date combinations show up, with missing values filled in as needed. And you also want all zip code + date combinations to have a DMA code associated with them.

If this panel were a Stata data set, the `xtset` command would make Stata recognize it as a panel, and then `tsfill, full` would take care of the first half of the problem. This would leave you with some missing values for `dma.cd`, corresponding to zip code + date combinations not observed in the original data set. You would fill these in with [`xfill`](http://www.sealedenvelope.com/stata/) for the complete solution. This is both elegant and well-documented elsewhere.

If your panel is a [data table](http://cran.r-project.org/web/packages/data.table/index.html) `x`, the function below is one way to solve both halves of the problem in one call:

{% highlight r %}
myXTFill <- function(x) {
  dt     <- copy(x)
  xtkeys <- c('zip.cd','cal.dt')
  clkeys <- c('zip.cd','dma.cd')
  xtfull <- CJ(unique(x[,zip.cd]),unique(x[,cal.dt]))
  clfull <- unique(subset(x,select=c(zip.cd,dma.cd)))
  setnames(xtfull,c('V1','V2'),xtkeys)
  setkeyv(xtfull,xtkeys)
  setkeyv(dt,xtkeys)
  setkeyv(clfull,clkeys)
  xtfull <- subset(merge(xtfull,dt,all=TRUE),select=-dma.cd)
  xtfull <- merge(xtfull,clfull,all=TRUE)
}
{% endhighlight %}

Use it at your own risk.
