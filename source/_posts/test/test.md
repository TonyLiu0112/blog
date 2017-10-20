---
title: 客户申请API说明
---
> NODE: 提供客户申请接口，存储申请客户信息

* 请求地址
	
```
http://www.findmud.com:38080/guard/api/customer/application/save
```
	
* 请求类型

````
POST
````

* 示例
 
``` 
	$.ajax({
	  url: 'http://www.findmud.com:38080/guard/api/customer/application/save',
	  dataType:"json",
	  contentType: 'application/json',
	  type:'post',
	  data:JSON.stringify({
	    name: 'TestName',
	    telephone: '13888888888',
	    company: 'TestCompany',
	    mail: 'testmail@gmai.com'
	  }),
	  success:function(response){
	    console.log(response);
	    // do youself...
	  },
	  error:function(){
		 // do youself...
	  }
	});
```