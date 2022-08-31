---
title: 'Maximum Likelihood Estimation in STATA'
subtitle: ''
summary: Maximum Likelihood Estimation in STATA
authors:
- admin
tags:
- Academic
categories:
- Teaching
date: "2019-01-12T00:00:00Z"
lastmod: "2019-01-12T00:00:00Z"
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Placement options: 1 = Full column width, 2 = Out-set, 3 = Screen-width
# Focal point options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
image:
placement: 2
caption: 'Image credit: [**Unsplash**](https://unsplash.com/photos/CpkOjOcXdUY)'
focal_point: ""
preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
STATA本身有很多estimator是通过MLE方法估计的，例如logit, probit等。在这些模型之外，STATA同时提供了ML syntax来拓展可以估计maximum likelihood模型。下面我们举例子说明ML syntax的用法。

## LF Estimator
当整体样本的Log likelihood可以通过每个样本点的log  likelihood累加得到时，STATA认为这类模型符合线性约束(Linear Form Restriction)，可以使用STATA ML syntax中的lf方法来估计这类模型。我们将使用lf方法来估计四种常见的模型: binary logit, binary probit, OLS, and mixed logit model.

### Logit Model
Log Likelihood formula for Logit Model：
prob(y=1) = exp(xb)/(1+exp(xb))

Code:
'''

    program logit_lf
        args lnfj theta 
        tempvar p 
        local y $ML_y1
        quietly gen `p' = exp(`theta')/(1+exp(`theta'))
        quietly replace `lnfj' = ln(`p') if `y' == 1
        quietly replace `lnfj' = ln(1-`p') if `y' == 0
    end

    sysuse auto, clear
    ml model lf logit_lf (eq1: foreign = weight length)
    ml maximize
'''

几个要注意的点：
+ args 是STATA提供的一种parse的工具，是把positional argument变成指定的local。例如，ml提供的第一个argument通过args变成了local lnfj，而在剩下的程序里，都将以`lnfj'的形式出现。
+ program 中的theta是所有x和系数的linear combination。这是使用lf方法最方便的地方，不需要考虑单独的系数大小，而是把系数的linear combination放在一起考虑。
+ lnfj 是每个observation 的log likelihood.
+ $ML_y1 是第一个公式中的 dependent variable，以global 的形式存在。
+ 所有program 中使用的新的变量要用tempvar
+ ml model lf logit_lf (eq1: foreign = weight length) 这一行是定义模型，即使用lf方法的logit_lf模型，模型中的y是foreign，而解释变量包括了weight和length。
+ ml maximize 这一行是对上述模型进行求解，计算出对应最大loglikelihood的参数。


### Probit Model
类似地，我们可以通过lf方法估计probit模型：
'''

    program probit_lf
        args theta lnfj
        local y $ML_y1
        quietly replace `lnfj' = ln(normal(`theta')) if `y' == 1
        quietly replace `lnfj' = ln(-normal(`theta')) if `y' == 0
    end

'''

### Linear Regression Model

线性回归模型(homoskedastic standard errors)也可以通过lf方法实现：
'''
program ols_lf
args lnfj theta std
local y $ML_y1
quietly replace `lnfj' = ln(normalden(`y',`theta',`std'))
end

    ml model lf ols_lf (eq1: mileage = weight length) (eq2:)
    * ml model lf ols_lf (eq1: mileage = weight length) /eq2 
    ml maximize 
