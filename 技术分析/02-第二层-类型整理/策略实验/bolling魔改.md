[toc]

>**前言**：
>
>1. 小生将魔改的大致方法根据自己的理解分了几个类，分类标准应该有很多不妥之处且有诸多遗漏之处，还请大家多包容，欢迎建议砸来；另外只收录了部分老板们的策略，未收录的老板们请海涵，收录的老板们认为不妥之处也砸来......
>2. 代码部分省略许多，大多是和基准策略不一样的内容，如果对某个策略感兴趣，详细代码还请老板们亲自翻贴。
>3. 部分策略下有小生的妄语“评价”，还请各位老板海涵，老板们随意一笑即可，不必多看，如老板感觉被冒犯，请留言告知，必给合理解释0 0


**思路**：

1. 过滤开仓条件
2. 止损止盈
3. 动态杠杆
4. 获利方法优化
5. 数学计算方法
6. 自适应
7. 新指标

### 布林本质

```python
df['zscore'] = (df['close'] - df['median']) / df['std']
#原本的开仓条件(多仓)
df['close'] > df['median'] + m * df['std']
#可化为(标准差必为非负数)
(df['close'] - df['median']) / df['std'] > m
#即
df['zscore'] > m
```

所以原始的布林通道突破，本质上是 zscore 的阈值突破, 替换固定阈值 m 为 zscore 绝对值的 n 周期移动最大值, 则通道定义变为:

```python
df['zscore_abs_max'] = df['zscore'].abs().rolling(n, min_periods=1).max().shift(1)
df['upper'] = df['median'] + df['zscore_abs_max'] * df['std']
df['lower'] = df['median'] - df['zscore_abs_max'] * df['std']
```

## 基准策略

```python
def signal_simple_bolling(df, para=[200, 2]):
    """
    :param df:
    :param para: n, m
    :return:

    # 布林线策略
    # 布林线中轨：n天收盘价的移动平均线
    # 布林线上轨：n天收盘价的移动平均线 + m * n天收盘价的标准差
    # 布林线上轨：n天收盘价的移动平均线 - m * n天收盘价的标准差
    # 当收盘价由下向上穿过上轨的时候，做多；然后由上向下穿过中轨的时候，平仓。
    # 当收盘价由上向下穿过下轨的时候，做空；然后由下向上穿过中轨的时候，平仓。
    """

    # ===策略参数
    n = int(para[0])
    m = para[1]

    # ===计算指标
    # 计算均线
    df['median'] = df['close'].rolling(n, min_periods=1).mean()
    # 计算上轨、下轨道
    df['std'] = df['close'].rolling(n, min_periods=1).std(ddof=0)  # ddof代表标准差自由度
    df['upper'] = df['median'] + m * df['std']
    df['lower'] = df['median'] - m * df['std']

    # ===计算信号
    # 找出做多信号
    condition1 = df['close'] > df['upper']  # 当前K线的收盘价 > 上轨
    condition2 = df['close'].shift(1) <= df['upper'].shift(1)  # 之前K线的收盘价 <= 上轨
    df.loc[condition1 & condition2, 'signal_long'] = 1  # 将产生做多信号的那根K线的signal设置为1，1代表做多

    # 找出做多平仓信号
    condition1 = df['close'] < df['median']  # 当前K线的收盘价 < 中轨
    condition2 = df['close'].shift(1) >= df['median'].shift(1)  # 之前K线的收盘价 >= 中轨
    df.loc[condition1 & condition2, 'signal_long'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓

    # 找出做空信号
    condition1 = df['close'] < df['lower']  # 当前K线的收盘价 < 下轨
    condition2 = df['close'].shift(1) >= df['lower'].shift(1)  # 之前K线的收盘价 >= 下轨
    df.loc[condition1 & condition2, 'signal_short'] = -1  # 将产生做空信号的那根K线的signal设置为-1，-1代表做空

    # 找出做空平仓信号
    condition1 = df['close'] > df['median']  # 当前K线的收盘价 > 中轨
    condition2 = df['close'].shift(1) <= df['median'].shift(1)  # 之前K线的收盘价 <= 中轨
    df.loc[condition1 & condition2, 'signal_short'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓

    # 合并做多做空信号，去除重复信号
    df['signal'] = df[['signal_long', 'signal_short']].sum(axis=1, min_count=1, skipna=True)  # 若你的pandas版本是最新的，请使用本行代码代替上面一行
    temp = df[df['signal'].notnull()][['signal']]
    temp = temp[temp['signal'] != temp['signal'].shift(1)]
    df['signal'] = temp['signal']

    # ===删除无关变量
    # df.drop(['median', 'std', 'upper', 'lower', 'signal_long', 'signal_short'], axis=1, inplace=True)
    df.drop(['std', 'signal_long', 'signal_short'], axis=1, inplace=True)

    return df
```

```python
def signal_simple_bolling_para_list(m_list=range(20, 1000+20, 20), n_list=[i / 10 for i in list(np.arange(3, 50+2, 2))]):
    """
    产生布林 策略的参数范围
    :param m_list:
    :param n_list:
    :return:
    """
    print('参数遍历范围：')
    print('m_list', list(m_list))
    print('n_list', list(n_list))

    para_list = []

    for m in m_list:
        for n in n_list:
            para = [m, n]
            para_list.append(para)

    return para_list
```

**策略总体评价(年化收益回撤比)**:BTC: 3.847; ETH: 8.774

## 过滤开仓条件

### 多策略信号过滤

多个模型集成成为的模型叫做集成评估器，组成集成评估器的每个模型都叫做基评估器。通常来说，有三类集成算法：装袋法bagging，提升法boosting和stacking。像下面记录的大多数过滤条件中都是纵向的boosting。

而bagging是将多个参数域一致的策略产生的信号组合，然后根据配置的信号总和的开平仓阈值，生成新的开平仓信号。

举例：

>signal_long_min = 3：开多信号值，信号总和达到此值开多
>signal_close_long = 2 ：平多信号值，持多仓时，信号总和达到此值平多
>signal_short_max = -3 ：开空仓信号值，信号总和达到此值开空
>signal_close_short = -2 ：平空信号值，持空仓时，信号总和达到此值平空

该方法所集成的策略虽然在年化收益上不如基策略，但可非常有效的降低回撤，从而提升年化收益回撤比，尤其适合收益高但回撤也很高的策略。

```python
# 配置开平仓信号值
signal_long_min = 2  # 开多信号值，信号总和达到此值开多
signal_close_long = 0  # 平多信号值，持多仓时，信号总和达到此值平多
signal_close_short = 0  # 平空信号值，持空仓时，信号总和达到此值平空
signal_short_max = -2  # 开空仓信号值，信号总和达到此值开空

signal_list = []  # 初始化信号列表

# 逐个计算策略信号，纳入信号列表
for strategy_name in [
    'adaptboll_with_mtm_v3_add_volume_change_close_VWAP',
    'adaptboll_with_mtm_v3_add_quote_volume_VWAP_WMA',
    'adaptboll_with_mtm_v3_add_quote_volume_change_VWAP_WMA'  # 配置策略列表，这里用丁国庆大佬论坛发的策略1、2、3
]:
    _df = df.copy()
    # 根据策略名称调用策略
    possibles = globals().copy()
    possibles.update(locals())
    method = possibles.get(strategy_name)
    if not method:
        raise NotImplementedError("未找到名为 %s 的策略" % strategy_name)
    _df = method(_df, para=para)
    # 将本次计算出的信号，纳入信号列表
    _df[str(strategy_name)] = _df['signal']
    signal_list.append(_df[str(strategy_name)])

# 合并策略信号集，求信号总和
df_signal = pd.DataFrame(signal_list).T
df_signal.fillna(method='ffill', inplace=True)
df_signal.fillna(0, inplace=True)
df_signal['signal_sum'] = df_signal.apply(lambda x: x.sum(), axis=1)

# 信号总和与配置的开平仓阈值对比，确定实际开平仓信号
df_signal.loc[df_signal['signal_sum'] >= signal_long_min, 'signal'] = 1
df_signal.loc[(df_signal['signal_sum'] < signal_long_min) &
                (df_signal['signal_sum'] > signal_close_long), 'signal'] = np.nan
df_signal.loc[(df_signal['signal_sum'] <= signal_close_long) &
                (df_signal['signal_sum'] >= signal_close_short), 'signal'] = 0
df_signal.loc[(df_signal['signal_sum'] < signal_close_short) &
                (df_signal['signal_sum'] > signal_short_max), 'signal'] = np.nan
df_signal.loc[df_signal['signal_sum'] <= signal_short_max, 'signal'] = -1
df_signal['signal'].fillna(method='ffill', inplace=True)
```

**策略总体评价(年化收益回撤比)**:BTC: 32.9949630525451; ETH: 184.483853735509

### 价格幅度过滤

如果K线短期内快速上涨穿过上布林线一大截，这很有可能是市场的短期异常波动，此时开仓买入会面临较大的回撤风险。

因此开仓前先对比收盘价格相对于均线的涨跌度，如果涨跌度超过某个值（比如7%），那么先不开仓，等价格回落到7%以内再开仓，减少短期快速上升后面临的回撤风险。

>也可以限制收盘价与中轨的距离

```python
# 计算bias
df['bias'] = df['close'] / df['median'] - 1
# ===根据bias，修改开仓时间
df['temp'] = df['signal']

# 将原始信号做多时，当bias大于阀值，设置为空
condition1 = (df['signal'] == 1)
condition2 = (df['bias'] > bias_pct)
df.loc[condition1 & condition2, 'temp'] = None

# 将原始信号做空时，当bias大于阀值，设置为空
condition1 = (df['signal'] == -1)
condition2 = (df['bias'] < -1 * bias_pct)
df.loc[condition1 & condition2, 'temp'] = None

# 原始信号刚开仓，并且大于阀值，将信号设置为0
condition1 = (df['signal'] != df['signal'].shift(1))
condition2 = (df['temp'].isnull())
df.loc[condition1 & condition2, 'temp'] = 0
```

**评价**：从价格维度上过滤开仓条件，虽然简易构造，但确实效果很好。

**策略总体评价(年化收益回撤比)**:BTC: 6.456; ETH: 14.297

### 时间过滤

时间周期提前5分钟

