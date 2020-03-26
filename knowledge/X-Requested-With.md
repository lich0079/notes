# X-Requested-With


最近看到一段server端的code， 依据 X-Requested-With header来判断是否 ajax 请求。
```
	public boolean isAjax(HttpServletRequest request) {
		if(request != null) {
			isAjax = "XMLHttpRequest".equals(request.getHeader("X-Requested-With"));
		}
	   
	   return isAjax;
	}

```

X-Requested-With 首先它是自定义的header，是由类似 jQuery, axios 这样的lib 发ajax请求时加上的。

所以上面的server端check要配合着 browser side 的lib来才行，如果lib没有加，你可以自己考虑加。

那么为什么jQuery要加上这个呢，有一篇文章说的比较清楚。

[CSRF Mitigation for AJAX Requests](https://markitzeroday.com/x-requested-with/cors/2017/06/29/csrf-mitigation-for-ajax-requests.html)

学习后记录如下:


* 如果在浏览器用js发跨域请求，虽然浏览器端会报错，但实际server端已经接受了，只是浏览器不处理返回结果而已，就是请求实际已经跨域，这会造成CSRF attack。

* 很多lib在ajax请求时会加上 X-Requested-With header， 而这个header在跨域请求时是不让传输的。

* 利用这一点, 每次提交ajax请求时在服务端验证： 一定要有X-Requested-With， 没有就不处理，可以防止CSRF，和token是同样的效果，但不用生产、储存和传输 token。

* 这个对form提交不管用，因为form提交时没法手动加上X-Requested-With header。  

* form 的情况下默认就是可以跨域提交的，所以还是要依靠CSRF token。

* 所以用X-Requested-With替代CSRF token来保证安全的方案只有在submit动作全部由ajax发起的情况下才能使用。


总结下来就是利用了自定义header没法跨域传输来提供了一种轻量级的CSRF解决方案。