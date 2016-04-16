Date: 2016-04-16
Title: PHP的多线程Curl,rolling Curl实现与坑
Tags:  PHP curl mutil_curl 多线程
Toc:no
Status: public
Position: 1

  最近需要做群发推送,之前的推送只有单发的API,因为分IOS跟安卓平台,最简单的实现还是塞到队列去一个一个发,毕竟用户不多也就几万个.之前的请求队列没用多线程请求,每次一个.然而这个推送接口速度太挫,ios经常要2-3s才返回,几万个单线程请求得等死了.于是开始研究PHP的多线程Curl
  之前知道有mutil_curl可以实现,以为很简单.实际操作起来,坑还是相当多的.

  php手册参考：

>http://php.net/manual/zh/book.curl.php

  curl_multi_init()相关：

>http://php.net/manual/zh/function.curl-multi-init.php

手册说的很简单，并附了一个演示脚本，如下。

```
<?php
// 创建一对cURL资源
$ch1 = curl_init();
$ch2 = curl_init();
 
// 设置URL和相应的选项
curl_setopt($ch1, CURLOPT_URL, "http://www.example.com/");
curl_setopt($ch1, CURLOPT_HEADER, 0);
curl_setopt($ch2, CURLOPT_URL, "http://www.php.net/");
curl_setopt($ch2, CURLOPT_HEADER, 0);
 
// 创建批处理cURL句柄
$mh = curl_multi_init();
 
// 增加2个句柄
curl_multi_add_handle($mh,$ch1);
curl_multi_add_handle($mh,$ch2);
 
$running=null;
// 执行批处理句柄
do {
    usleep(10000);
    curl_multi_exec($mh,$running);
} while ($running > 0);
 
// 关闭全部句柄
curl_multi_remove_handle($mh, $ch1);
curl_multi_remove_handle($mh, $ch2);
curl_multi_close($mh);
```
演示脚本说明了工作流程。

1、curl_multi_init，初始化多线程curl批处理会话。

2、curl_multi_add_handle，将具体执行任务的curl添加到multi_curl批处理会话。

3、curl_multi_exec，真正开始执行curl任务。

4、curl_multi_getcontent，获取执行结果。

5、curl_multi_remove_handle，将完成了的curl移出批处理会话。

6、curl_multi_close，关闭curl批处理会话。


mutil_curl相对于传统curl,最重要的还是理解几个概念.mutil_curl相当于一个请求线程池,这里的每一个请求都是一个单独的curl对象,跟传统的curl一样.curl_multi_add_handle方法就是往线程池里添加新的curl对象,但是并不执行.curl_multi_exec则会从线程池中拿一个curl去请求,这个时候是异步非阻塞的,所以很多请求也可以一下发出去,效率高多了.请求有返回了咋办呢,官方例子写的不好,其实有更好的方法.通过curl_multi_select方法,系统会阻塞直到有请求返回.这不就是回调通知么.而且curl_multi_select支持超时,所以非常好用.代码写出来如下:

```
<?php
//bad example. 错误的例子
$start_time = microtime();
date_default_timezone_set('PRC');
 
$targets = array();//150个网址，自行添加目标
 
$total = count($targets);
echo "共有{$total}个目标:\n";
 
$mh = curl_multi_init();
 
$opt = array ();
$opt[CURLOPT_HEADER] = false;
$opt[CURLOPT_CONNECTTIMEOUT] = 15;
$opt[CURLOPT_TIMEOUT] = 30;
$opt[CURLOPT_AUTOREFERER] = true;
$opt[CURLOPT_RETURNTRANSFER] = true;
$opt[CURLOPT_FOLLOWLOCATION] = true;
$opt[CURLOPT_MAXREDIRS] = 10;
 
foreach($targets as $target){
    $ch = curl_init($target);
    curl_setopt_array($ch, $opt);
    curl_multi_add_handle($mh, $ch);//要设置每个curl实例的属性
    unset($ch);
}
 
$index = 1;
do{
    do{
        curl_multi_exec($mh, $running);
        curl_multi_select($mh, 1.0);
    }while($running);
 
    while($info = curl_multi_info_read($mh)){
            $ch = $info['handle'];
            $url = curl_getinfo($ch, CURLINFO_EFFECTIVE_URL);
     
    if($info['result'] == CURLE_OK){
                $content = curl_multi_getcontent($ch);
                $detail = getTitle($content);
            }else
                $detail =  'cURL Error(' . curl_error($ch) . ").";
     
    echo "[$index][", date('H:i:s'), "]{$url}:{$detail}\n";
            $index++;
    }
 
}while($running);
 
curl_multi_close($mh);
 
function getTitle($html){
    preg_match("/<title>(.*)<\/title>/isU", $html, $title);
    return empty($title[1]) ? '未能获取标题' : $title[1];
}
 
echo "抓取完成!\n";
$end_time = microtime();
$start_time = explode(" ", $start_time);
$end_time = explode(" ",$end_time);
$execute_time = round(($end_time[0] - $start_time[0] + $end_time[1] - $start_time[1]) * 1000) / 1000;
$execute_time = sprintf("%s", $execute_time);
echo "脚本运行时间：{$execute_time} 秒\n";
```

  curl_multi_select()这个方法的返回值,手册上是这样写的:

