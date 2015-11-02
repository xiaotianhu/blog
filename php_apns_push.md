Date: 2015-11-02
Title: PHP 通过Apns给Ios设备推送Notification
Tags:  PHP Apns 推送
Toc:no
Status: public
Position: 1

前几天折腾Gopush,是没有通过Apns推送IOS实现的,找了几个Golang的实现都有问题,转而寻找PHP的实现方法,发现还是蛮简单的
Apns推送的原理懒得看,基本就是通过Http请求Apns然后苹果去给IOS设备发推送,所以只要请求苹果的服务器就好了.先弄证书

开发给我的都是p12结尾的证书,openssl搞一下
```
openssl pkcs12 -clcerts -nokeys -out cer.pem -in cer.pkcs12
openssl pkcs12 -nocerts -out key.pem -inkey.p12 
```
然后把key转换为不带密码的pem证书
```
openssl rsa -in key.pem -out key.unencrypted.pem
```
最后合并一下两个证书
```
cat cer.pem key.unencrypted.pem > ck.pem
```

PHP代码如下:

```
<?php
// 这里是我们上面得到的deviceToken，直接复制过来（记得去掉空格）
$deviceToken = 'device token';
// Put your alert message here:
$message = 'My first push test!.';
$ctx = stream_context_create();
stream_context_set_option($ctx, 'ssl', 'local_cert', 'ck.pem');
//stream_context_set_option($ctx, 'ssl', 'passphrase', $passphrase);
// Open a connection to the APNS server
//这个为正是的发布地址
 //$fp = stream_socket_client("ssl://gateway.push.apple.com:2195", $err, $errstr, 60, STREAM_CLIENT_CONNECT, $ctx);
//这个是沙盒测试地址，发布到appstore后记得修改哦
$fp = stream_socket_client('ssl://gateway.sandbox.push.apple.com:2195', $err,$errstr, 60, STREAM_CLIENT_CONNECT|STREAM_CLIENT_PERSISTENT, $ctx);

if (!$fp)
exit("Failed to connect: $err $errstr" . PHP_EOL);
echo 'Connected to APNS' . PHP_EOL;

// Create the payload body
$body['aps'] = array(
'alert' => $message,
'sound' => 'default'
);

// Encode the payload as JSON
$payload = json_encode($body);

// Build the binary notification
$msg = chr(0) . pack('n', 32) . pack('H*', $deviceToken) . pack('n', strlen($payload)) . $payload;
// Send it to the server
$result = fwrite($fp, $msg, strlen($msg));

if (!$result)
echo 'Message not delivered' . PHP_EOL;
else
echo 'Message successfully delivered' . PHP_EOL;
var_dump($result);
// Close the connection to the server
fclose($fp);
?>
```

运行一下 ,然后就收到啦~

注意:
记得判断好,app是sendbox的还是正式的,对不上也收不到推送