```python
# ===== rule_type 循环的H换算为T，以便于下面统一使用base参数
for rule_type in ['240T', '120T', '60T', '30T', '15T']:
# =====转换为其他分钟数据
period_df = df.resample(rule=rule_type, base=-5, on='candle_begin_time', label='left', closed='left').agg(
    {'open': 'first',
        'high': 'max',
        'low': 'min',
        'close': 'last',
        'volume': 'sum',
        'quote_volume': 'sum',
        'trade_num': 'sum',
        'taker_buy_base_asset_volume': 'sum',
        'taker_buy_quote_asset_volume': 'sum',
        })
```

**策略总体评价(年化收益回撤比)**:BTC: 5.114444; ETH: 10.007993

### 交易量过滤

>数字货币成交分散在各个不同的交易所，成交量数据差别较大，而且各个交易所还刷量，成交量策略比较难开发。--from @李冰

#### 交易量均值

求出过去一段时间成交额的平均值（这里用成交量代替），当当前周期的成交量 > 过去一段时间成交量的均值时，认为这是成交量有效放大，可以做多开仓；当当前周期的成交量 < 过去一段时间成交量的均值时，可以考虑做空。

拿BullPower和BearPower用来衡量市场的多空力量对比从而止盈。在多头市场，人们会更贪婪地在接近高价的地方买入，BullPower 越高则当前 多头力量越强；而在空头市场，人们可能因为恐惧而在接近低价的地方卖出。BearPower 越低则当前空头力量越强。当两者都大于 0 时，反映当前多头力量占据主导地位；两者都小于 0 则反映 空头力量占据主导地位。因此，用BullPower<0作为做多平仓的信号，BearPower>0作为做空平仓的信号。


```python
# 计算均线
df['MAAMT'] = df['volume'].rolling(N).mean()  # 求过去一段时间成交量的均值
df['BullPower'] = df['high'] - df['close'].rolling(N).mean()  # BullPower代表过去一段时间市场的多头力量
df['BearPower'] = df['low'] - df['close'].rolling(N).mean()  # BearPower代表过去一段时间市场的空头力量
# ===计算信号
condition3 = df['volume'] > df['MAAMT'] # 当成交量 > 平均成交量
df.loc[condition1 & condition2 & condition3, 'signal_long'] = 1
condition3 = df['BullPower'] < 0 #当前空头力量占据主导地位
df.loc[condition1 & condition2 & condition3, 'signal_long'] = 0
condition3 = df['volume'] < df['MAAMT'] # 当成交量 < 平均成交量
df.loc[condition1 & condition2 & condition3, 'signal_short'] = -1
condition3 = df['BearPower'] > 0 #当前多头力量占据主导地位
df.loc[condition1 & condition2 & condition3, 'signal_short'] = 0
```

**评价**：简单的指标叠加，改良效果还行，可能是因为N为固定值[20]，如果可以变动应该可以找到更好的效果。

**策略总体评价(年化收益回撤比)**:BTC: 5.739430894; ETH: 10.04435899

#### 主动买入额占总成交额

提出假设：主动买入额占总成交额比例高于一定阈值且该比例在提高时，可能说明上涨趋势更强，而主动买入额占总成交额比例低于一定阈值且该比例在下降时，可能说明下跌趋势更强。

>也可以去掉condition4，效果差距不大

```python
# 计算主动买入占成交额比例及其移动均线
df['taker_buy_rate'] = df['taker_buy_quote_asset_volume'] / df['quote_volume'] * 100
df['taker_buy_rate_mean'] = df['taker_buy_rate'].rolling(n, min_periods=1).mean()
# ===计算信号
condition3 = df['taker_buy_rate'] > df['taker_buy_rate_mean']  # 说明看多的比例提高
condition4 = df['taker_buy_rate'] > taker_buy_rate  # 主动买入比例较高
df.loc[condition1 & condition2 & condition3 & condition4, 'signal_long'] = 1  # 将产生做多信号的那根K线的signal设置为1，1代表做多
condition3 = df['taker_buy_rate'] < df['taker_buy_rate_mean']  # 说明看多的人少了
condition4 = df['taker_buy_rate'] < (100 - taker_buy_rate)  # 主动买入比例处于低位
df.loc[condition1 & condition2 & condition3 & condition4, 'signal_short'] = -1  # 将产生做空信号的那根K线的signal设置为-1，-1代表做空
```

**策略总体评价(年化收益回撤比)**:BTC: 4.995786; ETH: 9.48461

### 新指标过滤

#### 移动平均线过滤

用MA和N根K线之前的MA做比较。

如果原布林信号开多仓时，满足MA大于N根K线之前的MA，则满足条件，正常开仓，否则不开。

同理原布林信号开空仓时，满足MA小于N根K线之前的MA，则满足条件，正常开仓，否则不开。

>除了简单的比较大小外，还可以比较比率、速率等等，也可以拿收盘价和移动平均线进行比较，还可以增加两根移动平均线等等。

```python
# 计算均线
df['median'] = df['close'].rolling(n, min_periods=1).mean()
# ===计算信号
condition3 = df['median'] > df['median'].shift(c)
df.loc[condition1 & condition2 & condition3, 'signal_long'] = 1
condition3 = df['median'] < df['median'].shift(c)
df.loc[condition1 & condition2 & condition3, 'signal_short'] = -1
```

**结果**：回撤降低不少，效果也非常好，均线参数较大。

**评价**：简单的指标叠加，改良效果很明显，在保证通道突破的同时兼顾了趋势方向，减少了许多不必要的开仓。

**策略总体评价(年化收益回撤比)**:BTC: 7.101; ETH: 18.200


#### MACD过滤

```python
fast_n = int(n/1.2) * s   # macd 快线参数
slow_n = int(n/2.6) * s
sig_n = int(n/0.9) * s  # macd 红绿柱参数

# ===计算指标
# 计算均线
df['median'] = df['close'].rolling(n, min_periods=1).mean()
# 计算上轨、下轨道
df['std'] = df['close'].rolling(n, min_periods=1).std(ddof=0)  # ddof代表标准差自由度
df['upper'] = df['median'] + m * df['std']
df['lower'] = df['median'] - m * df['std']

###增加其他列
# DIF - DEA均为正，买入信号参考。
# DIF - DEA均为负，卖出信号参考。
# 调用talib计算MACD指标
df['MACD'], df['MACDsignal'], df['MACDhist'] = talib.MACD(df['close'], fastperiod=fast_n, slowperiod=slow_n,                           signalperiod=sig_n)

# 开仓信号
condition3 = df['MACD'] - df['MACDsignal'] > 0   # 此处新增开仓条件
df.loc[condition1 & condition2 & condition3 , 'signal_long'] = 1
condition3 = df['MACD'] - df['MACDsignal'] < 0  # 此处新增开仓条件
df.loc[condition1 & condition2 & condition3 , 'signal_short'] = -1
```

**策略总体评价(年化收益回撤比)**:BTC: 6.269311778; ETH: 16.55564018

#### 价格波动过滤

使用标准差来判断当前价格否稳定在一个区间内。

```python
df['std'] = df['close'].rolling(n, min_periods=1).std(ddof=0)  # ddof代表标准差自由度
condition3 = df['std'] < (df['open'] * s)  # 价格波动小于预设值比例再开仓
df.loc[condition1 & condition2 & condition3, 'signal_long'] = 1 
condition3 = df['std'] < (df['open'] * s)  # 价格波动小于预设值比例再开仓
df.loc[condition1 & condition2 & condition3, 'signal_short'] = -1 
```

**策略总体评价(年化收益回撤比)**:BTC: 6.23847837389481; ETH: 15.410845206452207

#### 大周期过滤

为了减少在震荡行情中的磨损，通过趋势过滤掉部分开仓，以期达到增收益，减回撤的效果。

##### 斜率

斜率：slope = ma/ma.shift() - 1 以此公式来近似斜率。斜率大于0，判定多头趋势，只做多不做空；斜率小于0，判定空头趋势，只做空不做多

```python
# ===计算指标
n = para[0]
m = para[1]
f = para[2]

# 计算均线
df['median'] = df['close'].rolling(n).mean()
df['ma'] = df['close'].rolling(int(n*f)).mean()
# 计算大周期均线的斜率
df['slope'] = (df['ma'] / df['ma'].shift() - 1) * 100
# 开平仓信号
condition3 = df['slope'] > 0  # 代表趋势向上
df.loc[condition1 & condition2 & condition3, 'signal_long'] = 1 
condition3 = df['slope'] < 0 # 代表趋势向下
df.loc[condition1 & condition2 & condition3, 'signal_short'] = -1
```

**策略总体评价(年化收益回撤比)**:BTC: 5.129; ETH: 12.697

##### 长均线

定义长均线`df[‘long_ma’] = df[‘close’].rolling(n * n_long_ma, min_periods=1).mean()`（在以前的策略开发过程中，发现这个长均线的`n_long_ma`参数普适性不错。这个参数胜在理念，而不是某一个具体值。）

只有收盘价同时大于上轨和长均线才开多仓，只有收盘价同时小于下轨和长均线才开空仓。

```python
# ===策略参数
n_long_ma = int(para[2])

# ===计算指标
# 计算长均线，用长周期趋势过滤短周期的交易。
df['long_ma'] = df['close'].rolling(n * n_long_ma, min_periods=1).mean()

# ===计算策略信号
# 找出做多信号
# 当前K线的收盘价 > 上轨 并且 当前K线的收盘价 > 长均线，只有收盘价同时大于上轨和长均线才开多仓。
condition1 = (df['close'] > df['upper']) & (df['close'] > df['long_ma'])
# 之前K线的收盘价 <= 上轨 或者 之前K线的收盘价 <= 长均线
condition2 = (df['close'].shift(1) <= df['upper'].shift(1)) | (df['close'].shift(1) <= df['long_ma'].shift(1))
# 将产生做多信号的那根K线的signal设置为1，1代表做多
df.loc[condition1 & condition2, 'signal_long'] = 1

# 找出做空信号
# 当前K线的收盘价 < 下轨 并且 当前K线的收盘价 < 长均线，只有收盘价同时小于下轨和长均线才开空仓。
condition1 = (df['close'] < df['lower']) & (df['close'] < df['long_ma'])
# 之前K线的收盘价 >= 下轨 或者 之前K线的收盘价 <= 长均线
condition2 = (df['close'].shift(1) >= df['lower'].shift(1)) | (df['close'].shift(1) >= df['long_ma'].shift(1))
# 将产生做空信号的那根K线的signal设置为-1，-1代表做空
df.loc[condition1 & condition2, 'signal_short'] = -1
```

