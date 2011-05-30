---
layout: post
title: iframe无刷新跨域上传文件并获取返回值
category: tech
---

通常我们会有一个统一的上传接口，这个接口会被其他的服务调用。如果出现不同域，还需要无刷新上传文件，并且获取返回值，这就有点麻烦了。比如，新浪微博启用了新域名www.weibo.com，但接口还是使用原来的域：picupload.t.sina.com.cn。

研究了一下新浪微博的处理方法，这里大概演示一下。

首先是一个正常的上传页面 upload.html

{% highlight html %}
<script>
	// 这个函数将来会被iframe用到
	function getIframeVal(val)
	{
		alert(val);
	}
</script>

<!-- 我把upload.com指向了127.0.0.1 -->
<form method="post" target="if" enctype="multipart/form-data" action="http://upload.com/playground/js/deal.php?cb=http://localhost/playground/js/deal_cd.html">
	<input type="file" name="file" />
	<input type="SUBMIT" value="upload" />
</form>
<IFRAME id="if" name="if" src="about:blank" frameborder='0'></IFRAME>
{% endhighlight %}

这里有一个关键点是form的target要指向iframe，同时把iframe隐藏起来，这样上传的处理结果就会显示在该iframe里。action里的cb(callback)参数表示处理完成后要跳转的url，因为我们的目标是iframe，所以只会把跳转的页面输出到iframe，而不会让当前页面跳转。

还有一点，callback url要和当前页面同域。跨域的iframe无法调用父页面的内容。

再来看看deal.php，也就是form的action

{% highlight php %}
<?php
// deal upload file
// and get file id, you can pass other params either
header('location:'.$_GET['cb'].'?file_id=123');
{% endhighlight %}

这里可以处理文件，然后入库。操作完成后，把文件的id及其他信息都放在url里，最后跳转到这个url。

最后来看看deal_cd.html，也就是刚刚deal.php跳转到的url，这个文件的内容会填充到页面的iframe里。

{% highlight html %}
<script type="text/javascript">
	var rs = window.location.search.split('?').slice(1);
	window.parent.getIframeVal(rs.toString().split('=').slice(1));
</script>
{% endhighlight %}

这里调用了父窗口的getIframeVal方法，这样父页面就获得了文件的id。