>成功时返回描述符集合中描述符的数量。失败时，select失败时返回-1，否则返回超时(从底层的select系统调用).

  但是其实curl_multi_select可能会一直返回-1,所以这时候并不是真失败了,建议写的是继续执行curl_multi_exec.所以会写在一个while循环里.注意这些问题除了受代码编写的影响，还受php和libcurl版本的影响，总而言之升级版本吧。
  至于像 CURLM_CALL_MULTI_PERFORM 之类的预定义常量，php方面并没有详细解释，多半靠看名字猜，呵呵。

  其实这些常量是由libcurl库定义的，参考地址：
  http://curl.haxx.se/libcurl/c/libcurl-errors.html
  CURLM_CALL_MULTI_PERFORM (-1)

  This is not really an error. It means you should call curl_multi_perform again without doing select() or similar in between. Before version 7.20.0 this could be returned by curl_multi_perform, but in later versions this return code is never used.
  当返回值为-1时，并不意味着这是一个错误，只是说明select时没有并没有完成excute，描述给的建议是不要执行select等阻塞操作，立即exec。
但是在libcurl的7.20版本之后，不再使用这个返回值了，原因是这个循环libcurl自己做了，就不再需要我们手动循环了。
同时注意curl_multi_select，其实还有第二个参数timeout，根据语焉不详的手册，这货应该是自带阻塞，所以就不再需要手动sleep了。
  
  整个脚本的执行时间就是30秒多一点，刚好是为每个curl设置的超时时间.想了一下,这样写是每次把所有请求都发出去,相当于有多少请求就开多少线程,那如果一次搞几万请求 系统还不挂了.而且其实php并不能很好的处理这150个线程，导致很多目标获取标题失败了。

  这就需要自己实现一个线程池，来掌控任务进度。
  思路就是用curl_multi_remove_handle一次添加n个url到multi_curl中，这个n就是线程数，这n个的组合队列就是线程池。

每执行完毕一个任务，就将对应的curl移除队列并销毁，同时加入新的目标，直至150个对象依次执行完毕。这样做的好处是，能保证线程池中始终有n个任务在进行，不必等这n个任务执行完毕后再执行下n个任务。
  思路有了，所以我们的代码看来是这样的：

