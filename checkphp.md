Date: 2016-11-17
Title: git commit后自动检查php文件语法
Tags:  php 自动检查 git commit 
Toc:no
Status: public
Position: 1

最近,同事有出现一些低级错误的情况,比如提交的代码有语法错误啊,var_dump没有去掉啊什么的.为了防止自己也这样(文件一多就容易乱啊...),写了个小脚本,git commit后自动检查语法和文件内容,如果语法有错误或者有var_dump/print_r就会显示出来,防止自己被耻笑...

PHP代码如下:

```
<?php
//git有变动的文件列表
$file_list = shell_exec("git diff-index --name-only --diff-filter=ACMR HEAD");
if(empty($file_list) || empty($_SERVER["argv"][1])) {
    cecho ("没有文件改动或文件路径错误", "RED");
    die();
}

$check_dir = $_SERVER["argv"][1];//要检查的目录路径

$file_list = explode("\n", $file_list);
if(empty($file_list[0])) {
    cecho ("没有更新文件 :)");
    die();
}

cecho("更新文件检查如下", "GREEN");
foreach($file_list as $file) {
    $file_full_path = $check_dir. "/". $file;
    if(substr($file, -4) != ".php") continue;//只检查php文件
    cecho( "检查.....", "GREEN");
    cecho( $file_full_path, "YELLOW");
    /* 语法检查 */
    $php_syntax = shell_exec("php -l ".$file_full_path);
    if(substr($php_syntax, 0, 2) !== "No" ) {
        cecho("有语法错误!!!!!!!\r\n", "RED");
        cecho($php_syntax);
    } else {
        cecho("没啥问题.........\r\n", "GREEN");
    }
    /*关键字检查 */
    $content = file_get_contents($file_full_path);
    if(empty($content)) cecho ("文件内容为空!!!!!", "RED");
    if(strpos($content, "var_dump")) cecho ("发现关键词:var_dump!!!!!", "RED");
    if(strpos($content, "print_r")) cecho ("发现关键词:print_r!!!!!", "RED");
}

function cecho($text, $color="LIGHT_RED"){
    $_colors = array(
        'LIGHT_RED'      => "[1;31m",
        'LIGHT_GREEN'     => "[1;32m",
        'YELLOW'     => "[1;33m",
        'LIGHT_BLUE'     => "[1;34m",
        'MAGENTA'     => "[1;35m",
        'LIGHT_CYAN'     => "[1;36m",
        'WHITE'     => "[1;37m",
        'NORMAL'     => "[0m",
        'BLACK'     => "[0;30m",
        'RED'         => "[0;31m",
        'GREEN'     => "[0;32m",
        'BROWN'     => "[0;33m",
        'BLUE'         => "[0;34m",
        'CYAN'         => "[0;36m",
        'BOLD'         => "[1m",
        'UNDERSCORE'     => "[4m",
        'REVERSE'     => "[7m",
    );
    $out = $_colors["$color"];
    if($out == ""){ $out = "[0m"; }
    echo chr(27)."$out$text".chr(27).chr(27)."[0m".chr(27);
    echo "\r\n";
}
?>
```


使用方法: 
在~/.bashrc 或者 .zshrc里添加个alias  
> alias 'gcm'='commit() {dir=$(pwd);php -c /home/luo/php.ini /home/luo/checkphp.php "$dir";git commit -a -m "$1"}; commit'  

指定了php.ini是为了用自己的配置来允许shell_exec函数执行.  
然后在git目录,需要提交的时候输入 gcm "提交注释" 就可以了.  


