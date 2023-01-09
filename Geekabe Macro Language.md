# Geekabe Macro Language

> Geekabe 宏语言

## 概述

`Geekabe` 的宏是一门解释型语言，用来预处理生成的 `Geekabe` 语言源代码，整体上有点类似于 `php` 生成 `html`。

考虑到尽可能降低宏语言的复杂度，限定了以下约束：

* `<? ?>` 只能在忽略注释后的文件顶部声明，且只能出现一次。这个代码块里通常用于定义和对外暴露宏代码函数，引入其它模块的宏代码函数，等。
* `<?= ?>` 在任意位置出现多次，但只支持宏代码表达式，用于将表达式的值打印成 Geekabe 源码。不同的类型会有不同的打印结果，详见下文。

### `<?= ?>` 打印

整体的打印原则是将变量的值打印成合法源代码。

* 数字、布尔：打印值。
* 字符串：加引号包裹，如果有字符需要转义也会打印成转义形式。
* 对象：打印时会用大括号包裹，打印键名和键值。每个键值根据其类型递归打印。
* 数组：打印时会加中括号包裹，元素之间加逗号。每一个元素根据其类型递归打印。

### 数据类型

`Geekabe` 宏语言语法上尽量和 `Geekabe` 语言保持一致，但因为是解释型语言，数据只支持弱类型且不支持枚举类型，枚举的场景可直接用对象实现。

通过 `type` 关键字定义结构体时，成员属性可定义也可省略。

```ga
<?
type Boy {
	name = 'default',
	say() {
		ret 'hello, I\'m {me.name}'
	}
}
var boy = Boy({
	age: 18
});
console.log(boy.age);
?>
```

需要注意的是，宏语言的第一版暂时还不支持 `type` 关键字。

### 示例

快速斐波拉契数列函数。原理很粗暴，利用宏代码预生成好数列，变成查表的形式。

```ga
<?
fn calc_fab(n) {
  var fab_nums = [], a = 1, b = 1;
  loop i of 0..n {
    if i == 0 || i == 1 {
      fab_nums.push(1)
    } else {
      var tmp = a + b;
      a = b;
      b = tmp;
      fab_nums.push(tmp)
    }
  }
  ret fab_nums;
}
?>
/**
 * FAB_NUMS 用于缓存提前算好的斐波那契数。
 * 下面这行代码等价于: `var FAB_NUMS = [1, 1, 2, 3, 5, 8, 13, 21]`
 */
var FAB_NUMS: uint[] = <?= calc_fab(8) ?>;
pub fn fab(n: uint) {
  if n < FAB_NUMS.length {
    ret FAB_NUMS[n]
  } else {
    ret fab(n - 2) + fab(n - 1)
  }
}
```