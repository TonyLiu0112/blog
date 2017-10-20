# 客户申请API说明

> NODE: 提供客户申请接口，存储申请客户信息

* 请求地址
	
		http://www.findmud.com:38080/guard/api/customer/application/save
	
* 请求类型

		POST

* 请求参数

	| 字段名     | 类型           | 必填   | 说明       | 
	| ----------|:-------------:| -----:|-----------:|
	| name      | String        | Yes   |联系人姓名    |
	| telephone | String        | Yes   |联系人号码    |
	| company   | String        | No    |公司名称      |
	| mail      | String        | No    |联系人邮箱地址 |

* 响应参数

	| 字段名     | 类型           | 必填   |    说明           | 
	| ----------|:-------------:| -----:|------------------:|
	| msg       | String        | Yes   |  响应消息          |
	| code      | String        | Yes   |  响应码            |
	| results   | String        | No    |  Null             |

* 响应码

	| 响应码     | 说明           | 
	| ----------|:-------------:| 
	| 0000      | 成功           | 
	| 0004      | 用户已参与      | 
	| 其他       | 失败           |

* 示例
 
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