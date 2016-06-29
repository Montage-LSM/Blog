---
title: 基于think3.2,webuploader的七牛直接多图上传的思路。
date: 2016-06-29
tags: [Qiniu,ThinkPHP,WebUpload]
other: true
---

分享一个基于webuploader的七牛多图上传的JS。

七牛要求的token与key我最后写。（PHP的，其他语言的自行翻文档）

<!--more-->
话不多说，直接贴代码：articel.js

```
/**
 * Created by Administrator on 2016/3/7.
 */
$(function () {
    // 初始化Web Uploader
    var uploader = WebUploader.create({

        // 选完文件后，是否自动上传。
        auto: true,

        // swf文件路径
        swf: "/Public/Extend/webuploader/Uploader.swf",

        // 文件接收服务端。(直接使用七牛)
        server: "http://up.qiniu.com",//$("#hidUrl").val(),

        // 选择文件的按钮。可选。
        // 内部根据当前运行是创建，可能是input元素，也可能是flash.
        pick: {
            id:"#content_img",
            lable:"<input type='button' class='btn' />",
            multiple:true
        },

        // 只允许选择图片文件。
        accept: {
            title: 'Images',
            extensions: 'gif,jpg,jpeg,bmp,png',
            mimeTypes: 'image/*'
        }
    });

    uploader.on( 'uploadSuccess', function(file,response  ) {

        //var id=$("#hidBtn").val();
        if(response.key){
            //alert(1);
            $("input[name='bg_img']").val($("#hidUrl").val()+"/"+response.key);
            var a_tag="<a title='点击查看' target='_blank' href='"+$("#hidUrl").val()+"/"+response.key+"'>上传成功</a>";
            $("#tips").html(a_tag);
            console.log($("#hidUrl").val()+"/"+response.key);
            console.log("-");
        }
        //重置uploader，开启此选项后
        //uploader.reset();
    });

    //生成多个token,file,此函数在多文件上传的时候只会被调用一次
    uploader.on('filesQueued',function(files ){
        var number=files.length;
        //不异步
        $.ajaxSetup({
            async : false
        });
        $.get(
            "/index.php?m=admin&c=Public&a=getTokens",
            {number:number},
            function(data){
                //console.log("data:"+data);
                CreateTokenList(data);
            }
        )
    });
    //创建tokenlist
    function CreateTokenList(data){
        $("#TokenList").remove();
        var str="<div id='TokenList' style='display:none'><ul>";
        $.each(data, function (k,v) {
            str+="<li token='"+ v['token_'+k]+"' key='"+ v['key_'+k]+"'>"+k+"</li>";
        })
        str+="</ul></div>";
        $("body").append(str);
    }
    ////图片上传的时候
    //uploader.on('fileQueued',function(file){
    //    console.log(file);
    //});

    //单个上传的时候判断。去动态的变更formatData
    uploader.on('uploadStart',function(files ){
        //生成多个token,file
        //console.log("k:"+files);
        var data=$("#TokenList").find("ul>li");
        var token="";
        var key="";
        $.each(data, function (k,v) {
            if(!$(v).attr("is_use")){
                token=$(v).attr('token');
                key=$(v).attr('key');
                $(v).attr('is_use',1);
                return false;
            }
        });

        uploader.options.formData.token = token;
        uploader.options.formData.key = key;

    });


    //上传进度
    uploader.on( 'uploadProgress', function( file, percentage ) {
        //console.log("进度:"+percentage);
        $("#tips").text("上传进度:"+percentage*100+"%");
    });


})

```

#### 前端部分：


```
<!--外部的图片转换为webuploader上传-->
    <link rel="stylesheet" href="{$b_static}Extend/webuploader/webuploader.css"/>
    <!--不含日志的。减少不必要的请求-->
    <script src="{$b_static}Extend/webuploader/webuploader.min.js"></script>
    <!--JS-->
    <script src="{$b_static}js/admin/article.js"></script>

                            <div class="control-group">
                                <label class="control-label">背景图片</label>
                                <div class="controls">
                                    <p id="tips">此处图片上传为下面的图片上传，允许多图上传</p>
                                    <span id="content_img">选择图片</span>
                                    <input type="hidden" name="content_img" />
                                </div>
                            </div>

```

#### 两个link就不解释了。一个是webuploader的样式文件，一个是js文件。 
     php，print_r的值：
     
