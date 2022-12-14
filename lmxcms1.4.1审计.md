# 环境搭建

- [下载地址](http://www.lmxcms.com/down/xitong/20210530/14.html)

- 导入phpstudy的www目录，访问环境的install目录即可

# 环境介绍

梦想CMS，标准的MVC结构的小众CMS，观察一下目录结构

![image-20220923114326627](img/image-20220923114326627.png)

观察一下路由：

![image-20220923114740392](img/image-20220923114740392-16639209414831.png)

跟进路由配置文件

![image-20220923115042866](img/image-20220923115042866.png)

![image-20220923115455073](img/image-20220923115455073.png)

![image-20220923115655201](img/image-20220923115655201.png)

# 漏洞挖掘

## 0x01 留言板SQL注入

浏览网站前台，发现前台存在留言板功能，根据路由找到`c/index/BookAction.php`文件

![image-20220923121508271](img/image-20220923121508271.png)

![image-20220923122744568](img/image-20220923122744568.png)

跟进`filter_str`函数

![image-20220923123237837](img/image-20220923123237837.png)

![image-20220923123522321](img/image-20220923123522321.png)

分别过滤了一部分关键字和保留字，小case，我的评价是随便绕

![image-20220923123623225](img/image-20220923123623225.png)

![image-20220923123720425](img/image-20220923123720425.png)

逐层追踪最终找到SQL语句，写了一堆自定义变量糅杂在一起的SQL语句，我哪能看的懂嘛！！

tips：因为是白盒审计我们可以加一丢丢代码，比如输出一下拼接好的SQL语句，因此我在152加上了`echo $sql;`

![image-20220923123930539](img/image-20220923123930539.png)

抓包能找到sql语句的完整写法

![image-20220923124844258](img/image-20220923124844258.png)

准备要开始注入了，突然发现一个问题，忽略了P函数的后半部分

![image-20220923125418852](img/image-20220923125418852.png)

![image-20220923125449928](img/image-20220923125449928.png)

直接G，绕不过去了，但是好像只过滤了值，没有过滤键，，因此键还是可以进行注入的，回到SQL语句处

![image-20220923125854394](img/image-20220923125854394.png)

接下来就可以开始思考如何开始注入了，一开始黑盒测试能够得到一个结论，我们提交的留言并不会展示在前台，一般来说是否展示会在数据库里面有字段来进行保存，链接数据库验证猜想，确实是被ischeck确定的

![image-20220923130830410](img/image-20220923130830410.png)

开始构造语句

```sql
已知：
INSERT INTO lmx_book(name,content,mail,tel,ip,time) VALUES('1','4','2','3','127.0.0.1','1663898632')
name 和 mail 可控

可以在mail处进行注入
mail,tel,ip,time,ischeck) VALUES('1','4','2','3','127.0.0.1','1663898632','1')#

拼接之后的SQL语句变成：
INSERT INTO lmx_book(name,content,mail,tel,ip,time,ischeck) VALUES('1','4','2','3','127.0.0.1','1663898632','1')#,tel,ip,time) VALUES('1','4','2','3','127.0.0.1','1663898632')

运行之后得到如下回显：
INSERT INTO lmx_book(name,content,mail,tel,ip,mail,tel,ip,time,ischeck)VALUES('1','4','2','3','127_0_0_1','1663898632','1')#,time) VALUES('1','4','','3','127.0.0.1','2','1663899230')
不难发现我们输入的数据被拼接到了末尾，因此删掉重复数据即可
time,ischeck) VALUES('1','4','2','3','127.0.0.1','1663898632','1')#
```

发现回显点是1,4两个位置

![image-20220923131849566](img/image-20220923131849566.png)

直接开注

![image-20220923132022793](img/image-20220923132022793.png)

## 0x02 搜索框SQL注入

搜索栏随便输入点什么，根据传参结合路由找到对应的实现类，`c/index/SearchAction.class.php`

![image-20220923171130505](img/image-20220923171130505.png)

![image-20220923171448233](img/image-20220923171448233.png)

![image-20220923171737265](img/image-20220923171737265.png)

接着阅读，`searchCounth`统计数量，明显是进行了SQL查询，跟进

![image-20220923172342631](img/image-20220923172342631.png)

![image-20220923172531527](img/image-20220923172531527.png)

继续跟进`countModel`,直到跟进到SQL代码处

老规矩，SQL代码过于复杂，直接输出SQL语句看完整语句的样子进行分析

![image-20220923172724638](img/image-20220923172724638.png)

![image-20220923173014523](img/image-20220923173014523.png)



```sql
根据上述操作我们能够获得完整语句为:
SELECT count(1) FROM lmx_product_data  WHERE time > 1632378529 AND classid in(11,12,13,14,5)  AND (title like '%root%')  ORDER BY id desc
可控参数为keyword值，因为被%%包裹并不好利用
```

回到上述，不要忘记还有几个参数并没有被用到，尝试传入其他参数

![image-20220923173647676](img/image-20220923173647676.png)

这样就成功制造出了一个可控参数

通过测试`tuijian=1%20or%200--`和`tuijian=1%20or%201--`数据包大小有很大差异，可以进行布尔盲注

## 0x03 后台任意文件删除

后台存在文件管理，打开之后发现可以删除图片抓包分析

![image-20220923181853053](img/image-20220923181853053.png)

对上面的值进行URL解码

![image-20220923181915314](img/image-20220923181915314.png)

找到对应的源码进行分析，使用`unLink`函数进行文件删除，以#####作为分割

![image-20220923182043608](img/image-20220923182043608.png)

没啥过滤，直接进行任意文件删除，在install下创建`test.txt`尝试删除它

![image-20220923182706611](img/image-20220923182706611.png)

文件成功的被删除

## 0x04 后台任意文件写入

能够修改模板一般是文件写入的高发点，编辑模板发现全都是html文件，这不直接裂开

![image-20220923185312101](img/image-20220923185312101.png)

尝试抓包进行分析，发现这里路径写的不是很死呀，看看这块的代码吧

![image-20220923185437748](img/image-20220923185437748.png)

从代码中可以看出，对路径并没有什么过滤，因此尝试路径穿越

![image-20220923185619426](img/image-20220923185619426.png)

直接访问到了index.php的代码，尝试对文件进行修改

![image-20220923190032386](img/image-20220923190032386.png)

提交如下数据包

![image-20220923190345823](img/image-20220923190345823.png)

发现确实写入成功了

![image-20220923190412925](img/image-20220923190412925.png)

回到修改文件的源代码，跟进修改文件的PUT方法

![image-20220923190815883](img/image-20220923190815883.png)

`file_put_contents`有个特性，当文件不存在时，直接创建新的文件

因此可以利用这个特性写shell

![image-20220923191014237](img/image-20220923191014237.png)

![image-20220923191031299](img/image-20220923191031299.png)

# 参考链接

https://xz.aliyun.com/t/11224

https://blog.csdn.net/weixin_42508548/article/details/121516587