**策略总体评价(年化收益回撤比)**:BTC: 5.389892; ETH: 9.886433

#### CCI与BBW过滤

利用CCI判断多头市场与否，利用BBW(布林带宽度)对开仓点位进行优化。

```python
df['width_ratio'] = 2 * df['max'] * df['std'] / df['close']

# 计算cci值
df['cci'] = talib.CCI(df.high, df.low, df.close, m)
condition3 = df['cci'] > 0
condition4 = df['width_ratio'] <= width_threshold
df.loc[condition1 & condition2 & condition3 & condition4, 'signal_long'] = 1
condition3 = df['cci'] < 0
condition4 = df['width_ratio'] <= width_threshold
df.loc[condition1 & condition2 & condition3 & condition4, 'signal_short'] = -1
```

**策略总体评价(年化收益回撤比)**:BTC: 6.017; ETH: 16.141

#### 布林带宽度过滤

##### 动态宽度

横有多长，竖有多宽。大行情的到来之前总是伴随不断的震荡，并且震荡幅度不断减小。

开仓条件新增布林带宽度过滤，当当前布林带宽度<前N周期布林带宽度的平均数，才开仓。

```python
df['scope'] = df['max'] * df['std'] # 布林带宽度
# ===计算信号
scope_condition = df['scope'] < df['scope'].rolling(n).mean() # 布林带宽度开仓条件
# 找出开仓信号
df.loc[condition1 & condition2 & scope_condition, 'signal_long'] = 1  # 将产生做多信号的那根K线的signal设置为1，1代表做多
df.loc[condition1 & condition2 & scope_condition, 'signal_short'] = -1  # 将产生做空信号的那根K线的signal设置为-1，-1代表做空
```

**策略总体评价(年化收益回撤比)**:BTC: 3.999910389116530; ETH: 8.518670087794050

##### 静态宽度

```python
# 指标
df['width'] = (df['upper'] - df['lower'])/df['median']
# 找出开仓信号
widthcondition = df['width'] < width
```

**策略总体评价(年化收益回撤比)**:BTC: 6.777607141; ETH: 15.67942223


#### 克罗均线过滤

克罗均线：利用3条均线，看短均线、中均线、长均线之间的相对条件。

克罗均线买入信号：收盘价大于所有均线，并且均线组多头排列(MA10>MA20>MA50)。开空条件则反之，均线组空头排列(MA10< MA20< MA50)。

```python
# 计算克罗均线组
df['ma_s'] = df['close'].rolling(k, min_periods=1).mean()
df['ma_m'] = df['close'].rolling(2*k, min_periods=1).mean()
df['ma_l'] = df['close'].rolling(5*k, min_periods=1).mean()
# ===计算信号
condition3 = df['close'] > df['ma_s']  # 克罗均线组多头排列
condition4 = df['ma_s'] > df['ma_m']
condition5 = df['ma_m'] > df['ma_l']
df.loc[condition1 & condition2 & condition3 & condition4 & condition5, 'signal_long'] = 1 
condition3 = df['close'] < df['ma_s']  # 克罗均线组空头排列
condition4 = df['ma_s'] < df['ma_m']
condition5 = df['ma_m'] < df['ma_l']
df.loc[condition1 & condition2 & condition3 & condition4 & condition5, 'signal_short'] = -1
```

**策略总体评价(年化收益回撤比)**:BTC: 5.913; ETH: 14.582

#### VHF指标过滤

VHF是描述波动率的指标之一，此指标在一定范围内波动，能有效过滤高波动率情况下，布林开仓情况。

1. VHF的值越高，代表目前正处于趋势行情。
2. VHF的值越低，代表目前正处于箱型行情。
3. VHF处于上升状态时，代表股价处于趋势行情。
4. VHF处于下跌状态时，代表股价处于箱型行情。
5. 布林开单限制在箱型行情与趋势行情转换之间。

>布林也可以与其他波动率指标结合

```python
# ===计算指标
df['num'] = (df['close'].rolling(vhf).max() - df['close'].rolling(vhf).min()).abs()
df['den'] = ((df['close'] - df['close'].shift(1))).abs().rolling(vhf).sum()
df['vhf'] = round(df['num'] / df['den'] * 100, 2)
df['vhf_median'] = df['vhf'].rolling(vhf).mean()

condition_ok = df['vhf'] < df['vhf_median']

# 计算均线
df['median'] = df['close'].rolling(m).mean()
# 计算标准差
df['std'] = df['close'].rolling(m).std(ddof=0)  # ddof代表标准差自由度
# 计算上轨、下轨
df['upper'] = df['median'] + n * df['std']
df['lower'] = df['median'] - n * df['std']
# 开多
df.loc[condition1 & condition2 & condition_ok, 'signal_long'] = 1
# 平多
df.loc[condition1 & condition2, 'signal_long'] = 0
# 开空
df.loc[condition1 & condition2 & condition_ok, 'signal_short'] = -1
# 平空
df.loc[condition1 & condition2 'signal_short'] = 0
```

**策略总体评价(年化收益回撤比)**:BTC: 6.641256; ETH: 15.224017

#### 恐慌指数过滤

抓取恐惧贪婪指数，恐惧指数过高，不做多，恐惧指数过低，不做空，恐惧指数过高时，提前点平多仓，恐惧指数过低时，提前点平空仓。

```python
# 获取FGI指数数据，并汇总到原始数据中
import requests, time

# 使用FGI地址获取数据
df2 = pd.read_csv(
    # 该参数为数据在电脑中的路径
    filepath_or_buffer="https://api.alternative.me/fng/?limit=0&format=csv&date_format=cn",  # FGI 获取地址
    # 下参数代表数据的分隔符，csv文件默认是逗号。其他常见的是'\t'
    sep=',',
    # 该参数代表跳过数据文件的的第1行不读入
    skiprows=5,  # 前四行数据混乱
    names=['time', 'FGI', 'classification']  # 指定列名，也可在后面改名df2.rename(columns={0: 'ini'}, inplace=True)  # 重命名
)

'''对print出的数据格式进行修正'''
row_total = df2.shape[0]  # 取行数量，相应的列数量就是df2.shape[1]
df2.drop(index=[row_total - 5, row_total - 4, row_total - 3, row_total - 2, row_total - 1], inplace=True)  # 删除最后五行混乱的数据

'''排序函数, 必须要替换，不然无效'''
df2.sort_values(by=['time'], ascending=1, inplace=True)  # by参数指定按照什么进行排序，acsending参数指定是顺序还是逆序，1顺序，0逆序
df2.reset_index(drop=True, inplace=True)  # 排序之后重新设置索引，就从0开始了，否则是倒序的

'''这个数据在合并时一直报错You are trying to merge on datetime64[ns] and object columns. If you wish to proceed you should use pd.concat， 原来是因为两者time列类型不一样，一个是datetime64[ns]，一个是object（可用df2.info()查看），只需要加一句df['time'] = pd.to_datetime(df['time'])转换成一致的就可以了'''

'''数据合并'''
df2['time'] = pd.to_datetime(df2['time'])
df['time'] = df['candle_begin_time'].dt.date  # 仅保留日期
df['time'] = pd.to_datetime(df['time'])
df = pd.merge(df, df2, how='left', on='time')

df.drop(['time','classification'], axis=1, inplace=True)  # 去掉无用的列
df['FGI'].fillna(value=0, inplace=True)  # 将空的指数FGI补全为0，方便后面计算.因为FGI从2018-02-01才有，若是NA没法判断大于小于，Fill NA/NaN values using 0

# =====作者杨卫祥  加入FGI策略
def signal_Bolling_FGI(df, para=[120, 0.5, 20, 80]):
    index_low = float(para[2])  # 指数小于多少时不开空
    index_high = float(para[3])  # 指数大于多少时不开多
    condition3 = df['FGI'] < index_high  ### 恐慌指数小于指定最大值才开，太高了风险太大，不开
    condition3 = df['FGI'] > index_high  ### 恐慌指数高于一定值平仓
    condition3 = df['FGI'] > index_low  ### 恐慌指数大于指定最小值才开空，低于不开，风险太大
    condition3 = df['FGI'] < index_low  ### 恐慌指数小于一定值平仓
    condition4 = df['FGI'] > 0  ### 排除为0的空值
    df.loc[condition3 & condition4, 'signal_short'] = 0
```

简单布林：**策略总体评价(年化收益回撤比)**:BTC: 7.474; ETH: 18.950

自适应：**策略总体评价(年化收益回撤比)**:BTC: 11.5; ETH: 45.0

#### 日期过滤

日历效应（Calendar Effect）是指金融市场与日期相联系的非正常收益、非正常波动及其他非正常高阶矩。

>除了单纯的周几这个概念外，可以区分上、下午、晚上的时间，相信很多人都在夜晚损失惨重，也可以根据月份和月里的时间等等。

```python
df['weekDay'] = df['candle_begin_time'].dt.weekday+1 # 这里增加是星期几的列，+1是因为返回值是0-6，返回1-7更符合阅读习惯
condition3 = df['weekDay'] != 6 # 周六不做多
condition3 = df['weekDay'] != 1 # 周一不做空
```

最后跑出来是「周一不空+周四不多」，最高，结果是「BTC: 4.6」、「ETH:11.8」

### 大盈利后休市

经历一场大的盈利之后，下一笔交易往往会亏损，导致利润减少。原因可能是因为市场在经历一波大行情之后，在尚未调整好之前会进入一段震荡阶段，这时候开仓往往难以准确判断趋势，稍微一点假突破就会产生亏损。

设定一个阈值，当一笔交易盈利超过该阈值后，下笔交易不开仓。

