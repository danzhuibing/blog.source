title: 逐层解构时间序列ARIMA模型 
date: 2015-12-08 21:07:36
tags:
- 时间序列
- 统计学
- Python

categories: 算法

---

## ARIMA模型的组成结构

**ARIMA(p, d, q)**由三个部分组成：

- **AR(p)**：AR是autoregressive的缩写，表示自回归模型，含义是当前时间点的值等于过去若干个时间点的值的回归——因为不依赖于别的解释变量，只依赖于自己过去的历史值，故称为自回归；如果依赖过去最近的p个历史值，称阶数为p，记为AR(p)模型。
- **I(d)**：I是integrated的缩写，含义是模型对时间序列进行了差分；因为时间序列分析要求平稳性，不平稳的序列需要通过一定手段转化为平稳序列，一般采用的手段是差分；d表示差分的阶数，t时刻的值减去t-1时刻的值，得到新的时间序列称为1阶差分序列；1阶差分序列的1阶差分序列称为2阶差分序列，以此类推；另外，还有一种特殊的差分是季节性差分S，即一些时间序列反应出一定的周期T，让t时刻的值减去t-T时刻的值得到季节性差分序列。
- **MA(q)**：MA是moving average的缩写，表示移动平均模型，含义是当前时间点的值等于过去若干个时间点的预测误差的回归；预测误差=模型预测值-真实值；如果序列依赖过去最近的q个历史预测误差值，称阶数为q，记为MA(q)模型。

I(d)很好理解，将不平稳序列差分得到平稳序列，略过不表。假设我们现在的时间序列已经是平稳的了。

AR(p)模型很好理解，一般而言，时间序列的变量具有时序上的相关性。比如说一条道路的速度，当时间间隔足够小时，上一个时间点速度如果慢，下一个时间点也往往很慢慢。这种内在的相关性，使得我们可以根据最近几个时间点的观测值来预测下几个时间点的值。

{% math %} 
Y_t = \epsilon_t + \beta_1Y_{t-1} + \beta_2Y_{t-2} + ... + \beta_pY_{t-p} 
{% endmath %}


MA(q)模型是我在学习ARIMA时最难理解的地方。说白了，MA模型就是无穷阶AR模型的等价表示。为什么这么说？我们来看一个特殊的无穷阶的AR模型。

{% math %}
Y_t = \epsilon_t + \beta Y_{t-1} - \beta^2Y_{t-2} + \beta^3Y_{t-3} - \beta^4Y_{t-4} + ...        (1)
{% endmath %}


将(1)中的t用t-1替代，得到(2)：


{% math %}
Y_{t-1} = \epsilon_{t-1} + \beta Y_{t-2} - \beta^2Y_{t-3} + \beta^3Y_{t-4} - \beta^4Y_{t-5} + ...        (2)
{% endmath %}


将(2)的左右两边同乘{% math %}\beta{% endmath %}:

{% math %}
\beta Y_{t-1} = \beta\epsilon_{t-1} + \beta^2Y_{t-2} - \beta^3Y_{t-3} + \beta^4Y_{t-4} - \beta^5Y_{t-5} + ...       (3)
{% endmath %}


(1) + (2)，得到(4)：

{% math %}
Y_t = \epsilon_t + \beta\epsilon_{t-1}        (4)
{% endmath %}


式(4)就是MA(1)。对比式(1)和式(4)，我们可以得到MA(1)相当于一个无穷阶的AR模型。同样的，对于任意的q，MA(q)均可以找到一个AR模型与之对应。因此，我们可以得到，时间序列数据归根到底，是可以用统一用AR模型来表示的。

那么，为什么还要MA模型呢？如果只有AR模型，那么一些时间序列必然会需要很高的阶数p来刻画。阶数p，就是待估参数的个数。待估参数越多，需要付出的参数估计代价就越大，所以我们当然希望参数个数越少越好。因此我们自然希望能够用低阶的MA模型来替换高阶的AR模型；反之亦然。

这也可以从自相关系数和偏自相关系数来理解。MA模型的阶数看自相关系数，AR模型的阶数看偏自相关系数。如果自相关系数q阶以后都趋于0，说明是MA(q)模型；这时去看偏自相关系数，必然是无穷阶后都不收敛于0——因为MA模型对应无穷阶的AR模型。同样的，如果偏自相关系数p阶以后都趋于0，说明是AR(p)模型；这时去看自相关系数，必然是无穷阶后都不收敛于0——因为AR模型对应无穷阶的MA模型。

