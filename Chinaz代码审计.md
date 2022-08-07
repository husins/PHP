## 前言

刚刚接触代码审计，PHPStorm的动调使用的还不是很流畅，开始审计的是一个非常简单的`chinaz`，但是即便跟着别的师傅的思路走，很多地方的函数调用还是弄不明白，可以说是非常的痛苦。代码审计真的很吃基本功，要对危险函数敏感，这也是我CTF一直打不好的原因吧，面对危险函数太不敏感了，一定要培养自己扎实的基本功。**黄沙百战穿金甲，不破楼兰誓不还**。

## 思路

- 通读全文法：寻找入口文件，通读全文代码，查找跟踪可控变量
- 回溯法：定位危险函数，查找跟踪可控变量

## 审计

### 结构分析

这个`cms`的体量非常小，没有使用`MVC`框架，因此分析起来不是很困难，具体结构如图：

![image-20210812164820999](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812164820999.png)

### 入口分析

这是个但单入口的的`cms`，直接进入`index.php`开始分析：

- 首先包含了公共文件的函数和方法
- 对包含的`view.php`文件中的类进行实例化
- 第五行声明一个数组`$data`
- 接着，对`page`参数使用了`filter`函数，进行处理，并赋值到`$data[page]`中
- 将经过处理`page`，传递到类内方法`echoContent`中

![image-20210812165422589](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812165422589.png)

跟踪一下`filter`（ctrl + 鼠标左键，直接跳转了）：

- `filter`是`common.php`的一个函数，用来将传递的参数进行过滤，将`.` 替换为空，由此可以避免`../../`这样的路径穿越

![image-20210812170030076](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812170030076.png)

- 这里感觉可以绝对路径进行绕过，打断点，验证思路，确实可以通过绝对路径绕过

![image-20210812174538837](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812174538837.png)

跟踪一下`echoContent`:

- 将传递进来的参数，拼接`views/`和`.php`后放入到`loadFile`中
- 然后调用三个类内方法，处理`loadFile`返回值

![image-20210812170540742](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812170540742.png)

跟踪一个`loadFile`:

- 判断文件是否存在：
  - 如果文件不存在，就调用`wirte_log`,并返回报错页面路径（`$cfg_basedir`,跟踪一下是提前声明好的返回路径的全局变量）
  - 如果文件存在，通过`fopen`、`fread`、`fclose`读取文件信息，并返回

![image-20210812170806745](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812170806745.png)

跟踪一下`write_log`：

- 将访问错误路径，格式化为字符串，写入日志中

![image-20210812171550206](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812171550206.png)	

到这里入口分析完毕

### seay扫描

- 个人感觉Seay不是很好用，我这套cms，本身是一套AWD题目，有个预留后门，这都扫不出来，玩个屁。这也间接告诉我们，**不要迷信工具。**或者说，重构一下Seay的正则。
- 但是对于现在我这个菜逼来说够用了，以后接触到更好的工具咱再换
- 这里Seay扫描的结果是危险的函数，还需要手工去验证

![image-20210812172156048](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812172156048.png)

### 漏洞分析

#### `action.php`文件包含

根据Seay的扫描结果对`action.php`进行分析：

- 包含两个公用函数和方法
- 获取`post`型`page`传参，使用`filter`函数进行处理，并拼接后缀`.php`
- 将所有`post`型传参都存储到数组`$post_data`中
- 如果文件存在进行包含。
- 综合入口分析的结果，我们可以使用绝对路径绕过`filter`对路径穿越的处理
- 由此我们可以读取所有的php文件
- 如果当PHP版本<5.3.4的时候，甚至可以使用`%00`截断，达到任意文件包含的效果

![image-20210812175315670](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812175315670.png)

- 打断点进行测试，发现传递的值是正常的

![image-20210812185212104](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812185212104.png)

- 提前写好 `phpinfo.php`文件，运行断点进行测试，成功显示phpinfo页面说明，分析正确

![image-20210812185329528](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812185329528.png)

#### normaliz.php变量覆盖