```
   Array
   (
       [0] => Array
           (
               [token_0] => h6NVJgW4C-WLJlyhdyu_fsOBQa9dF5NZvaa35g2f:Vw4mq0PYTRSUdnBH-HOlydIqHWI=:eyJzY29wZSI6ImFydGljbGU6YXJ0aWNsZV81NmRlN2Q4YmRhOGUxNDQ0MDkxXzAiLCJkZWFkbGluZSI6MTQ1NzQyNTMwN30=
               [key_0] => article_56de7d8bda8e1444091_0
           )
   
       [1] => Array
           (
               [token_1] => h6NVJgW4C-WLJlyhdyu_fsOBQa9dF5NZvaa35g2f:2FXSS8JkojeeVDbgOJF-wTrEJG0=:eyJzY29wZSI6ImFydGljbGU6YXJ0aWNsZV81NmRlN2Q4YmRhOGUxNDQ0MDkxXzEiLCJkZWFkbGluZSI6MTQ1NzQyNTMwN30=
               [key_1] => article_56de7d8bda8e1444091_1
           )
   
   )
     
     
```
多维数组，结构是key值与里面的token_ key_ 后面的相同。这个是个人生成token 与key的习惯。自行修改就好。


说一下思路， 
uploader对象的创建是根据文档里面，之前搞的上传的方式是先传到服务器，然后服务器再去传七牛。那样还得费服务器的带宽，比较浪费，就换了一种方式。直接传七牛。 
所以 server 那边也就换成了：”http://up.qiniu.com” 
另外多图上传的时候需要在pick里面设置multiple 设置为true。开启多图上传。

filesQueued 这个事件是针对多图上传的事件。

```
    var number=files.length;

```


是为了获取上传文件的数量。因为你在那个页面，不知道要上传多少文件。所以当你选中后，获取文件数量，根据数量去后台同步的把token与key 请求回来。 
这里说下，为什么要同步，而不是异步呢，因为如果异步的话，他可能在你回调回来之前就去做上传动作了，导致上传失败。所以这边是要设置同步的。 
CreateTokenList 这个函数为的是将返回过来的token与key 存起来。

最后在 uploadStart 事件的时候，摘出其中一个token与key。去设置到上传参数formData中去。uploadStart 是一个文件执行一次，所以纵然是多图上传，也是一个个的传上去。

下面两个事件：uploadProgress 就是获取进度了。是单个文件的进度，想做特效的，就具体可以根据你文件上传进度和现在的文件数量。具体做特效了，如果想看效果，建议用360之类的，设置一下你浏览器的上传速度，就可以看到效果了。 
And…. 
图片上传成功后： 
uploadSuccess也就是这个事件的时候。 
确保你的response里面的响应带回了Key值。这样你的图片就算是上传成功了。 
使用你七牛上传对应空间的域名 ，注意理解这句话，后面跟上你的responese.key 就是你上传图片的链接了。 
比如：article空间七牛给你的对应域名是： dasd.111.com ，返回的responese.key 的值是 abc 
那么你的图片访问路径便是： dasd.111.com/abc

我是分割线：-----------------(关于php这边生成的token)

```
$data=$this->mkQiniuToken(C('Qiniu.article_bucket'),'article_',2);

    /**
     * @param $bucket.七牛存储空间
     * @param $prefix.生成的文件前缀
     * @param int $more。是否生成多个。此处填写生成的数量，多个的前缀取配置
     * return 返回生成的 token 和key
     */
    public function mkQiniuToken($bucket,$prefix,$more=''){
        import("Org.Util.Auth");
        $auth=new Auth(C('Qiniu.AK'), C('Qiniu.SK'));
        $title=$prefix;
        $file_key=$title.uniqid().rand(111111, 999999);
//        $file_key=$title.uniqid().rand(111111, 999999);;
        if($more>1){
            $arr=array();
            for($i=0;$i<$more;$i++){
                $token = $auth->uploadToken($bucket,$file_key."_".$i);
                $arr[$i]['token_'.$i]=$token;
                $arr[$i]['key_'.$i]=$file_key."_".$i;
            }
            //返回多维token与key
            return $arr;
        }
        else{
            //单一的token 与KEY
            $token = $auth->uploadToken($bucket,$file_key);

            return array('token'=>$token,'file_key'=>$file_key);
        }
    }


```

mkQiniuToken 这个函数是我自己封装的生成七牛token的函数，当然里面核心的也是根据七牛给php提供的案例去写的。此函数基于think3.2 . 用think的朋友可以拿去试试，注意自己AK与SK的秘钥别写错。另外，调用函数的时候别忘了use Qiniu\Auth; 里面的各种import。

有一点是要注意的，在生成token的时候最好不要跟key对应上。就单纯的生成KEY。因为很有可能是你前端去生成一个唯一标识。
如果你在生成token的时候指定了key.那你上传的时候就蒙逼了。