```python
# 单笔收益大于设定值l的下一笔交易不开仓
_df = df[['signal', 'open']] # 用两根交易的开盘价近似计算收益
_df = _df.dropna()
# 近似计算收益
_df['change'] = _df['open'].pct_change()
_df['change'] = _df['change'].shift(-1)
_df = _df.dropna()
_df['benefit'] = _df['change']*_df['signal']*(1-_df['signal']*0.02) # 考虑到滑点、手续费等因素，计算收益要乘以一定磨损系数，这里设为2%
# 找出要跳过的交易
_df['signal_index'] = _df.index
_df['to_del_signal_1'] = _df['signal_index'].shift(-2)
_df['to_del_signal_2'] = _df['signal_index'].shift(-3)
maxbft_index = _df[_df['benefit'] > l]
# 删除要跳过的交易
try:
    df = df.drop(labels=maxbft_index['to_del_signal_1'].values,axis=0)
    df = df.drop(labels=maxbft_index['to_del_signal_2'].values,axis=0)
except:
    print('回测中没有超过预定收益的交易')
```

**策略总体评价(年化收益回撤比)**:BTC: 5.171486; ETH: 13.7176

## 止损止盈

### 固定止损加两倍固定止盈

突破上轨或跌破下轨后就吃固定的一段利润，到达固定止盈位就平仓。并设置固定止损为固定止盈的一半。

```python
def plus_fixed_stop_lose_double_stop_win(df, hloc, stop_lose_pct):

    stop_win_pct = 2 * stop_lose_pct
    df["signal"] = np.nan
    df.reset_index(drop=True, inplace=True)
    info_dict = {"pre_signal": 0, "stop_lose_price": None, "stop_win_price": None}

    for row in df.itertuples():
        i = getattr(row, "Index")

        if info_dict["pre_signal"] == 0:
            if getattr(row, "signal_long") == 1:
                df.at[i, "signal"] = 1
                pre_signal = 1
                stop_lose_price = getattr(row, hloc) * (1 - stop_lose_pct / 100)
                stop_win_price = getattr(row, hloc) * (1 + stop_win_pct / 100)
                info_dict = {
                    "pre_signal": pre_signal,
                    "stop_lose_price": stop_lose_price,
                    "stop_win_price": stop_win_price,
                }
            elif getattr(row, "signal_short") == -1:
                df.at[i, "signal"] = -1
                pre_signal = -1
                stop_lose_price = getattr(row, hloc) * (1 + stop_lose_pct / 100)
                stop_win_price = getattr(row, hloc) * (1 - stop_win_pct / 100)
                info_dict = {
                    "pre_signal": pre_signal,
                    "stop_lose_price": stop_lose_price,
                    "stop_win_price": stop_win_price,
                }
            else:
                info_dict = {"pre_signal": 0, "stop_lose_price": None, "stop_win_price": None}
        elif info_dict["pre_signal"] == 1:
            if (getattr(row, "signal_long") == 0) | (getattr(row, hloc) < info_dict["stop_lose_price"]) | (getattr(row, hloc) > info_dict["stop_win_price"]):
                df.at[i, "signal"] = 0
                info_dict = {"pre_signal": 0, "stop_lose_price": None, "stop_win_price": None}
            if getattr(row, "signal_short") == -1:
                df.at[i, "signal"] = -1
                pre_signal = -1
                stop_lose_price = getattr(row, hloc) * (1 + stop_lose_pct / 100)
                stop_win_price = getattr(row, hloc) * (1 - stop_win_pct / 100)
                info_dict = {
                    "pre_signal": pre_signal,
                    "stop_lose_price": stop_lose_price,
                    "stop_win_price": stop_win_price,
                }
        elif info_dict["pre_signal"] == -1:
            if (getattr(row, "signal_short") == 0) | (getattr(row, hloc) > info_dict["stop_lose_price"]) | (getattr(row, hloc) < info_dict["stop_win_price"]):
                df.at[i, "signal"] = 0
                info_dict = {"pre_signal": 0, "stop_lose_price": None, "stop_win_price": None}
            if getattr(row, "signal_long") == 1:
                df.at[i, "signal"] = 1
                pre_signal = 1
                stop_lose_price = getattr(row, hloc) * (1 - stop_lose_pct / 100)
                stop_win_price = getattr(row, hloc) * (1 + stop_win_pct / 100)
                info_dict = {
                    "pre_signal": pre_signal,
                    "stop_lose_price": stop_lose_price,
                    "stop_win_price": stop_win_price,
                }
        else:
            raise ValueError("不可能出现其他的情况，如果出现，说明代码逻辑有误，报错")
    return df


def signal_simple_bolling_plus_fixed_stop_lose_fixed_stop_win(df, para):
    df = plus_fixed_stop_lose_double_stop_win(df, 'close', stop_lose_pct)
```

**策略总体评价(年化收益回撤比)**:BTC: 4.536512903013405; ETH: 9.867616236511232

### 通过时间止损止盈

增加candle_num止损参数，根据每次开仓后经历的k线数量进行止损。

>此处candle_num可以认为即是时间参数，且这里是固定时间，如果可以将其动态化效果应该会好些。

```python
count = 0
df['count'] = 0
for i in range(df.shape[0]):
    if df.at[i, 'signal'] == df.at[i, 'signal']:
        count = 1
        df.at[i, 'count'] = count
    elif df.at[i, 'signal'] != df.at[i, 'signal']:
        count += 1
        df.at[i, 'count'] = count
    if count >= candle_num:
        df.at[i, 'signal'] = 0
        count = 0
        df.at[i, 'count'] = count
```

**结果**：改良效果有限。

**评价**：candlenum比较集中，且回测区间小，本来应该重点回测的区间只有6个参数，如果参数密集些效果可能会好一些；此处candlenum相当于固定时间，如果止盈止损动态化效果应该会好很多。

**策略总体评价(年化收益回撤比)**:BTC: 4.202; ETH: 9.356

### 寻找更优平仓位置止盈

#### 寻找提前平仓的阈值参数

>也可以单纯以其他指标作为止盈信号，比如EMA

布林线的上下轨的确定是使用median +/- mstd 来确定，为了寻找上下轨与均线之间合适的出场点，新增一个参数r，r位于区间[0,1)内，做多出场阈值为median + r * mstd，做空出场阈值为median - r * m*std。

```python
# ===计算指标
df['upclose'] = df['median'] + r * m * df['std']
df['downclose'] = df['median'] - r * m * df['std']
# 找出做多平仓信号
condition1 = df['close'] < df['upclose']  # 当前K线的收盘价 < 新出场轨道
condition2 = df['close'].shift(1) >= df['upclose'].shift(1)  # 之前K线的收盘价 >= 新出场轨道
df.loc[condition1 & condition2, 'signal_long'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓
# 找出做空平仓信号
condition1 = df['close'] > df['downclose']  # 当前K线的收盘价 > 新出场轨道
condition2 = df['close'].shift(1) <= df['downclose'].shift(1)  # 之前K线的收盘价 <= 新出场轨道
df.loc[condition1 & condition2, 'signal_short'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓
```

**策略总体评价(年化收益回撤比)**:BTC: 4.486794; ETH: 10.088625

#### 多次穿越上下轨平仓

可以考虑多头多次穿上轨或者空头多次穿下轨之后就提前平仓。原因在于这样显得趋势不是特别大。

```python
# &&找出下穿上轨信号
condition1 = df['close'] > df['upper']
condition2 = df['close'].shift(1) < df['upper'].shift(1)
df.loc[condition1 & condition2, 'signal_over_upper'] = 0  # 上穿上轨signal=0
# &&找出上穿下轨信号
condition1 = df['close'] < df['lower']
condition2 = df['close'].shift(1) > df['lower'].shift(1)
df.loc[condition1 & condition2, 'signal_under_lower'] = 0  # 下穿下轨signal=0
# &&找出k次下穿上轨和k次上穿下轨的信号
df.loc[df['signal'].notnull(), 'pos_group'] = 1  # 开仓和平仓信号都设为1
df['pos_group'] = df['pos_group'].cumsum().fillna(method='ffill')  # 同一段持仓，pos_group相同
condition = df[['signal_over_upper','signal_under_lower']].sum(axis=1, min_count=1, skipna=True).notnull()  # 合并上轨和下轨的信号
df.loc[condition, 'signal_upper_lower'] = 1  # 多次穿轨信号都设为1
cross_count = df['signal_upper_lower'].groupby(df['pos_group']).cumsum()  # 计算在每段持仓中穿轨的次数
condition1 = cross_count == k
condition2 = df['signal'] != 1
condition3 = df['signal'] != -1
condition = condition1 & condition2 & condition3
df.loc[condition, 'signal'] = 0  # 穿轨次数超过k就产生平仓信号
```

**策略总体评价(年化收益回撤比)**:BTC: 5.977940; ETH: 14.704368

### 根据浮盈设置平仓条件止损止盈

当浮盈达到一定程度后，才将平仓线改为上下轨。浮盈情况未达标时，仍然以中轨为平仓线。

考虑到大幅浮盈后，穿过上/下轨平仓后，距离中线可能仍有较大距离，因此以上下轨平仓后，再次突破上下轨时不开仓。

```python
# 实际持仓信号
df['pos'] = df['signal'].shift()
df['pos'].fillna(value=0, inplace=True)  # 将初始行数的pos补全为0

# 记录开仓价格
condition1 = abs(df['pos']) == 1  # 当前周期不为空仓
condition2 = df['pos'] != df['pos'].shift(1)  # 当前周期和上个周期持仓方向不一样。
open_condition = condition1 & condition2
df.loc[open_condition, 'open_price'] = df['open']
df['open_price'].fillna(method='ffill', inplace=True)
df.loc[df['pos'] == 0, 'open_price'] = None

# 盈利大于一定比例k后 平仓条件由穿过布林中轨变为上轨/下轨
condition3 = ((df['close']/df['open_price']-1)*df['pos']) > k
close_condition = condition1 & condition3
df.loc[close_condition, 'upper_lower'] = 1
df.loc[df['pos'] == 0, 'upper_lower'] = 0
df['upper_lower'].fillna(method='ffill', inplace=True)  # upper_lower 标记为1的情况，即是触发转变平仓条件的情况。此时多仓的平仓条件变为从上跌破上轨，而不是跌破中轨。空仓类似

# 做多平仓信号 跌破上轨即平仓
condition4 = df['upper_lower'] == 1
condition5 = df['close'] < df['upper']  # 当前K线的收盘价 < 上轨
condition6 = df['close'].shift(1) >= df['upper'].shift(1)  # 之前K线的收盘价 >= 上轨
df.loc[condition4 & condition5 & condition6, 'upper_lower_long'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓

# 做空平仓信号 穿越下轨即平仓
condition5 = df['close'] > df['lower']
condition6 = df['close'].shift(1) <= df['lower'].shift(1)
df.loc[condition4 & condition5 & condition6, 'upper_lower_long'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓

# 将原始的signal中的重复值删除
temp = df[['signal']]
temp = df[df['signal'].notnull()][['signal']]
temp = temp[temp['signal'] != temp['signal'].shift(1)]
df['signal'] = temp
#
# 将upper_lower_long平仓信号加入
df.loc[df['upper_lower_long'] == 0,'signal'] = 0
```

