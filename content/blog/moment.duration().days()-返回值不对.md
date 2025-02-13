---
title: moment.duration().days() 返回值不对
date: 2025-02-12T08:39:19.926Z
---

Moment.js 的 duration 对象在处理输入毫秒数时，会先将时间拆分成各个单位（小时、分钟、秒等），然后再进行“归一化（bubble）”，也就是把低单位的溢出部分向上转换到高单位。这个过程并不是简单地按数学公式“总秒数除以单位秒数”来计算，而是分步骤地截断和取余，可能会导致一些舍入和截断误差，从而使你看到的各个单位值与直接数学计算得到的结果有所不同。

下面详细解释一下为什么对于 3188739 秒（即 3188739×1000 毫秒）的输入，调用  
```js
moment.duration(3188739 * 1000).days()
```  
返回 5 而不是直观计算的 6。

### 1. 内部存储与拆分

当你用数字构造一个 duration 对象时，Moment.js 会把传入的毫秒数分别存储在内部的三个属性中：
- `_milliseconds`：表示不足一天的毫秒部分  
- `_days`：表示天数（整数部分）  
- `_months`：表示月份（整数部分，默认 1 个月 = 30 天）

对于纯数字构造（比如 `moment.duration(3188739 * 1000)`），所有时间都最初存储在 `_milliseconds` 中，`_days` 和 `_months` 都为 0。接着会调用内部的 **bubble** 方法，将这些毫秒转换并“归一化”为各个单位。

### 2. Bubble（归一化）过程

归一化过程大致如下（简化版伪代码）：
```js
function bubble() {
  // 先从毫秒中提取秒、分钟、小时
  data.milliseconds = this._milliseconds % 1000;
  var seconds = Math.floor(this._milliseconds / 1000);
  data.seconds = seconds % 60;
  var minutes = Math.floor(seconds / 60);
  data.minutes = minutes % 60;
  var hours = Math.floor(minutes / 60);
  data.hours = hours % 24;
  
  // 把剩下的小时数转换成天数
  this._days += Math.floor(hours / 24);

  // 再将天数转换成月份（按 30 天一个月）
  data.days = this._days % 30;
  data.months = Math.floor(this._days / 30);
}
```

在这个过程中，每一步都是通过整除（`Math.floor`）来提取整数部分，较小单位的剩余部分通过取余得到。

### 3. 截断导致的差异

以 3188739 秒为例，我们直观上可以计算：
- 总秒数：3188739 秒  
- 换算成天：3188739 ÷ 86400 ≈ 36.90625 天  
- 按 30 天一个月：月份 = floor(36.90625 ÷ 30) = 1 个月  
- 剩余天数 = 36.90625 - 30 = 6.90625 天 → 取整数 6 天（直观计算）

然而，Moment.js 并不是先算出 36.90625 天后再整体做除法，而是先把毫秒转换成各个独立单位。由于：
- 在从毫秒转换到秒、分钟、小时时，都会用 `Math.floor` 舍弃小数部分；  
- 当将小时转换为天数时，用的是整数除法（即 `Math.floor`），这时本来“不到一整天”的部分会被截断掉；

因此，实际计算中可能得到的“天数”值会比用总秒数直接除以 86400 得到的整数天略少。也就是说，经过分步拆分后，累计到天的部分可能会比 36 少 1，从而使得：
- 最终计算得出：总天数 = 35（而不是 36）  
- 然后月份 = floor(35 ÷ 30) = 1 个月  
- 剩余天数 = 35 % 30 = 5 天

这种现象正是因为在 bubble 过程中，每一步都对小数部分进行截断，导致最后在“天”的部分丢失了不足 24 小时的那部分累计到天的值。

### 4. 总结

- **直观计算**：如果把 3188739 秒直接换算，约等于 36.90625 天，再按 30 天一个月划分，会得到 1 个月和 6 天（取整数部分）。
- **Moment.js 的归一化**：内部先将毫秒分解成小时、分钟、秒，然后再把小时转换成天时，由于在每步转换中都用整除（截断小数部分），导致累计到天的总数比直观计算少 1。于是，归一化后得到的结果是 1 个月和 5 天，再加上剩下的 21 小时 45 分 39 秒。

这就是为什么调用  
```js
moment.duration(3188739 * 1000).days()
```  
返回 5 而不是 6 的原因。这并非错误，而是 Moment.js 内部归一化算法（bubble 过程）的已知行为和由连续整除导致的截断效应。

### 如何应对

如果你需要得到直观上“总天数余数”的结果，可以自行写转换函数（如前面所示的 customFormatDuration），直接基于总秒数做计算，而不依赖 Moment.js 内部的归一化结果。这样可以避免这种由于分步截断带来的差异。