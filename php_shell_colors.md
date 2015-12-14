Date: 2015-12-14
Title: 用PHP-CLI在SHELL中输出颜色 
Tags:  PHP CLI 
Toc:no
Status: public
Position: 1

刚才找PHP的手册,发现一个评论挺有意思.在PHP的CLI模式,也就是SHELL中,输出颜色.
写简单的脚本,Log什么的时候,有个颜色 ,看起来舒服多了.一直黑白的看的脑袋晕乎乎的.

```
<? 
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

function termcolored($text, $color="NORMAL", $back=1){ 
    global $_colors; 
    $out = $_colors["$color"]; 
    if($out == ""){ $out = "[0m"; } 
    if($back){ 
        return chr(27)."$out$text".chr(27).chr(27)."[0m".chr(27); 
    }else{ 
        echo chr(27)."$out$text".chr(27).chr(27)."[0m".chr(27); 
    }//fi 
}// end function 

foreach ($_colors as $key=>$v) {
    echo termcolored("colors", $key);
    echo "\r\n";
}
?>
```