**策略总体评价(年化收益回撤比)**:BTC: 6.35; ETH: 14.18


## 动态杠杆

根据上下轨距离来决定杠杆的大小。上下轨窄的时候开仓仓位增加，上下轨宽的时候开仓仓位减小。

Signals.py
```python
# &&计算杠杆伸缩比，范围0-1
df['lev_multiplier'] = np.exp((df['bandmin'] - df['band'])/df['bandmin'])
```

Evaluate.py
```python
# ===在开仓时
# 在open_pos_condition的K线，以开盘价计算买入合约的数量。（当资金量大的时候，可以用5分钟均价）
# && 杠杆伸缩比df['lev_multiplier'].shift(1)
df.loc[open_pos_condition, 'contract_num'] = initial_cash * leverage_rate * df['lev_multiplier'].shift(1) / (face_value * df['open'])
```

**策略总体评价(年化收益回撤比)**:BTC: 4.683644; ETH: 10.054776

## 获利方法优化

### 区分趋势与震荡吃利润

根据CMI因子判断当前是趋势市还是震荡市，当市场处于趋势市时，跑自适应趋势布林，当市场处于震荡市时，跑自适应震荡布林。趋势布林价格上穿上轨开多仓，下穿中轨平多仓，下穿下轨开空仓，上穿中轨平空仓。震荡布林开平仓方向跟趋势布林完全相反，上穿上轨开空仓，下穿中轨平空仓，下穿下轨开多仓，上穿中轨平多仓。

```python
# 计算指标
df['cmi'] = 100 * abs(df['close'] - df['close'].shift(n)) / (
        df['high'].rolling(n, min_periods=1).max() - df['low'].rolling(n, min_periods=1).min())

# 趋势市, 上穿上轨, 做多
condition1 = df['close'] > df['upper']  # 当前K线的收盘价 > 上轨
condition2 = df['close'].shift(1) <= df['upper'].shift(1)  # 之前K线的收盘价 <= 上轨
condition3 = df['cmi'] > p
df.loc[condition1 & condition2 & condition3, 'signal_long'] = 1
# 趋势市, 下穿中轨, 平多
condition1 = df['close'] < df['median']  # 当前K线的收盘价 < 中轨
condition2 = df['close'].shift(1) >= df['median'].shift(1)  # 之前K线的收盘价 >= 中轨
df.loc[condition1 & condition2 & condition3, 'signal_long'] = 0 
# 趋势市, 下穿下轨, 做空
condition1 = df['close'] < df['lower']  # 当前K线的收盘价 < 下轨
condition2 = df['close'].shift(1) >= df['lower'].shift(1)  # 之前K线的收盘价 >= 下轨
df.loc[condition1 & condition2 & condition3, 'signal_short'] = -1 
# 趋势市, 上穿中轨, 平空
condition1 = df['close'] > df['median']  # 当前K线的收盘价 > 中轨
condition2 = df['close'].shift(1) <= df['median'].shift(1)  # 之前K线的收盘价 <= 中轨
df.loc[condition1 & condition2 & condition3, 'signal_short'] = 0
# 震荡市, 上穿上轨，做空
condition1 = df['close'] > df['upper']  # 当前K线的收盘价 > 上轨
condition2 = df['close'].shift(1) <= df['upper'].shift(1)  # 之前K线的收盘价 <= 上轨
condition3 = df['cmi'] < p
df.loc[condition1 & condition2 & condition3, 'signal_short'] = -1
# 震荡市, 下穿中轨，平空
condition1 = df['close'] < df['median']  # 当前K线的收盘价 < 中轨
condition2 = df['close'].shift(1) >= df['median'].shift(1)  # 之前K线的收盘价 >= 中轨
df.loc[condition1 & condition2 & condition3, 'signal_short'] = 0
# 震荡市, 下穿下轨, 做多
condition1 = df['close'] < df['lower']  # 当前K线的收盘价 < 下轨
condition2 = df['close'].shift(1) >= df['lower'].shift(1)  # 之前K线的收盘价 >= 下轨
df.loc[condition1 & condition2 & condition3, 'signal_long'] = 1
# 震荡市, 上穿中轨, 平多
condition1 = df['close'] > df['median']  # 当前K线的收盘价 > 中轨
condition2 = df['close'].shift(1) <= df['median'].shift(1)  # 之前K线的收盘价 <= 中轨
df.loc[condition1 & condition2 & condition3, 'signal_long'] = 0
```

**策略总体评价(年化收益回撤比)**:BTC: 4.173237552993820; ETH: 10.879247117262600


### 逆交易吃过滤利润

>很多策略为了避免错误开仓和开仓后震荡选取各种方法来过滤开仓平仓条件，潜在含义即**过滤反原信号方向的运动**，那么为什么不吃掉这个反方向的利润？

@邢不行的策略在**避免价格瞬间走高后开仓回撤大**这样的后果时，根据价格幅度过滤，吃掉他过滤的利润：

```python
# 计算bias
df['bias'] = df['close'] / df['median'] - 1
# ===计算原始布林策略信号
df['temp'] = None
df['temp1'] = None

# 找出做多信号
condition1 = df['close'] > df['upper']  # 当前K线的收盘价 > 上轨
condition2 = df['close'].shift(1) <= df['upper'].shift(1)  # 之前K线的收盘价 <= 上轨
df.loc[condition1 & condition2, 'signal_long'] = 1  # 将产生做多信号的那根K线的signal设置为1，1代表做多
df.loc[condition1 & condition2, 'temp1'] = 1

# 找出做多平仓信号
condition1 = df['close'] < df['median']  # 当前K线的收盘价 < 中轨
condition2 = df['close'].shift(1) >= df['median'].shift(1)  # 之前K线的收盘价 >= 中轨
df.loc[condition1 & condition2, 'signal_long'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓
df.loc[condition1 & condition2, 'temp1'] = 0
df.loc[condition1 & condition2, 'temp'] = 0

# 找出做空信号
condition1 = df['close'] < df['lower']  # 当前K线的收盘价 < 下轨
condition2 = df['close'].shift(1) >= df['lower'].shift(1)  # 之前K线的收盘价 >= 下轨
df.loc[condition1 & condition2, 'signal_short'] = -1  # 将产生做空信号的那根K线的signal设置为-1，-1代表做空
df.loc[condition1 & condition2, 'temp1'] = -1

# 找出做空平仓信号
condition1 = df['close'] > df['median']  # 当前K线的收盘价 > 中轨
condition2 = df['close'].shift(1) <= df['median'].shift(1)  # 之前K线的收盘价 <= 中轨
df.loc[condition1 & condition2, 'signal_short'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓
df.loc[condition1 & condition2, 'temp1'] = 0
df.loc[condition1 & condition2, 'temp'] = 0
# ===将long和short合并为signal
df['signal_short'].fillna(method='ffill', inplace=True)
df['signal_long'].fillna(method='ffill', inplace=True)
df['signal'] = df[['signal_long', 'signal_short']].sum(axis=1)
df['signal'].fillna(value=0, inplace=True)

# 新建信号
condition3 = df['signal'] != df['signal'].shift(1)
condition4 = df['signal'] == 1
condition5 = df['bias'] > bias_pct
df.loc[condition3 & condition4 & condition5, 'temp'] = -2
condition4 = df['signal'] == -1
condition5 = df['bias'] < -1 * bias_pct
df.loc[condition3 & condition4 & condition5, 'temp'] = 2
```

**策略总体评价(年化收益回撤比)**:BTC: 11.1081241; ETH: 26.15165526



## 数学计算方法

>包括移动平均线自身的计算，还可以采取加权移动平均线、指数移动平均线等等。

### 平均差布林策略

布林boll策略里面的标准差std用到了开平方，@李冰很不爽，就拿它开刀。

标准差是离均差平方的算术平均数的算术平方根。平均差是各值与算术平均数之间的平均差异,是各个数据与平均值差值的绝对值的平均数。我认为，标准差经过平方开方处理，更多反应的是价格数据的离散程度，而平均差更能真实反映价格到均线的平均距离。

>关于平均差布林策略和标准差布林策略的差别，我用商品期货20个品种进行了近10年历史的组合回测。直接说结论：单利回测，固定ma_len参数，调整m使得两个策略的交易次数一样的情况下进行比较，盈利因子平均差布林策略比标准差布林策略普遍高0.02，净利润平均差布林策略比标准差布林策略普遍高20%。--from @李冰

```python
# ===计算指标
# 计算均线
df['median'] = df['close'].rolling(n, min_periods=1).mean()
# 计算每根k线收盘价和均线的差值，取绝对数
df['cha'] = abs(df['close'] - df['median'])
# 计算平均差
df['ping_jun_cha'] = df['cha'].rolling(n, min_periods=1).mean()
# 计算上轨、下轨道
df['upper'] = df['median'] + m * df['ping_jun_cha']
df['lower'] = df['median'] - m * df['ping_jun_cha']
```

**策略总体评价(年化收益回撤比)**:BTC: 4.605912; ETH: 10.650483



## 自适应

close > ma(n1) + m*std(n1) 即:(close-ma(n1))/std(n1) > m

close < ma(n1) - m*std(n1) 即:-(close-ma(n1))/std(n1) > m

将布林突破上下轨的条件合并

abs((close-ma(n1))/std(n1)) > m

通过将m转为n的函数达到构建自适应布林的目的，即令 m=max(abs((close-ma(n1))/std(n1)))

此处修改，令m=mean(abs((close-ma(n1))/std(n1)))

