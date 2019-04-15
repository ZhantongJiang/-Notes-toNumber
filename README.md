# js字符串转数字与小数保留的那点事儿
最近在总结工作中使用频率较高的方法，由于项目中经常会涉及到流量、money等等数字，就想着出一个规范的方法来统一展示在前端页面上的数字以适应产品提出的各种需求。好家伙，这写起来可是发现了巨渊深坑。
```
/**
 * 完成Boolean、Null、String转换为Number类型，并支持以向上取整、向下取整、四舍五入的方式保留指定数位
 * @param {Boolean || Null || String || Number} val 转换目标
 * @param {Number} precision 精确数位，默认保留两位小数，根据项目经验暂定最多支持9位小数
 * @param {String} type 转换方式，默认round四舍五入，同时支持ceil向上取整、floor向下取整
 * @return 返回转换后的数字所对应的字符串
 */
const toNumber = (val, precision=2, type='round') => {
  // 待填充
}
```
我的心路历程：
1. 过滤掉不能转化成有效数字的数据类型。  
最霸道的方式莫过于Number()大法，然而却惊喜的发现Number([])竟然输出0！！！  
那么6种原始类型 + Object作为Number的参数究竟都会输出些什么呢？（以下结果均在Chrome v73.0.3683.86 64位正式版本运行）
* Boolean
![Number_boolean.png](https://user-gold-cdn.xitu.io/2019/4/12/16a0f04231f03518?w=117&h=79&f=png&s=1486)
* Null
![Number_null.png](https://user-gold-cdn.xitu.io/2019/4/12/16a0f05aca72863e?w=111&h=42&f=png&s=1025)
* Undefined
![Number_undefined.png](https://user-gold-cdn.xitu.io/2019/4/12/16a0f06126440918?w=143&h=44&f=png&s=1299)
* Number 毕竟是一家，就不验证了
* String
![Number_string.png](https://user-gold-cdn.xitu.io/2019/4/12/16a0f06c2a04b720?w=145&h=278&f=png&s=5136)
* Symbol（“那么问题来了”系列之一：Symbol到底是个什么玩意）
![Number_symbol.png](https://user-gold-cdn.xitu.io/2019/4/12/16a0f06f77808480?w=445&h=147&f=png&s=7007)
* Object  
array
![Number_object_array.png](https://user-gold-cdn.xitu.io/2019/4/12/16a0f0826b625ade?w=445&h=525&f=png&s=19592)
function
![Number_object_function.png](https://user-gold-cdn.xitu.io/2019/4/12/16a0f0882321a674?w=187&h=145&f=png&s=3661)
date
![Number_object_date.png](https://user-gold-cdn.xitu.io/2019/4/12/16a0f08dfef1af26?w=171&h=116&f=png&s=3577)
math
![Number_object_math.png](https://user-gold-cdn.xitu.io/2019/4/12/16a0f091d5446b42?w=141&h=77&f=png&s=2455)
RegExp
![Number_object_RegExp.png](https://user-gold-cdn.xitu.io/2019/4/12/16a0f0962d1c1cfe?w=169&h=114&f=png&s=3212)
……  
……  
……  
（“那么问题来了”系列之二：Number到底是如何处理的）  
由上所示我们可以发现，虽然数组啊、日期啊在作为Number()的参数的时候可能会返回数字，但从产品的角度上来说这简直太……了吧。所以大胆的将这种情况刨除在外，只支持Boolean、Null、String、Number这四种类型，就有了一下的类型判断：
```
const toNumber = (val, precision=2, type='round') => {
  // 刨除非Boolean、Null、String、Number这四种的类型
  // typeof null === 'object'
  if (typeof val !== 'boolean' && typeof val !== 'string' && typeof val !== 'number' && val !== null) {
    return null
  }

  // 刨除是字符串类型但不能解析成数字的输入
  if (typeof val === 'string' && isNaN(Number(val))) {
    return null
  }

  // 待填充
}
```
2. 坑--toFixed  
在此之前，公司内部前端对于金钱的处理经常使用的是toFixed方法，然而这个方法嘛，你懂得……  
坊间传闻toFixed()采用的是银行家算法，也就是 **"四舍六入五成双"** ：这里“四”是指≤4 时舍去，"六"是指≥6时进上，"五"指的是根据5后面的数字来定，当5后有数时，舍5入1；当5后无有效数字时，需要分两种情况来讲：①5前为奇数，舍5入1；②5前为偶数（包括0），舍5不进。但实际上运行的结果并没有完全按照这个规律来执行（红框中的就是不符合改规律的）：
![tofixed_example.png](https://user-gold-cdn.xitu.io/2019/4/12/16a10bd3244a157f?w=166&h=391&f=png&s=7409)
toFixed()到底是如何运行的呢？参考ECMAScript 3rd Edition (ECMA-262)（[英文版参考文献地址](https://www.ecma-international.org/publications/files/ECMA-ST-ARCH/ECMA-262,%203rd%20edition,%20December%201999.pdf)，[中文版参考文献地址](https://www.w3.org/html/ig/zh/wiki/ES5/%E6%A0%87%E5%87%86_ECMAScript_%E5%86%85%E7%BD%AE%E5%AF%B9%E8%B1%A1#Number.prototype.toFixed_.28fractionDigits.29)）所述应有如下流程（以中文版为例）：
![tofixed_step.png](https://user-gold-cdn.xitu.io/2019/4/12/16a10ec3f1608905?w=1600&h=789&f=png&s=146823)
我成功的懵逼了……  
总而言之言而总之： **toFixed尽量避免使用吧**
3. 坑--浮点数精度  
众所周知，0.1+0.2!==0.3，为什么腻？
```
0.1的二进制表示：0.000110011……0011…… (0011无限循环)
0.2的二进制表示：0.00110011……0011…… (0011无限循环)
```
计算机存储位数有限，因此对于这种无限循环的数字是无法精准的存储，舍来入去的就有了0.1+0.2!==0.3这种违背事实的事实。具体计算原理请参考[如何避开JavaScript浮点数计算精度问题（如0.1+0.2!==0.3）](https://blog.csdn.net/u013347241/article/details/79210840)  
4. [toPrecision()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Number/toprecision)  
numObj.toPrecision(precision)  
这个函数是以定点表示法或指数表示法表示的一个数值对象的字符串表示，四舍五入到 precision 参数指定的显示数字位数。需要注意的是precision是从左到右第一个非0的数位开始计算，另外当结果值对应的数字是10的整数倍且大于1的时候，返回的结果会以科学技术法来表示：
![toPrecision](https://user-gold-cdn.xitu.io/2019/4/13/16a142b79a7e554a?w=393&h=395&f=png&s=46800)
so，只用toPrecision()来处理小数部分效果更佳。![0.1+0.2.png](https://user-gold-cdn.xitu.io/2019/4/15/16a1ecbc31880d06?w=191&h=84&f=png&s=2754)
看，0.1+0.2在数值上就等于0.3了吧（至于为什么参数是12，浏览了n篇博客后了解到的“经验所得”）。  
5. 关于负数的四舍五入  
遵循规则：先将其转为正数再四舍五入，然后再转为负数。  
Finally，上代码（非typeScript版）
```
/**
 * 完成Boolean、Null、String转换为Number类型，并支持以向上取整、向下取整、四舍五入的方式保留指定数位
 * @param {Boolean || Null || String || Number} val 转换目标
 * @param {Number} precision 精确数位，默认保留两位小数，根据项目经验暂定最多支持9位小数
 * @param {String} type 转换方式，默认round四舍五入，同时支持ceil向上取整、floor向下取整
 * @return 返回转换后的数字所对应的字符串
 */
const toNumber = (val, precision=2, type='round') => {
  // 刨除非Boolean、Null、String、Number这四种的类型
  // typeof null === 'object'
  if (typeof val !== 'boolean' && typeof val !== 'string' && typeof val !== 'number' && val !== null) {
    throw('toNumber() can not convert val argument to number')
  }

  // 刨除是字符串类型但不能解析成数字的输入
  if (typeof val === 'string' && isNaN(Number(val))) {
    throw('toNumber() can not convert val argument to number')
  }

  // 校验precision类型及大小：按项目经验首先满足是数字类型，介于0--9之间的整数
  let reg = /^\d{1}$/
  if (typeof precision !== 'number' || !reg.test(precision)) {
    throw('toNumber() precision argument must be Integer and between 0 and 9')
  }

  // 校验type是否为指定类型
  const types = ['round', 'ceil', 'floor']
  if (!types.includes(type)) {
    throw('toNumber() type argument must be one of "round", "ceil", "floor"')
  }

  let target = Math.floor(Number(val)) + Number((Number(val) % 1).toPrecision(12))

  // 处理符号位
  let target_s = target < 0 ? '-' : ''
  target = target < 0 ? -1 * target : target

  let multiple = Math.pow(10, precision - 0)
  if (type === 'ceil') {
    target = Math.ceil(target * multiple) / multiple
  } else if (type === 'floor') {
    target = Math.floor(target * multiple) / multiple
  } else {
    target = Math.round(target * multiple) / multiple
  }

  target = target.toString()
  // 处理小数部分
  let target_int = target.includes('.') ? target.split('.')[0] : target
  let target_decimal = target.includes('.') ? target.split('.')[1] : ''

  if (target_decimal.length < precision) {
    for (let i = 0; i < precision; i++) {
      target_decimal = target_decimal + '0'
      if (target_decimal.length === precision) {
        break
      }
    }
  }

  return target_s + target_int + (precision > 0 ? '.' + target_decimal : '')
}
```
**若有不对烦请批评指正，哪位大神有更好的方法还请指教**
