---
title: 一个简单的R脚本，不一定对
date: 2016-03-15 18:25:48
category: R
tags: 
- 线性回归
- R语言
---

第一个R语言的脚本。
这个语言不是一个好语言，它的存在合理性是需要商榷的，然而我现在必须要用，因为看起来比较简单... 
等我研究研究scikit-learn之后可能就会无情地抛弃它，非常无情。

{% codeblock lang:r beta_lm.r%}
threshold<-20


cwd<-'~/wde/news/collect/paper/_workspace'

datafile<-sprintf('threshold_%d.csv', threshold)

factors<-c()
for(i in 1:threshold){
    factors<-append(factors, sprintf('loop_%d', i))
}

setwd(cwd)
data<-read.csv(datafile)


for(start in 1:threshold){
    for(end in start:threshold){
        t_factors<-factors[start:end]
        s_formula<-paste('gupcnt', paste(t_factors, collapse='+'), sep='~')
        f_formula<-as.formula(s_formula)
        currentmodel<-lm(f_formula, data)
        currentmodelsummary<-summary(currentmodel)
        currentargs<-c(head(t_factors, n=1), tail(t_factors, n=1), currentmodelsummary$r.squared)
        print(paste(currentargs, sep='\t'))
    }
}
{% endcodeblock %}
