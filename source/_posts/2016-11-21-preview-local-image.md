---
layout:      post
title:      "How to preview local image and beautify style"
subtitle:   "预览本地图片（取消）+input样式美化"
date:       2016-11-02
author:     "Chingsalt"
catalog:     true
tags:
     - 15级
     - 杂文
---

# 预览功能：input[type=file]的onchange事件

## JS部分
### 预览效果
要做到预览本地图片，最重要的就是获取到文件的路径。

    var clear = document.getElementById('clear_img');
    var img = document.getElementById('preview');  //img
    var file = document.getElementById('upload_img');  //input


#### 获取文件路径  

    file.onchange = function() {
    	console.log(getObjectUrl(this.files[0]));
    	var src = getObjectUrl(this.files[0]);
    	img.src = src;
        }
    
    function getObjectUrl(file) {
        var url = null;
        if(window.createObjectURL != undefined) {
            url = window.createObjectURL(file);
        }
        else if(window.URL != undefined) {
            url = window.URL.createObjectURL(file);
        }
        else if(window.webkitURL != undefined) {
            url = window.webkitURL.createObjectURL(file);
        }
    	return url;
        } 

### 取消预览
即使file的value=null；

    clear.onclick = function() {
    	img.src = "";
    	file.value = null;
    }

## HTML

    <div  id="preview_fake" style="position:absolute;left:640px;top:150px">
		<img id="preview" onload="onPreviewLoad(this)" width="140" height="140"/>  
		<div class="img-btn">
			<a href="javascript:;" class="a-upload ">选择文件
				<input id="upload_img" type="file" onchange="onUploadImgChange(this)"/></a> 
	    	<a href="javascript:;" class="a-clear ">取消
	    		<input id="clear_img" type="button" value="取消" onclick="" /></a>					
		</div>  
    </div>
    
# input file样式美化
就是在input外面用a标签封装起来，然后将我们想要的样式写在a标签上，用a标签来覆盖掉原本的input（opacity：0）所在的位置。

    .img-btn {
    	margin-top: 10px;
    	color: #ffffff;
    }
    .a-clear {
        position: relative;
        display: inline-block;
        background: #61A3E1;
        border-radius: 4px;
        padding: 4px 12px;
        overflow: hidden;
        text-decoration: none;
        text-indent: 0;
        line-height: 20px;
    } 
    .a-clear input {
        position: absolute;
        font-size: 100px;
        right: 0;
        top: 0;
        opacity: 0;
    }
    .a-upload  {
        position: relative;
        display: inline-block;
        background: #61A3E1;
        border-radius: 4px;
        padding: 4px 12px;
        overflow: hidden;
        text-decoration: none;
        text-indent: 0;
        line-height: 20px;
    }
    .a-upload input {
        position: absolute;
        font-size: 100px;
        right: 0;
        top: 0;
        opacity: 0;
    }
    
    
记录一下自己这次使用的一个功能吧，写前觉得懵逼写后觉得根本没啥好写的。。希望能找到一个更好的方法，然后去做一下缩略图的功能（即图片等比例缩放）~