最后，来说说ARMA模型。当自相关系数和偏自相关系数都没有收敛于0，说明这个时间序列不能纯用低阶的AR模型或者纯用低阶的MA模型来解释，需要低阶的AR和低阶的MA模型混合来解释。也可以换个角度来思考，前面提到，任何一个时间序列都可以用纯粹的AR模型来刻画。但是偏自相关系数无穷阶后都不收敛于0，说明只能用一个高阶的AR来解释。但这样的话，阶数太高，待估参数太多，我们就不开心了。所以我们对这个高阶AR模型做分解，分解出一个低阶的AR模型和另一个特殊的高阶AR模型，其中分解出来的高阶AR模型恰好等价于一个低阶的MA模型。于是我们就可以用低阶的AR模型和低阶的MA模型来描述这个时间序列了，这就是ARMA模型。

## ARIMA模型的实践心得
个人觉得ARIMA里面的MA很让人讨厌，因为没有MA，ARIMA就是一个特殊的线性回归模型，可以用大量现成的线性回归包进行参数估计。来自杜克大学的一段关于ARIMA模型的论述很好地阐述了这个点：

> Predicted value of *Y = a constant and/or a weighted sum of one or more recent values of Y and/or a weighted sum of one or more recent values of the errors.*
**If the predictors consist only of lagged values of Y, it is a pure autoregressive (“self-regressed”) model, which is just a special case of a regression model and which could be fitted with standard regression software.**  For example, a first-order autoregressive (“AR(1)”) model for Y is a simple regression model in which the independent variable is just Y lagged by one period (LAG(Y,1) in Statgraphics or Y_LAG1 in RegressIt).  If some of the predictors are lags of the errors, an ARIMA model it is NOT a linear regression model, because there is no way to specify “last period’s error” as an independent variable:  the errors must be computed on a period-to-period basis when the model is fitted to the data.  
出处：http://people.duke.edu/~rnau/411arim.htm


所以我的感觉是，能不用MA就不用MA，这样会让问题的实现变得简单很多，参数估计问题可以自行解决，而不用依赖专业的统计软件，如R、SAS、SPSS等。统计软件固然方便，但是不便于做模型的扩展。比如，我们做时间序列时，除了时间序列的观测值外，还希望加入别的解释变量。举我做过的一个例子，我们想预测一条路未来一个小时的路况，输入数据除了这条路过去的历史路况和当前路况外，还希望把上下游联系紧密的道路的历史路况和当前路况也加进来，即对时间序列做空间上的扩展，得到空间时间序列，在学术上被称为ST-ARIMA（Spatial Temporal ARIMA）。另外，我们知道时间序列的预测，说白了就是认为历史的行为在将来会复现，因此如果有突发情况，肯定是预测不准的。但是如果这种突发情况，我们是能够事先捕获，并作为输入，加入到预测模型里去，就可以很好地改善模型的预测效果；这本质的作用类似用突发情况，对时间序列的训练机做分组，从而让每组的参数不一样。举个具体的例子，北京会时不时实行下单双号限行，尾号还会定期轮换，这些因素都可以作为解释变量加入到模型中去。因此，采用AR模型的话，我们就可以方便地引入各种解释变量，对时间序列数据做拓展，以机器学习的思路来建模。

下面以Python为例介绍对AR模型进行工程实现的思路。

首先，确定下输入数据的格式，以道路速度预测为例，每隔10分钟一个数据，长相如下：

| 时间批次 | 道路索引 | 道路信息（实时速度，profile速度，关联系数）|
| ---- | ---- | ---- | 
| 20150807180200 | 0 |  42, 40, 100 |
| 20150807180200 | 1 |  20, 43, 69 |
| 20150807180200 | 2 |  27, 40, 51 |
| 20150807181200 | 0 |  23, 45, 100 |
| 20150807181200 | 1 |  18, 50, 69 |
| 20150807181200 | 2 |  25, 47, 51 |

下一步，要解决怎么在Python中表示时间序列。显然，我们可以用一个二级dict，让第一级的key是时间批次，第二级的key是道路索引，value是道路信息。

然后，我们要根据时间序列，生成机器学习的标准训练格式(label, feature)，因为我们往往想知道这组数据发生在什么时间，所以再加一个时间字段，输出为(t, label, feature)。伪代码如下：

``` python
ts_dict = 时间序列
AR_P = AR模型的阶数
NUM = 关联道路数
final_list = []
for t in range(start_time, stop_time, step_time):
	if ts_dict.has_key[t]:
		if ts_dict[t].has_key[0]:
			label = 道路信息
			feature = {}
			for x in range(1, AR_P+1):
				t_p = t - step_time * x
				if ts_dict.has_key[t_p]:
					for 道路索引,道路信息 in ts_dict[t_p].iteritems()
					idx = 道路索引 + x * NUM
					feature[idx] = 道路信息
			final_list.append((t, label, feature))
```

这里面需要注意的是，feature的下标idx的公式。

整理成这样的格式后，可以用线性回归参数估计方法，得到AR模型的估计结果。事实上，整理成上述格式以后，我们已经可以不仅限于AR模型了，任何回归和分类模型都可以用上，如逻辑回归、支持向量机、决策树、随机森林、GBDT等。


