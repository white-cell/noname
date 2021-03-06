# 记一次mssql注入到全局waf绕过
* 因为涉及客户信息和某知名厂商waf所以做了打码处理，不过不影响技术部分。

首先通过测试发现一处数字型盲注，正常请求如下图所示<br />
![正常请求](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/正常请求.jpg)<br />
因为之前测试就发现存在某厂商waf，奇数个单引号、and、or、xor等都被过滤了，可以通过| & 等运算符做注入测试，通过减运算可以发现成功执行了<br />
![减运算请求](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/减运算请求.jpg)<br />
* 首先此厂商waf存在很多waf的通病，就是对multipart/form-data 混杂模式支持不是很好，所在此模式下有一些原本拦截的就放行了，如version()、select 1，之前也有遇到过可以构造畸形包全局绕过的但是这里试了没成功，所以后面的绕过在混杂模式下进行<br />
![version拦截](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/version拦截.jpg)<br />
![version不拦截](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/version不拦截.jpg)<br />

想通过注释符判断下数据库类型，但是不管是# --都返回出错页面,substring函数和@@version过滤了，于是先通过2017-len(1)成功返回2016页面排除oracle，然后通过2017-left(version(),1)出错说明没有version()、尝试substr()也出错证明了不是mysql，通过2017-len(db_name())返回2013页面判断数据库为mssql且当前库名为四位。<br />
![拦截请求](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/拦截请求.jpg)<br />
![出错请求](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/出错请求.jpg)<br />
![判断mssql请求](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/判断mssql请求.jpg)<br />
接下来想获取当前库名，通过测试发现还过滤了substring()，charindex(),ascii(),我使用left()和right()组合使用来代替substring和charindex的缺失，通过测试发现waf会放行ascii('，当ascii(后面紧跟单引号的情况不会拦截，于是构造2017-ascii(''+left(right(db_name(),2),1))来一位位的判断库名的ascii码。<br />
![ascii()过滤](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/出错请求.jpg)<br />
![ascii()绕过](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/ascii绕过.jpg)<br />
如图判断出数据库的倒数第二位ascii码为99，是字母c。至此可以用已有的获取当前数据库，当前用户等信息，但是这还达不到拖取数据的阶段。
# 接下来试图绕过select from的过滤
![ascii出数据](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/ascii判断.jpg)<br />

* 通过fuzz了% /\*\*/等绕过正则的方法都没法绕过waf，于是还是把注意力放在了构造畸形包的思路上，于是收获了一个惊喜。

首先我们的select from是被拦截的<br />
![from拦截](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/from拦截.jpg)<br />
下面是绕过的请求
![全局绕过](https://github.com/white-cell/blog/raw/master/20180417-记一次mssql注入到全局waf绕过/全局绕过.jpg)<br />
绕过的几个构造点
==
* POST改成PO(其实是可以任意)
* https要换成http，当把POST换成GET时没有此限制
* 要用混杂模式不然服务器也会获取不到参数
* 然后就可以无视waf为所欲为了

思考
==
* waf在匹配到请求头POST不完整时，就放行了请求包。
* waf在匹配到请求头为GET时，就不检测请求体里的参数了。
* 服务器在匹配到请求头POST不完整或为GET时，虽然也获取不到普通请求体里的参数，但却可以获取到混杂模式的。因为是黑盒测试所以不清楚这里面的逻辑。
* 所以还是因为waf和服务器对畸形包的处理不同导致的全局绕过。