```
<?php
$start_time = microtime();
date_default_timezone_set('PRC');
 
$targets = array();//150个网址，自行添加目标
 
$total = count($targets);
 
$mh_pool = array();
$threads = 10;
$total_time = 0;
echo "共有{$total}个目标:\n";
echo "线程数：{$threads}\n";
 
$mh = curl_multi_init();
 
$opt = array ();
$opt[CURLOPT_HEADER] = false;
$opt[CURLOPT_CONNECTTIMEOUT] = 15;
$opt[CURLOPT_TIMEOUT] = 30;
$opt[CURLOPT_AUTOREFERER] = true;
$opt[CURLOPT_RETURNTRANSFER] = true;
$opt[CURLOPT_FOLLOWLOCATION] = true;
$opt[CURLOPT_MAXREDIRS] = 10;
 
if($total < $threads)
    $threads = $total;
 
for($i=0;$i<$threads;$i++){
    $task = curl_init(array_pop($targets));
    curl_setopt_array($task, $opt);
    curl_multi_add_handle($mh, $task);//要设置每个curl实例的属性
    unset($task);
}
 
$index = 1;
 
do{
    do{
        curl_multi_exec($mh, $running);
        if(curl_multi_select($mh, 1.0) > 0)
            break;
    }while($running);
 
    while($info = curl_multi_info_read($mh)){
        $ch = $info['handle'];
        $url = curl_getinfo($ch, CURLINFO_EFFECTIVE_URL);
        $total_time += curl_getinfo($ch, CURLINFO_TOTAL_TIME);
         
        if($info['result'] == CURLE_OK){
            $content = curl_multi_getcontent($ch);
            $detail = getTitle($content);
        }else
            $detail =  'cURL Error(' . curl_error($ch) . ").";
         
        echo "[$index][", date('H:i:s'), "]{$url}:{$detail}\n";
        $index++;
         
        curl_multi_remove_handle($mh, $ch);
        curl_close($ch);
        unset($ch);
         
        if($targets){
            $new_task = curl_init(array_pop($targets));
            curl_setopt_array($new_task, $opt);
            curl_multi_add_handle($mh, $new_task);//要设置每个curl实例的属性
        }
    }
 
}while($running);
 
curl_multi_close($mh);
 
function getTitle($html){
    preg_match("/<title>(.*)<\/title>/isU", $html, $title);
    return empty($title[1]) ? '未能获取标题' : $title[1];
}
 
echo "抓取完成!\n";
$end_time = microtime();
$start_time = explode(" ", $start_time);
$end_time = explode(" ", $end_time);
$execute_time = round(($end_time[0] - $start_time[0] + $end_time[1] - $start_time[1]) * 1000) / 1000;
$execute_time = sprintf("%s", $execute_time);
echo "http请求时间：{$total_time} 秒\n";
echo "脚本运行时间：{$execute_time} 秒\n";
```

  为了细化处理多线程curl每个请求的执行结果，我在curl_multi_select的返回值大于0的时候也跳出了当前exec循环，并通过curl_multi_info_read来获取已经完成的任务信息。这里封装下就可以做个回调，精细化处理每个任务。

  另一个地方就是注意curl_multi_info_read需要多次调用。这个函数每次调用返回已经完成的任务信息，直至没有已完成的任务。问题4产生的原因就是因为我当时用了if没用while，这是一个小坑，但坑了我相当长的时间。当时非常无奈的解决方式是监控了整个执行过程，在所有任务完成后清点队列，把遗漏的再取出来。。。

  这样搞之后,发现几百个请求已经很ok了,公司网络不是很好,请求1000次百度大概要几十秒.而且同时开起的$threads进程数量也不是越多越好,根据网络和CPU核数自己测试估计能取得个平衡.把请求队列扩大,搞到5000次,发现问题来了.

  5000次请求,基本只有1500-1800次能成功,这肯定是有问题了.然而跟超时时间并没有关系.在每个环节打了log发现,在任务结束的时候$targets并没有全部pop出去.也就是说,剩下的请求根本就没发.这特么,是有错误还是停止条件的问题?

  通过log记录每个请求之后的$running值,也就是停止条件,发现$running确实是变成0了,所以会退出循环.这个$running是这样来的:
  >curl_multi_exec($mh, $running);

  手册写的是:

```
int curl_multi_exec ( resource $mh , int &$still_running )
处理在栈中的每一个句柄。无论该句柄需要读取或写入数据都可调用此方法。

参数
mh              由 curl_multi_init() 返回的 cURL 多个句柄。
still_running   一个用来判断操作是否仍在执行的标识的引用。
```

  这特么说了没说一样啊.谁知道是啥意思. 通过log不断的观察,发现这个值,表示的是 "正在进行中的线程数量",也就是当前有多少请求正在执行.然后就发现,这个值并不是每次都会减少1...有时候会有多个请求一起返回了,所以会减少多个.但是每次往池子里添加的新请求都是1,这样肯定越来越少了,最后$running就变成0了,但是请求任务还没有完...
  最终写了一个class,凑合撸吧.这样基本没问题了.