- 从26行开始看起，首先实例化一个对象
- 创建三个变量并赋值，调用`action`函数，将结果保存在`$data['res']`中
- 接着调用`echoContent`处理页面
- 跟踪一下`action`函数
  - `$post_data`哪来的，跟踪一下发现，是`action.php`里面的数组
  - 接下来就是对数组的键值对，转化为变量和值的形式
  - 如果我们在`$post_data`里面传递包含`$ip_replacement`和`$mail_replacement`的键值对，当数组转化为变量和值的形式，是不是就相当于对函数接收的形参，进行了重新赋值，达到了变量覆盖的效果
  - 这里应该如何利用呢，就需要一个小Tips：**`preg_replace`函数在`e`模式下，匹配成功会造成任意代码执行**（但是这个必须要在`PHP<7.0`，才可以）
  - 因此我们需要让`method`为`/xxx/e`开启`e`模式，`mail_replacement`函数为我们要执行的代码，`$source`为任意值即可

![image-20210812185649001](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812185649001.png)



- 因为要7.0 以下版本才可以，我的版本太高，因此这里只能演示动调结果：

![image-20210812192354431](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812192354431.png)

![image-20210812192947539](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812192947539.png)

![image-20210812192809294](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812192809294.png)

![image-20210812193152564](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812193152564.png)

![image-20210812193322131](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812193322131.png)

#### common.php任意文件写入

- 这里存在一个`write_log`函数，在入口分析时，知道他会把错误路径写入到`logs/logfile.php`

![image-20210812193945159](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812193945159.png)

- 全局搜索看在哪里调用了这个函数（PHPStrom 直接Ctrl + shift + F，我这里热键冲突了，使用seay进行查找）,发现有两处调用了它，其中一处可疑，进行跟踪。

![image-20210812194508977](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812194508977.png)

- 也就是我们入口分析时，已经提到过的`loadFile`
- 也就是说，只要我们传递错误的路径，这个错误的路径就会被写入到日志中，在通过`action.php`的文件包含就可以达到任意文件写入的目的

![image-20210812194645245](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812194645245.png)

- 全局搜索，看看哪里调用了`loadFile`函数
- 这里我们发现有三处调用了该函数，但是前两处参数都是不可控的。只有第三处可控，跟踪第三处

![image-20210812195318904](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812195318904.png)

- 这里任然是我们入口分析处，提到的函数，由之前分析可知，`$vId`是我们可控的，也就是`index.php`
- 整理漏洞里利用过程 `index.php`中利用`page`参数访问我们想要写入的文件信息，因为文件不存在会被写入日志。我这里没有禁止对`logfile.php`的访问，要是被禁止了，使用`action.php`进行文件包含即可。

![image-20210812195448956](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812195448956.png)

- 利用过程演示

![image-20210812200213513](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812200213513.png)

![image-20210812200320221](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812200320221.png)

#### view.php任意代码执行

- 剩下为验证的只有`view.php`的`eval`，对其进行跟踪

![image-20210812200435270](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812200435270.png)

- 我们发现所有的这些`eval` 全部是出现在`view.php`的`parseSubIf`方法中
- 而且我们发现，`eval`里面的可控参数`strIf`是和`$conntect`关联起来的，要想控制`$strIf`必须通过`$content`正则替代出来

![image-20210812201848536](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812201848536.png)

- 全局搜索，发现只有函数本身和`echoContent`进行了调用，跟踪`echoContent`

![image-20210812202240937](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812202240937.png)

- 这个函数在四个文件进行了调用，挨个文件进行分析

![image-20210812202613714](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812202613714.png)

- 发现只有`md5.php`文件中的`$data[page]`通过变量覆盖，可以控制

![image-20210812203104607](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812203104607.png)

- 利用`md5.php`文件构造payload进行动调

![image-20210812203445057](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812203445057.png)

![image-20210812203914346](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812203914346.png)

![image-20210812204131689](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812204131689.png)

![](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812204320360.png)

![image-20210812204805996](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812204805996.png)

![image-20210812204957450](https://husins.oss-cn-beijing.aliyuncs.com/image-20210812204957450.png)

## 总结

1. 第一次做代码审计，审计的cms也比较小，总体来说很累也很有成就感
2. 代码审计一定要对危险函数敏感，看到这个函数就知道要怎么利用，基础不牢地动山摇
3. 后续要完善Seay的正则，现在用着非常不流畅
4. 代码审计一定要有思路，跟踪好危险函数才行。

## 参考

https://rj45mp.github.io/php-chinaz

https://ca01h.top/code_audit/PHP/2.PHP%E4%BB%A3%E7%A0%81%E5%AE%A1%E8%AE%A1%E5%A4%8D%E7%8E%B0%E2%80%94%E2%80%94chinaz/