'''

要注意的点：
+ Linear regression model和前面的logit probit模型不同的地方在于：线性回归模型的log likelihood不能通过一个theta(linear combination of variables)来表达，而是由两个系数同时决定的，一个是所有变量的线性组合，一个是残差项的standard deviation。因此，我们需要在写函数，和使用ml model时候都要做出相应的调整。首先，ml model 要加入第二个公式，即(eq2:)。这里，由于模型的假设包括了homoskedasticity,残差项的std不和任何解释变量相关，因此(eq2:)中不需要包括任何解释变量，只需要一个constant变量即可。
+ 当放松homoskedasticity的假设时，我们只需要稍微修改ml model中的eq2部分即可。


### Mixed Binary Logit Model
Logit model虽然直观易估计，但是对于individual preference有比较强的限制。因此，我们可以估计一个mixed binary logit model，来解决这个问题。但是mixed logit model的log likelihood表达式没有closed form solution,因此只能通过simulation来解决，而STATA的ML syntax可以较好的提供simulation的方法。

    // Simulate data for estimation 
    /*
    x~N(0,1)
    beta1 = 1
    beta2 ~ N(1,1)
    epsilon~ extreme value distribution
    y = (beta_1+ beta_2*x +epsilon>0)
    */

    clear
    set obs 1000
    set seed 10101
    gen x = rnormal()
    gen beta_1 = 1
    gen beta_2 = rnormal(1,1)
    gen u = uniform()
    gen epsilon = log(u)-log(1-u)
    gen y = (beta_1 + beta_2*x+epsilon) > 0
    forvalues i = 1/1000{
        quietly gen draw_`i' = uniform()
    }

    program mixed_logit
       args lnfj beta_1 beta_2 std 
       tempvar p sim_f sim_avg_f
       quietly gen `sim_avg_f' = 0
       quietly{
       forvalues i = 1/1000{
           gen `p' = exp(`beta'1+`beta_2'*x +  /// `std'*`draw_i'*x)/(1+exp(`beta'1+`beta_2'*x + `std'*`draw_i'*x))
           gen `sim_f' = `p' if $ML_y1 == 1
           replace `sim_f' = 1- `p' if $ML_y1 == 0
           replace `sim_avg_f` = `sim_avg_f` + `sim_f`/1000
       }
       replace `lnfj' = ln(sim_avg_f)
       }
    end

    ml model lf mixed_logit (beta_1:y = )(beta_2:x)(std:)
    ml maximize

这里需要注意的是：
+ Log likelihood是一千次Simulation的平均。
+ 由于Log likelihood不能够通过所有变量的线性组合来表示，我们需要把系数拆分成三个部分，常数项的系数，变量X的系数的均值，变量X的std.这样的拆分不影响我们使用lf方法，因为样本的log likelihood仍然是由所有的样本的Log likelihood加总得到。

## D0 Estimator
上述lf estimator只适用于样本的log likelihood仍然是由所有的样本的Log likelihood加总得到的情况，当上述条件不成立时，我们需要使用STATA提供的d0 estimator. D0 estimator的特点是我们需要在模型中提供整体样本的log likelihood，而不是每个单独样本点的log likelihood。下面我们用Conditional logit model来说明d0 estimator的用法。

###  Conditional Multinomial Logit Model
Model:
Individual: indexed by i
Choice situation: indexed by j
U_ij = XB+ epsilon, epsilon~extreme value distribution
Log Likelihood if choice j is chosen: exp(x_kb)/\sum(exp(x_jb))

Code:

    cls
    program drop _all
    webuse choice,clear
    gen japan = car==2
    gen europe = car ==3 

    program clogit_sch 
    version 14
    /* 
    model: lnfi = exp(xb)/(\sum exp(xb))
    */
    args todo b lnf 
    tempvar xb e_xb sum_e_xb lnfj
    // local group_id $ML_id 
    local y $ML_y1
    mleval `xb' = `b', eq(1)
    sort id
    quietly{
        gen `e_xb' = exp(`xb')
        bysort id: egen `sum_e_xb' = total(`e_xb')
        gen double `lnfj'   = ln(`e_xb'/`sum_e_xb')
        mlsum `lnf' = `lnfj' if `y' == 1
    }
    end

   
    global ML_id id 
    ml model d0 clogit_d0 (eq1: choice = dealer japan europe) 
    ml check 
    ml max

需要注意的是：
+ mleval: 计算系数`b'带来的linear combination的`xb'。
+ mlsum: 计算所有相关的loglikelihood的总和。
+ mlcheck：用来对ml model进行debug。

和STATA提供改的asclogit的结果进行对比

    webuse choice,clear   
    set more off
    gen japan = car==2
    gen europe = car ==3 
    cd C:/Users/yan/Dropbox/college_entrance_exam/program/2018.12
    program drop _all
    global ML_id id 
    ml model d0 clogit_sch (eq1: choice = dealer japan europe,noconstant) 
    ml check 
    ml max  
    asclogit choice dealer, case(id) alternatives(car)

这里需要指出，asclogit会在后台生成choice specific fixed effect,而我们的estimator需要自己生成这些变量，加入到模型中，但是好处是我们的模型更加flexible。

另外几个很有用的ml command是 ml query, ml report, ml init,和 ml graph。ml query是显示目前处理的模型，包括了每条equation,变量，和系数的初始值。ml report会显示目前的系数vector, gradient vector, negative Hessian, and maximization direction。ml init用来设置模型参数的初始值，例如 ml init 1 2 -2, copy。Ml graph可以用来检测潜在的convergence issue。Newton-Rhapson 是默认的maximization routine，可以通过technique()选项来尝试其他方法，例如bhhh, dfp, and bfgs等。