由此，只需一个参数即可完成布林通道的构建。

如上，$m=\text{max}_{1< x< n1}|\frac{\text{close}-\text{ma}(n1)}{\text{std}(n1)}|$, 布林通道的构建为$m=\frac{\sum_{i=1}^{n1}|\frac{\text{close}-\text{ma}(n1)}{\text{std}(n1)}|}{n1}$

而后自适应的情况可以如此解释：假设运动员的成绩是m，它是否突破不一定要与固定值比，可以和它过去n周期的最大值比(max), 表现为突破自我，而取mean则为与平均值比，min则为最小值比，以下即为与min值比，因此极为敏感，故设置了范围如上轨外不做空，下轨外不做多。

### 动量自适应布林短线策略

通过对动量指标构建自适应布林通道，可以得到日内级别短线参数。

当价格在母布林上轨上方，不做空。价格在母布林下轨下方不做多。当价格在布林包络线内，多空都做。

```python
n1 = para[0]
n2 = 35*n1
df['median'] = df['close'].rolling(window=n2).mean()
df['std'] = df['close'].rolling(n2, min_periods=1).std(ddof=0)  # ddof代表标准差自由度
df['z_score'] = abs(df['close'] - df['median']) / df['std']
df['m'] = df['z_score'].rolling(window=n2).mean()
df['upper'] = df['median'] + df['std'] * df['m']
df['lower'] = df['median'] - df['std'] * df['m']

condition_long = df['close'] > df['upper']
condition_short = df['close'] < df['lower']

df['mtm'] = df['close'] / df['close'].shift(n1) - 1
df['mtm_mean'] = df['mtm'].rolling(window=n1, min_periods=1).mean()

# 基于价格atr，计算波动率因子wd_atr
df['c1'] = df['high'] - df['low']
df['c2'] = abs(df['high'] - df['close'].shift(1))
df['c3'] = abs(df['low'] - df['close'].shift(1))
df['tr'] = df[['c1', 'c2', 'c3']].max(axis=1)
df['atr'] = df['tr'].rolling(window=n1, min_periods=1).mean()
df['avg_price'] = df['close'].rolling(window=n1, min_periods=1).mean()
df['wd_atr'] = df['atr'] / df['avg_price']

# 参考ATR，对MTM指标，计算波动率因子
df['mtm_l'] = df['low'] / df['low'].shift(n1) - 1
df['mtm_h'] = df['high'] / df['high'].shift(n1) - 1
df['mtm_c'] = df['close'] / df['close'].shift(n1) - 1
df['mtm_c1'] = df['mtm_h'] - df['mtm_l']
df['mtm_c2'] = abs(df['mtm_h'] - df['mtm_c'].shift(1))
df['mtm_c3'] = abs(df['mtm_l'] - df['mtm_c'].shift(1))
df['mtm_tr'] = df[['mtm_c1', 'mtm_c2', 'mtm_c3']].max(axis=1)
df['mtm_atr'] = df['mtm_tr'].rolling(window=n1, min_periods=1).mean()

# 参考ATR，对MTM mean指标，计算波动率因子
df['mtm_l_mean'] = df['mtm_l'].rolling(window=n1, min_periods=1).mean()
df['mtm_h_mean'] = df['mtm_h'].rolling(window=n1, min_periods=1).mean()
df['mtm_c_mean'] = df['mtm_c'].rolling(window=n1, min_periods=1).mean()
df['mtm_c1'] = df['mtm_h_mean'] - df['mtm_l_mean']
df['mtm_c2'] = abs(df['mtm_h_mean'] - df['mtm_c_mean'].shift(1))
df['mtm_c3'] = abs(df['mtm_l_mean'] - df['mtm_c_mean'].shift(1))
df['mtm_tr'] = df[['mtm_c1', 'mtm_c2', 'mtm_c3']].max(axis=1)
df['mtm_atr_mean'] = df['mtm_tr'].rolling(window=n1, min_periods=1).mean()

indicator = 'mtm_mean'

# mtm_mean指标分别乘以三个波动率因子
df[indicator] = df[indicator] * df['mtm_atr']
df[indicator] = df[indicator] * df['mtm_atr_mean']
df[indicator] = df[indicator] * df['wd_atr']

# 对新策略因子计算自适应布林
df['median'] = df[indicator].rolling(window=n1).mean()
df['std'] = df[indicator].rolling(n1, min_periods=1).std(ddof=0)  # ddof代表标准差自由度
df['z_score'] = abs(df[indicator] - df['median']) / df['std']
# df['m'] = df['z_score'].rolling(window=n1).max().shift(1)
# df['m'] = df['z_score'].rolling(window=n1).mean()
df['m'] = df['z_score'].rolling(window=n1).min().shift(1)
df['up'] = df['median'] + df['std'] * df['m']
df['dn'] = df['median'] - df['std'] * df['m']

# 突破上轨做多
condition1 = df[indicator] > df['up']
condition2 = df[indicator].shift(1) <= df['up'].shift(1)
condition = condition1 & condition2
df.loc[condition, 'signal_long'] = 1

# 突破下轨做空
condition1 = df[indicator] < df['dn']
condition2 = df[indicator].shift(1) >= df['dn'].shift(1)
condition = condition1 & condition2
df.loc[condition, 'signal_short'] = -1

# 均线平仓(多头持仓)
condition1 = df[indicator] < df['median']
condition2 = df[indicator].shift(1) >= df['median'].shift(1)
condition = condition1 & condition2
df.loc[condition, 'signal_long'] = 0

# 均线平仓(空头持仓)
condition1 = df[indicator] > df['median']
condition2 = df[indicator].shift(1) <= df['median'].shift(1)
condition = condition1 & condition2
df.loc[condition, 'signal_short'] = 0

df.loc[condition_long, 'signal_short'] = 0
df.loc[condition_short, 'signal_long'] = 0
```


**策略总体评价(年化收益回撤比)**:BTC: 15.527636; ETH: 70.825722

### 多空双标布林带的宽度过滤自适应

亏损最大的10笔里有8笔是做空导致的，因此是否可以限制做空的开仓次数来提高整个策略的回撤比？故做多和做空分别用其最好的参数，但存在过拟合的嫌疑。

