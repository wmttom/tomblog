[原创]Python2.x编码问题
======================

4字节UTF8问题
-----------------

| python的使用的utf8分为utf8-UCS2和UCS4
| 在UCS4模式下可以支持4字节utf8，如emoji表情
| 默认为UCS2模式，在处理4字节内容时会报错

| 检验方法：
| >>> import sys
| >>> print sys.maxunicode
| 65535 is UCS-2
| 1114111 is UCS-4

| 需要UCS-4模式时，需要在编译python时加入--enable-unicode=ucs4参数
| 注意：在UCS-4模式下，有些C/C++绑定的python库无法使用，使用时需要测试。MySQL使用时，
| MySQLdb版本需大于等于1.2.4，系统mysql-client版本需大于等于5.5.3，MySQL表字段字符集为utf8mb4时才可完美支持。

UTF8和unicode问题
------------------------

| Python2.x版本中，字符串编码是区分utf8和unicode，比如
| >>> name_utf8 = '编码' #为utf8
| >>> name_unicode = u'编码' #为unicode

| 可以如此相互转换
| # utf8 >> unicode
| >>> unicode(name_utf8, 'utf8') # 写为utf8/utf-8皆可
| # unicode >> utf8
| >>> name_unicode.encode('utf8')

在普通使用中感觉不到明显区别，但是在某些时候就会产生问题：

1. 格式化字符串时

    | 格式化字符串时，如果参数中包含unicode对象，那么格式化的结果也会为unicode对象。python会把整个字符串转为unicode编码，这个过程无法指定字符集，并且会使用ASCII，如果其中包含中文自然会报错。

    | >>> "{0}".format(u"编码")  # 报错
    | >>> "{0}".format(u"编码".encode("utf8"))  # 未报错，结果为utf8
    | >>> "%s编码" %(u"test")   # 报错
    | >>> "%s" %(u"编码")  # 未报错，结果为unicode

    | 在日常使用中，个人的建议是全部使用unicode，在最终需要格式化输出时，再根据情况转为utf8

2. urlencode参数时

   | 在web开发中，越来越多的使用中文的url参数，如http://www.foo.com/search?keyword=编码
   | 很多时候需要对url的参数进行urlencode，在python中常用的方法是urllib中的quote和urlencode

   | >>> from urllib import quote, urlencode
   | >>> quote(u"编码")  # 报错
   | >>> quote(u"编码".encode("utf8"))  # 未报错
   | >>> quote("编码")  # 未报错
   | >>> urlencode({"keyword": u"编码"})  # 报错
   | >>> urlencode({"keyword": "编码"})  # 未报错
   | >>> urlencode({"keyword": u"编码".encode("utf8")})  # 未报错

   | 在日常使用中，个人的建议是全部使用unicode，需要urlencode参数时，一律encode为utf8。
   | ASCII范围内（包括空字符）的utf8字符串进行encode("utf8")时不会报错，可进行多次。
