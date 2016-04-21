Date: 2016-03-23
Title: QueryPath中文乱码问题,UTF8乱码问题解决
Tags:  QueryPath 中文 UTF8
Toc:no
Status: public
Position: 1

最近研究Spider,找到了PHP的QueryPath库.之前用Python的BeautifulSoup也挺方便,这个跟Python的那个库还是蛮像的.

但是用起来的时候就蛋疼了,发现UTF8的中文总是乱码.研究了好久,尝试了mb_convert_encoding各种姿势,还是不能解锁

最终在github的一个issue里发现了,这个issue有人提出来了.按着lz的参数配置,传给qp()方法:

``` 
$qp_options = array(
        'convert_from_encoding' => 'UTF-8',
        'convert_to_encoding' => 'UTF-8',
        'strip_low_ascii' => FALSE,
        );

$qp = htmlqp($html, null, $qp_options)->find(".text")->find("p")->text();
```

并且记得,如果是HTML的片段,没有完整的HTML,还是要自己给补全一下HTML才能正确识别.

```
$html = '<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">';
$html .= '<html xmlns="http://www.w3.org/1999/xhtml"><head><meta http-equiv="Content-Type" content="text/html; charset=utf-8" /></head><body>';
$html .= $htmlString;
$html .= '</body></html>';
$qp = htmlqp($html, null, $qp_options)->find(".text")->find("p")->text();
```

中文问题就解决了.看样子是在新版中修复了这个bug,并不需要去搞mb_convert_encoding了.吐槽一下querypath的手册,写的还是有点shit.

ps: 如果是完整的HTML代码/网页,GBK编码也会有问题.这个是跟<meta http-equiv="Content-Type" content="text/html; charset=gbk" /> 有巨大关系的.需要把HTML的代码用mb_convert_encoding转换成UTF8,并把html中meta标签那个charset=gbk replace成charset=utf8,如此才能正确显示.编码问题真是坑爹啊.

github的issue地址:https://github.com/technosophos/querypath/issues/94