```
<?php 
/*
 * Multi curl in PHP 
 * @author  rainyluo 
 * @date    2016-04-15
 */
class MultiCurl {
    //urls needs to be fetched 
    public $targets = array();
    //parallel running curl threads 
    public $threads = 10;
    //curl options 
    public $curlOpt = array();
    //callback function 
    public $callback = null;
    //debug ,will show log using echo 
    public $debug = true;
    
    //multi curl handler
    private $mh = null;
    //curl running signal 
    private $runningSig = null;


    public function __construct() {
        $this->mh = curl_multi_init();
        $this->curlOpt             = array(
            CURLOPT_HEADER         => false,
            CURLOPT_CONNECTTIMEOUT => 10,
            CURLOPT_TIMEOUT        => 10,
            CURLOPT_AUTOREFERER    => true,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_FOLLOWLOCATION => true,
            CURLOPT_MAXREDIRS      => 5,
        );
        $this->callback = function($html) {
            echo md5($html);
            echo "fetched"; 
            echo "\r\n";
        };
    }

    public function setTargets($urls) {
        $this->targets = $urls;
        return $this;
    }
    public function setThreads($threads) {
        $this->threads = intval($threads);
        return $this;
    }
    public function setCallback($func) {
        $this->callback = $func;
        return $this;
    }
    /* 
     * start running 
     */
    public function run() {
        $this->initPool();
        $this->runCurl(); 
    }

    /*
     * run multi curl 
     */
    private function runCurl() {
        do{
            //start request thread and wait for return,if there's no return in 1s,continue add request thread 
            do{
                curl_multi_exec($this->mh, $this->runningSig);
                $this->log("exec results...running sig is".$this->runningSig);
                $return = curl_multi_select($this->mh, 1.0);
                if($return > 0){
                    $this->log("there is a return...$return");
                    break;
                }
                unset($return);
            } while ($this->runningSig>0);

            //if there is return,read it 
            while($returnInfo = curl_multi_info_read($this->mh)) {
                $handler = $returnInfo["handle"];
                if($returnInfo["result"] == CURLE_OK) {
                    $url = curl_getinfo($handler, CURLINFO_EFFECTIVE_URL);
                    $this->log($url."returns data");
                    $callback = $this->callback;
                    $callback(curl_multi_getcontent($handler));
                } else {
                    $url = curl_getinfo($handler, CURLINFO_EFFECTIVE_URL);
                    $this->log("$url fetch error.".curl_error($handler));
                }
                curl_multi_remove_handle($this->mh, $handler);
                curl_close($handler);
                unset($handler);
                //add new targets into curl thread 
                if($this->targets) {
                    $threadsIdel = $this->threads - $this->runningSig;
                    $this->log("idel threads:".$threadsIdel);
                    if($threadsIdel < 0) continue;
                    for($i=0;$i<$threadsIdel;$i++) {
                        $t = array_pop($this->targets);
                        if(!$t) continue;
                        $task = curl_init($t);
                        curl_setopt_array($task, $this->curlOpt);
                        curl_multi_add_handle($this->mh, $task);
                        $this->log("new task adds!".$task);
                        $this->runningSig += 1;
                        unset($task);
                    }

                } else {
                    $this->log("targets all finished");
                }
            }
        }while($this->runningSig);
    }

    /*
     * init multi curl pool
     */
    private function initPool() {
        if(count($this->targets) < $this->threads) $this->threads = count($this->targets);
        //init curl handler pool ...
        for($i=1;$i<=$this->threads;$i++) {
            $task = curl_init(array_pop($this->targets));
            curl_setopt_array($task, $this->curlOpt);
            curl_multi_add_handle($this->mh, $task);
            $this->log("init pool thread one");
            unset($task);
        }
        $this->log("init pool done");
    }

    private function log($log) {
        if(!$this->debug) return false;
        ob_start();
        echo "---------- ".date("Y-m-d H:i",time())."-------------";
        if(is_array($log)) {
            echo json_encode($log);
        } else {
            echo $log;
        }
        $m = memory_get_usage();
        echo "memory:".intval($m/1024)."kb\r\n";
        echo "\r\n";
        flush();
        ob_end_flush();
        unset($log);
    }
    public function __destruct(){
        $this->log("curl ends.");
        curl_multi_close($this->mh);
    }
    

}
```

  用法:

```
$mu = new MultiCurl();
$callback = function($html) {
    var_dump($html);
};
$mu->setTargets($urls)->setCallback($callback)->setThreads(5)->run();
```

  PS:一个插曲
  代码写的差不多的时候发现,每次请求快到2k的时候就崩溃了,报segmentation fault.这不坑爹呢么,上GDB看Coredump,捣鼓半天发现应该是跟内存回收有关系,找了半天也没有相关的解决方案,迷迷糊糊的.想想还是老路子,打log吧.每一步的内存都打出来,发现内存占用越来越多啊,每次请求要十几k的往上涨,最后一百多M了就挂了.这有问题啊,肯定有啥没释放的地方.把不用的变量都unset掉,猛然发现少了curl_multi_remove_handle($this->mh, $handler);跟curl_close($handler) 这不坑爹呢么...链接忘了close,肯定不是释放了啊.终归还是对curl_mutil的内部不是很熟悉.估计里面的每个curl对象都是独立的,只有close之后才会被回收释放吧.改好之后,10个threads一起跑,基本上内存占用十几m不到,5k的请求也一会搞定了.妈蛋,充实的一天 自己给自己挖坑啊.
  