```python
df = data.copy()

time_rule_type = (df.iloc[1]['candle_begin_time'] - df.iloc[0]['candle_begin_time']).seconds / 60
str_t = str(int(time_rule_type))
dict_time_interval = {'15': 58, '30': 29, '60': 15, '120': 7, '240': 4}
n1 = dict_time_interval[str_t]

# n1 = para[0]
n2 = para[0] * n1
n3 = para[1] * n1

# ===做多代码区域
df['median'] = df['close'].rolling(window=n2).mean()
df['std'] = df['close'].rolling(n2, min_periods=1).std(ddof=0)  # ddof代表标准差自由度
df['z_score'] = abs(df['close'] - df['median']) / df['std']
df['m'] = df['z_score'].rolling(window=n2).mean()
df['upper'] = df['median'] + df['std'] * df['m']
df['lower'] = df['median'] - df['std'] * df['m']

# ===计算信号
df['scope'] = df['std'] * df['m']  # 布林带宽度

condition_long = df['high'] > df['upper'].shift(1)

# 将原来的condition_short 用布林带的宽度过滤
scope_condition = df['scope'] < df['scope'].rolling(n1).mean()
condition_short = df['low'] < df['lower'].shift(1)
condition_short = condition_short & scope_condition

df['mtm'] = df['close'] / df['close'].shift(n1) - 1
df['mtm_mean'] = df['mtm'].rolling(window=n1, min_periods=1).mean()

# 基于价格atr，计算波动率因子wd_atr
df['c1'] = df['high'] - df['low']
df['c2'] = abs(df['high'] - df['close'].shift(1))
df['c3'] = abs(df['low'] - df['close'].shift(1))
df['tr'] = df[['c1', 'c2', 'c3']].max(axis=1)
df['atr'] = df['tr'].rolling(window=n1, min_periods=1).mean()
df['avg_price'] = df['close'].rolling(window=n1, min_periods=1).mean()
df['wd_atr'] = df['atr'] / df['avg_price']

# 参考ATR，对MTM指标，计算波动率因子
df['mtm_l'] = df['low'] / df['low'].shift(n1) - 1
df['mtm_h'] = df['high'] / df['high'].shift(n1) - 1
df['mtm_c'] = df['close'] / df['close'].shift(n1) - 1
df['mtm_c1'] = abs(df['mtm_h'] - df['mtm_l'])
df['mtm_c2'] = abs(df['mtm_h'] - df['mtm_c'].shift(1))
df['mtm_c3'] = abs(df['mtm_l'] - df['mtm_c'].shift(1))
df['mtm_tr'] = df[['mtm_c1', 'mtm_c2', 'mtm_c3']].max(axis=1)
df['mtm_atr'] = df['mtm_tr'].rolling(window=n1, min_periods=1).mean()

# 参考ATR，对MTM mean指标，计算波动率因子
df['mtm_l_mean'] = df['mtm_l'].rolling(window=n1, min_periods=1).mean()
df['mtm_h_mean'] = df['mtm_h'].rolling(window=n1, min_periods=1).mean()
df['mtm_c_mean'] = df['mtm_c'].rolling(window=n1, min_periods=1).mean()
df['mtm_c1'] = abs(df['mtm_h_mean'] - df['mtm_l_mean'])
df['mtm_c2'] = abs(df['mtm_h_mean'] - df['mtm_c_mean'].shift(1))
df['mtm_c3'] = abs(df['mtm_l_mean'] - df['mtm_c_mean'].shift(1))
df['mtm_tr'] = df[['mtm_c1', 'mtm_c2', 'mtm_c3']].max(axis=1)
df['mtm_atr_mean'] = df['mtm_tr'].rolling(window=n1, min_periods=1).mean()

indicator = 'mtm_mean'

# mtm_mean指标分别乘以二个波动率因子

df[indicator] = df[indicator] * df['mtm_atr_mean']
df[indicator] = df[indicator] * df['wd_atr']

# 对新策略因子计算自适应布林
df['median'] = df[indicator].rolling(window=n1).mean()
df['std'] = df[indicator].rolling(n1, min_periods=1).std(ddof=0)  # ddof代表标准差自由度

# 增加交易量因子
df['volume_median'] = df['volume'].rolling(window=n1).mean()
df['volume_std'] = df['volume'].rolling(n1, min_periods=1).std(ddof=0)
df['z_score'] = abs(df[indicator] - df['median']) * abs(df['volume'] - df['volume_median']) / (
            df['std'] * df['volume_std'])

df['m'] = df['z_score'].rolling(window=n1).min().shift(1)
df['up'] = df['median'] + df['std'] * df['m']
df['dn'] = df['median'] - df['std'] * df['m']

# 突破上轨做多
condition1 = df[indicator] > df['up']
condition2 = df[indicator].shift(1) <= df['up'].shift(1)

condition = condition1 & condition2
df.loc[condition, 'signal_long'] = 1

# 均线平仓(多头持仓)
condition1 = df[indicator] < df['median']
condition2 = df[indicator].shift(1) >= df['median'].shift(1)
condition = condition1 & condition2
df.loc[condition, 'signal_long'] = 0

df.loc[condition_short, 'signal_long'] = 0

# ===做空代码区域
df['median'] = df['close'].rolling(window=n3).mean()
df['std'] = df['close'].rolling(n3, min_periods=1).std(ddof=0)  # ddof代表标准差自由度
df['z_score'] = abs(df['close'] - df['median']) / df['std']
df['m'] = df['z_score'].rolling(window=n3).mean()
df['upper'] = df['median'] + df['std'] * df['m']
df['lower'] = df['median'] - df['std'] * df['m']

# ===计算信号
df['scope'] = df['std'] * df['m']  # 布林带宽度

condition_long = df['high'] > df['upper'].shift(1)

# 将原来的condition_short 用布林带的宽度过滤
scope_condition = df['scope'] < df['scope'].rolling(n1).mean()
condition_short = df['low'] < df['lower'].shift(1)
condition_short = condition_short & scope_condition

df['mtm'] = df['close'] / df['close'].shift(n1) - 1
df['mtm_mean'] = df['mtm'].rolling(window=n1, min_periods=1).mean()

# 基于价格atr，计算波动率因子wd_atr
df['c1'] = df['high'] - df['low']
df['c2'] = abs(df['high'] - df['close'].shift(1))
df['c3'] = abs(df['low'] - df['close'].shift(1))
df['tr'] = df[['c1', 'c2', 'c3']].max(axis=1)
df['atr'] = df['tr'].rolling(window=n1, min_periods=1).mean()
df['avg_price'] = df['close'].rolling(window=n1, min_periods=1).mean()
df['wd_atr'] = df['atr'] / df['avg_price']

# 参考ATR，对MTM指标，计算波动率因子
df['mtm_l'] = df['low'] / df['low'].shift(n1) - 1
df['mtm_h'] = df['high'] / df['high'].shift(n1) - 1
df['mtm_c'] = df['close'] / df['close'].shift(n1) - 1
df['mtm_c1'] = abs(df['mtm_h'] - df['mtm_l'])
df['mtm_c2'] = abs(df['mtm_h'] - df['mtm_c'].shift(1))
df['mtm_c3'] = abs(df['mtm_l'] - df['mtm_c'].shift(1))
df['mtm_tr'] = df[['mtm_c1', 'mtm_c2', 'mtm_c3']].max(axis=1)
df['mtm_atr'] = df['mtm_tr'].rolling(window=n1, min_periods=1).mean()

# 参考ATR，对MTM mean指标，计算波动率因子
df['mtm_l_mean'] = df['mtm_l'].rolling(window=n1, min_periods=1).mean()
df['mtm_h_mean'] = df['mtm_h'].rolling(window=n1, min_periods=1).mean()
df['mtm_c_mean'] = df['mtm_c'].rolling(window=n1, min_periods=1).mean()
df['mtm_c1'] = abs(df['mtm_h_mean'] - df['mtm_l_mean'])
df['mtm_c2'] = abs(df['mtm_h_mean'] - df['mtm_c_mean'].shift(1))
df['mtm_c3'] = abs(df['mtm_l_mean'] - df['mtm_c_mean'].shift(1))
df['mtm_tr'] = df[['mtm_c1', 'mtm_c2', 'mtm_c3']].max(axis=1)
df['mtm_atr_mean'] = df['mtm_tr'].rolling(window=n1, min_periods=1).mean()

indicator = 'mtm_mean'

# mtm_mean指标分别乘以二个波动率因子

df[indicator] = df[indicator] * df['mtm_atr_mean']
df[indicator] = df[indicator] * df['wd_atr']

# 对新策略因子计算自适应布林
df['median'] = df[indicator].rolling(window=n1).mean()
df['std'] = df[indicator].rolling(n1, min_periods=1).std(ddof=0)  # ddof代表标准差自由度

# 增加交易量因子
df['volume_median'] = df['volume'].rolling(window=n1).mean()
df['volume_std'] = df['volume'].rolling(n1, min_periods=1).std(ddof=0)
df['z_score'] = abs(df[indicator] - df['median']) * abs(df['volume'] - df['volume_median']) / (df['std'] * df['volume_std'])

df['m'] = df['z_score'].rolling(window=n1).min().shift(1)
df['up'] = df['median'] + df['std'] * df['m']
df['dn'] = df['median'] - df['std'] * df['m']

# 突破下轨做空
condition1 = df[indicator] < df['dn']
condition2 = df[indicator].shift(1) >= df['dn'].shift(1)
condition = condition1 & condition2
df.loc[condition, 'signal_short'] = -1

# 均线平仓(空头持仓)
condition1 = df[indicator] > df['median']
condition2 = df[indicator].shift(1) <= df['median'].shift(1)
condition = condition1 & condition2
df.loc[condition, 'signal_short'] = 0

df.loc[condition_long, 'signal_short'] = 0
```

**策略总体评价(年化收益回撤比)**:BTC: 68.354131; ETH: 230.724223


## 新指标

构建新指标代替布林线本身的开平仓条件，找到更有效率的指标。

### 动量指标改造轨道

对因子的叠加，既可以叠加到因子本身上，也可以叠加到轨道上，对轨道本身进行调整。采用动量因子mtm对轨道改造的方式来进行（同样可以用波动率等因子对轨道改造）。

只有在动量较强的时候开仓是有效开仓的概率会更大，同时让开仓稍微提前一些以增大盈利，那么就可以让动量更大的时候突破更容易。为实现这个效果，只需要对上下轨乘上mtm的倒数即可，以实现mtm越大，轨道越向下，越容易开仓。

```python
# ===计算指标
# 计算mtm
df['mtm'] = df['close'] / df['close'].shift(n)

# 计算均线
df['median'] = df['close'].rolling(n, min_periods=1).mean()
# 计算上轨、下轨道
df['std'] = df['close'].rolling(n, min_periods=1).std(ddof=0)  # ddof代表标准差自由度
df['upper'] = df['median'] + m * df['std']
df['lower'] = df['median'] - m * df['std']

# 轨道调整
# df['median'] = df['median'] / df['mtm']
df['upper'] = df['upper'] / df['mtm']
df['lower'] = df['lower'] / df['mtm']
```

**策略总体评价(年化收益回撤比)**:BTC: 5.608; ETH: 9.277

### 卡夫曼均线

原布林策略均线为简单均线，其优点是适合判断长期趋势变化。但币圈是快速变化的市场，简单均线对市场波动会出现明显的滞后，如果让中轨对市场变化感知更灵敏或许能提高收益。

卡夫曼均线(AMA)根据一段时间内价格的波动率会自行调整。当市场出现盘整、趋势不明显时AMA趋向慢速均线；当市场波动较大，趋势明显时AMA趋向快速均线。用卡夫曼均线替换布林线中轨，能让策略更快感知市场变化。

```python
from numba import njit
import numpy as np

@njit
def cal_ama(_s, _a, _c):
    s, a, c = _s.copy(), _a.copy(), _c.copy()
    for i in range(s.shape[0] - 1):
        a[i + 1] = a[i] + s[i + 1] ** 2 * (c[i + 1] - a[i])  # AMA = 上一个AMA + 系数 * (价格 - 上一个AMA)
    return a


def kaufman(df, n, fast_n=2, slow_n=30):
    # ===计算价格效率
    n = int(n)
    df["c"] = df["close"].diff(n).abs()  # 价格变动值: 当前价格 与 前n根k线(1根)的 价格差 的绝对值
    df["v"] = df["close"].diff().abs().rolling(n).sum()  # 价格波动值: n根k线内价格变动绝对值之和
    df["e"] = df["c"] / (df["v"] + 1e-9)  # 效率系数=价格变动值/价格波动值，分母加极小值防止除零，出现NaN

    # ===计算平滑系数
    # 用一系列移动平均速度描述平滑系数
    # 根据价格所占权重定义快速和慢速趋势速度系数
    # 如2天的平均可定义为快速，30天为慢速，快速趋势系数为 2/(2+1)，慢速趋势系数为 2/(30+1)
    # 2/(2+1)=0.6667; 2/(30+1)=0.0645; 0.6667-0.0645=0.6022
    # 平滑系数 = 效率系数 * (快速-慢速) + 慢速 = 0.6022 * 效率系数 + 0.0645
    df["s"] = (2 / (fast_n + 1) - 2 / (slow_n + 1)) * df["e"] + 2 / (slow_n + 1)  # 平滑系数

    # ===计算AMA
    # 系数 = 平滑系数 * 平滑系数
    # AMA = 上一个AMA + 系数 * (价格 - 上一个AMA)
    df["AMA"] = np.NaN
    # 定义从有 平滑系数 开始的 第一个 AMA 值为  价格*系数
    df.loc[n, "AMA"] = df.loc[n, "close"] * df.loc[n, "s"] * df.loc[n, "s"]

    df.loc[n:len(df), "AMA"] = cal_ama(
        df.loc[n:len(df), "s"].values,
        df.loc[n:len(df), "AMA"].values,
        df.loc[n:len(df), "close"].values
    )
    del df['c'], df['v'], df['e'], df['s']

    return df

# =====简单布林 均线改卡夫曼 策略
def kfm_bolling(df, para=None):

    # ===策略参数
    if para is None:
        para = [200, 2]

    n = int(para[0])
    m = para[1]

    # ===计算指标
    # 计算均线
    df = kaufman(df, n, fast_n=2, slow_n=30)
    df['median'] = df['AMA']

    # 计算上轨、下轨道
    df['std'] = df['median'].rolling(n, min_periods=1).std(ddof=0)  # ddof代表标准差自由度
    df['upper'] = df['median'] + m * df['std']
    df['lower'] = df['median'] - m * df['std']
```

