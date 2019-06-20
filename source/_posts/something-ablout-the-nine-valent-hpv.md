---
title: 两次秒杀九价疫苗
date: 2019-05-24 20:21:22
---
最近抢了两次九价疫苗，感觉每个小仙女的朋友/自己都是程序员啊，各种0秒以内的秒杀。。。
<!--more-->

## 第一次
### 居然是个问卷星
得到的预约链接是个问卷星，就自己注册了个账号，开了个问卷，试着构造请求。请求构造是不难，难的是怎么能知道各个字段的name呢？
鬼使神差，在问卷星上，看到了个`复制他人问卷`。尝试着输入真实问卷的Id，竟然真的复制成功了！
接着按照真实表单设置好了参数，写了脚本，还练习了利用手机输入法的快捷输入功能，凭借手速抢HPV。
心里美滋滋，两手准备，等着手到擒来了。
``` bash
# macOs
lastTime=$(date -v-16S "+%Y/%m/%d %H:%M:%S") 
last1Second=$(date "+%s")
echo ${lastTime}
echo ${last1Second}
urlencode() {
    # urlencode <string>
    local length="${#1}"
    for (( i = 0; i < length; i++ )); do
        local c="${1:i:1}"
        case $c in
            [a-zA-Z0-9.~_-]) printf "$c" ;;
            *) printf '%%%02X' "'$c"
        esac
    done
}
curlCMD=`curl "https://www.wjx.top/joinnew/processjq.ashx?curid=37747717&starttime=$(urlencode "${lastTime}")&source=directphone&submittype=1&ktimes=77&hlv=1&rn=3082155441.38864532&iwx=1&t=${last1Second}&jqnonce=e04253b0-8873-4ffe-8bd9-69a851149746&jqsign=b73524e7*%3F%3F04*3aab*%3Fec%3E*1%3Ef%3F2663%3E031" -H 'Pragma: no-cache' -H 'Origin: https://www.wjx.top' -H 'Accept-Encoding: gzip, deflate, br' -H 'Accept-Language: zh-CN,zh;q=0.9' -H 'User-Agent: User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 12_1_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/16C101 MicroMessenger/6.7.4(0x1607042c) NetType/WIFI Language/zh_CN' -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' -H 'Accept: text/plain, */*; q=0.01' -H 'Cache-Control: no-cache' -H 'X-Requested-With: XMLHttpRequest' -H 'Cookie: ******' -H 'Connection: keep-alive' -H 'Referer: https://www.wjx.top/m/37747717.aspx' --data 'submitdata=******' --compressed`
echo ${curlCMD}
```
### 惨痛教训
一早写好了脚本，想着伪造的真实一点，预约开始后15秒再提交吧～
让我没想到的是，这场`秒杀`8秒钟就结束了，事后看公示的预约时间，大半的`0秒`。。。真实个P啊，回去要被骂了- -。


## 第二次

过了几天，又有卫生所放出疫苗预约了。这次也事先放出了预约地址，利用支付宝公众号来进行预约。有了前车之耻，决定抓出请求来提前**循环发送**。

### 支付宝上的预约网站
在支付宝上打开网站，看了看感觉它是一个H5。作者很心机，在页面Mount的时候，还隐藏了分享按钮，不想让人知道链接地址。
支付宝容器是只认自己信任的证书的，抱着绝望的心态，还是开启了代理进行抓包。没想到啊，居然没有使用SSL加密，很顺利的拿到了H5的地址。
`http://***.**.com/jkzweb/#/healthCard/choiceServiceCenter?page=2&app_id=123456789&source=alipay_wallet&scope=auth_user&auth_code=39ff37d685ee44a3b0b78786a05bXX55`
美滋滋的在Chrome中打开链接，却被无情地嘲弄了。

![](/post-images/hpv-1.png)

这是一个依赖支付宝鉴权的H5，只有通过了H5鉴权才能获得它自己的session。

翻阅了[蚂蚁的认证授权文档](https://docs.open.alipay.com/289/105656)，尝试拼接认证链接：
`https://openauth.alipay.com/oauth2/publicAppAuthorize.htm?app_id=123456789&redirect_uri=http%3a%2f%2f**.*.com%2fjkzweb%2fpalmDoctorIndex.html&scope=auth_user`
再次尝试，等链接跳到支付宝认证页面后，怀着忐忑的心情等着转跳到H5。然而依旧开心的太早了啊。

![](/post-images/hpv-2.png)

没办法，硬着头皮再试着自定义user-agent。
```
Mozilla/5.0 (iPhone; CPU iPhone OS 12_1_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/16C101 NebulaSDK/1.8.100112 Nebula WK PSDType(1) AlipayDefined(nt:4G,ws:375|603|2.0) AliApp(AP/10.1.65.6000) AlipayClient/10.1.65.6000 Alipay Language/zh-Hans
```
终于，跳到了H5应用里了！
### 分析H5
看了看是个Vue的应用，跟了几遍代码，很自然的找出了Store、Routes和各种Actions。全程模拟着正常的操作，最后抓出了提交链接。
并且最后的提交，是不验证session，只认用户支付宝用户id的。因此，直接copy出了一条curl，到点循环发送了。
``` bash
curl 'http://***.*.com/api/appointment/appointment' -H 'Accept: application/json, text/plain, */*' -H 'Referer: http://***.*.com/jkzweb/' -H 'Origin: http://***.*.com' -H 'User-Agent: User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 12_1_2 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/16C101 NebulaSDK/1.8.100112 Nebula PSDType(1) AlipayDefined(nt:WIFI,ws:375|603|2.0) AliApp(AP/10.1.58.6100) AlipayClient/10.1.58.6100 Language/zh-Hans' -H 'Content-Type: application/x-www-form-urlencoded;charset=UTF-8' --data 'hospId=33010460&openId=1111111&patientName=%E8&patientCardID=3*******8&patientPhone=123456789' --compressed
```
到了秒杀的时候，定时器疯狂地发送脚本，终于抢到了。
活下来了啊～

## 蕴含的知识

### encodeURIComponent的作用
本质是把除了ASCII字母、数字、~!*()’的字符转义，转为16进制。[参考](https://www.cnblogs.com/season-huang/p/3439277.html)


### JS中如何把字符处理成16进制
``` js
str.charCodeAt(i).toString(16);

function stringToHex(str){
  var val = '';
  for(var i = 0; i < str.length; i++){
    if (val == '') {
      val = str.charCodeAt(i).toString(16);
    } else {
      val += ',' + str.charCodeAt(i).toString(16);      
    }
　}
　return val;
}

arr[i].fromCharCode(i)
function hexToString(str){
  var val = '';
  var arr = str.split(',');
  for(arr i = 0; i < arr.length; i++){
    val += arr[i].fromCharCode(i);
  }
  return val;
}
```
[参考](https://www.cnblogs.com/duhuo/p/5608246.html)

### 写sh脚本时
* $() 执行括号里面的内容 结果赋值
* ${} 变量
* "" 里面可以有$变量
* '' 里面的值是纯文本
* [sh里如何encode url](https://stackoverflow.com/questions/296536/how-to-urlencode-data-for-curl-command)
* [mac date命令详解](https://www.cnblogs.com/amyzhu/p/10177086.html)
* 其中%%%02x 可以分开为两部分"%%"和"%02x" 两个%%是输出一个'%'，这里第一个%是转义符 %02x中的%x是把数字输出为16进制的格式，%02x是保证输出至少占两个字符的位置,如果不够两位的话前面补0.