**策略总体评价(年化收益回撤比)**:BTC: 5.338497; ETH: 14.680064



### 缠论

```python
# ===计算分型顶
condition1 = df["high"].shift(1) > df["high"]
condition2 = df["high"].shift(1) > df["high"].shift(2)
condition3 = df["low"].shift(1) > df["low"]
condition4 = df["low"].shift(1) > df["low"].shift(2)
df.loc[condition1 & condition2 & condition3 & condition4, 'vf'] = 1
df.loc[condition1 & condition2 & condition3 & condition4, 'vh'] = df["high"]

# ===计算分型底
condition5 = df["high"].shift(1)  < df["high"]
condition6 = df["high"].shift(1)  < df["high"].shift(2)
condition7 = df["low"].shift(1) < df["low"]
condition8 = df["low"].shift(1) < df["low"].shift(2)
df.loc[condition5 & condition6 & condition7 & condition8, 'vf'] = -1
df.loc[condition5 & condition6 & condition7 & condition8, 'vl'] = df["low"]

# ===计算分型通道
df["vh"] = df["vh"].fillna(method='ffill')
df["vl"] = df["vl"].fillna(method='ffill')

# 计算上下轨道
df['upper'] = df["vh"].rolling(n, min_periods=1).max()
df['lower'] = df["vl"].rolling(n, min_periods=1).min()

df['bias'] = df['close'] / df['median'] - 1

# 找出做多信号
condition1 = df['close'] > df['upper']  # 当前K线的收盘价 > 上轨
condition2 = df['close'].shift(1) <= df['upper'].shift(1)  # 之前K线的收盘价 <= 上轨
df.loc[condition1 & condition2, 'signal_long'] = 1  # 将产生做多信号的那根K线的signal设置为1，1代表做多

# 找出做多平仓信号
condition1 = df['close'] < df['median']  # 当前K线的收盘价 < 中轨
condition2 = df['close'].shift(1) >= df['median'].shift(1)  # 之前K线的收盘价 >= 中轨
df.loc[condition1 & condition2, 'signal_long'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓

# 找出做空信号
condition1 = df['close'] < df['lower']  # 当前K线的收盘价 < 下轨
condition2 = df['close'].shift(1) >= df['lower'].shift(1)  # 之前K线的收盘价 >= 下轨
df.loc[condition1 & condition2, 'signal_short'] = -1  # 将产生做空信号的那根K线的signal设置为-1，-1代表做空

# 找出做空平仓信号
condition1 = df['close'] > df['median']  # 当前K线的收盘价 > 中轨
condition2 = df['close'].shift(1) <= df['median'].shift(1)  # 之前K线的收盘价 <= 中轨
df.loc[condition1 & condition2, 'signal_short'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓
```

**策略总体评价(年化收益回撤比)**:BTC: 5.6282; ETH: 13.1187

### RSI+止盈止损

RSI是相对强弱指标，通过一定时间内上涨和下跌幅度之和的比率来分析市场的后续趋势

>此类摆动指标还有诸如ROC、%D%K、%R等等。

```python
df['rtn'] = df['close'].diff()
df['median'] = df['close']
df['lower'] = df['close']
df['upper'] = df['close']
df['up'] = np.where(df['rtn'] > 0, df['rtn'], 0)
df['down'] = np.where(df['rtn'] < 0, abs(df['rtn']), 0)

df['A'] = df['up'].rolling(n).sum()
df['B'] = df['down'].rolling(n).sum()

df.dropna(inplace = True)
df.reset_index(drop = True, inplace = True)
df['rsi'] = df['A'] / (df['A'] + df['B']) * 100
# 生成做多做空的初步信号
df.loc[df['rsi'] > (50 + p), 'signal_long'] = 1
df.loc[df['rsi'] < (50 - p), 'signal_short'] = -1
df['signal'] = np.nan
info_dict = {'pre_signal':0, 'stop_lose_price':None, 'stop_profit_price':None}
for i in range(df.shape[0]):
    if info_dict['pre_signal'] == 0:
        if df.at[i, 'signal_long'] == 1:
            df.at[i, 'signal'] = 1
            pre_signal = 1
            stop_lose_price = df.at[i, 'close'] * ( 1 - stop_loss_pct / 100)
            stop_profit_price = df.at[i, 'close'] * ( 1 + stop_profit_pct / 100)
            info_dict = {'pre_signal' : pre_signal, 'stop_lose_price':stop_lose_price, 'stop_profit_price':stop_profit_price}
        elif df.at[i, 'signal_short'] == - 1:
            df.at[i, 'signal'] = -1
            pre_signal = -1
            stop_lose_price = df.at[i, 'close'] * ( 1 + stop_loss_pct / 100)
            stop_profit_price = df.at[i, 'close'] * ( 1 - stop_profit_pct / 100)
            info_dict = {'pre_signal' : pre_signal, 'stop_lose_price':stop_lose_price, 'stop_profit_price':stop_profit_price}
        else:
            info_dict = {'pre_signal' : 0, 'stop_lose_price': None, 'stop_profit_price':None}

    elif info_dict['pre_signal'] == 1:
        # 当前做多，遇到清仓或价格超出止盈或止损线时清空
        if (df.at[i, 'signal_long'] == 0) or (df.at[i, 'close'] < info_dict['stop_lose_price']) or (df.at[i, 'close'] > info_dict['stop_profit_price']):
            df.at[i, 'signal'] = 0
            info_dict = {'pre_signal': 0, 'stop_lose_price': None, 'stop_profit_price': None}
        if df.at[i, 'signal_short'] == -1:
            df.at[i, 'signal'] = -1
            pre_signal = -1
            stop_lose_price = df.at[i, 'close'] * ( 1 + stop_loss_pct / 100)
            stop_profit_price = df.at[i, 'close'] * ( 1 - stop_profit_pct / 100)
            info_dict = {'pre_signal' : pre_signal, 'stop_lose_price':stop_lose_price, 'stop_profit_price':stop_profit_price}


    elif info_dict['pre_signal'] == - 1:
        # 当前做空，遇到清仓或价格超出止盈或止损线时清空

        if (df.at[i, 'signal_short'] == 0) or (df.at[i, 'close'] > info_dict['stop_lose_price']) or (df.at[i, 'close'] < info_dict['stop_profit_price']):
            info_dict = {'pre_signal' : 0, 'stop_lose_price': None, 'stop_profit_price':None}
            df.at[i, 'signal'] = 0
        if df.at[i, 'signal_long'] == 1:
            df.at[i, 'signal'] = 1
            pre_signal = 1
            stop_lose_price = df.at[i, 'close'] * ( 1 - stop_loss_pct / 100)
            stop_profit_price = df.at[i, 'close'] * ( 1 + stop_profit_pct / 100)
            info_dict = {'pre_signal' : pre_signal, 'stop_lose_price':stop_lose_price, 'stop_profit_price':stop_profit_price}
    else:
        raise ValueError('error')
```

**评价**：参数较多，单纯的RSI指数表现没有布林线好，加上止损止盈后有一点效果。

**策略总体评价(年化收益回撤比)**:BTC: 9.949465902; ETH: 11.49210565

### ATR+均线作布林带

ATR：平均真实波动。ATR是用来测量价格波动性的指标。它不仅能够表明价格运行的趋势，还可以表现股价波动的幅度大小。该指标的波动程度小的时候,市场处于震荡区域，当价格突破此区域时表示可能会有趋势行情发生。

1. 可以考虑用某均线作为中轨，均线+ATR作为上轨，均线-ATR作为下轨，突破上下轨开仓。
2. 布林的平仓，可能由于中轨距离当前价格太远，导致大幅回撤也是一个比较大的问题，同样可以考虑使用ATR来平仓。当价格快速拉升之后，或者高位震荡，当某一根反向K线出现，而且大于这段时间的振幅，说明趋势可能结束，立即平仓不要等破中轨。

```python
# ===计算指标
# 计算均线
df['median'] = df['close'].rolling(n, min_periods=1).mean()
# 计算TR
df['h-l']=abs(df['high']-df['low'])
df['h-c']=abs(df['high']-df['close'].shift(1))
df['l-c']=abs(df['low']-df['close'].shift(1))
df['max']=df[['h-l','h-c','l-c']].max(axis=1)
#计算ATR
df['atr'] = df['max'].rolling(n, min_periods=1).mean()
df['zscore'] = (df['close'] - df['median']) / df['atr']
df['zscore_final']=df['zscore']/(abs(df['zscore']).rolling(n).max().shift(1))

# ===计算信号
# 找出做多信号
condition1 = df['zscore_final'].shift()<1
condition2 = df['zscore_final']>1
df.loc[condition1 & condition2, 'signal_long'] = 1

# 找出做多平仓信号
condition1 = df['close'] < df['median']  # 当前K线的收盘价 < 中轨
condition2 = df['close'].shift(1) >= df['median'].shift(1)  # 之前K线的收盘价 >= 中轨
condition3 = df['close'].shift(1)-df['close'] > m*df['atr'] 
df.loc[condition1 & condition2 | condition3, 'signal_long'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓

# 找出做空信号
condition1 = df['zscore_final'].shift()>-1
condition2 = df['zscore_final']<-1
df.loc[condition1 & condition2, 'signal_short'] = -1  # 将产生做空信号的那根K线的signal设置为-1，-1代表做空

# 找出做空平仓信号
condition1 = df['close'] > df['median']  # 当前K线的收盘价 > 中轨
condition2 = df['close'].shift(1) <= df['median'].shift(1)  # 之前K线的收盘价 <= 中轨
condition3 = df['close']-df['close'].shift(1) > m*df['atr'] 
df.loc[condition1 & condition2 | condition3, 'signal_short'] = 0  # 将产生平仓信号当天的signal设置为0，0代表平仓
```

**策略总体评价(年化收益回撤比)**:BTC: 5.017984; ETH: 9.878536
