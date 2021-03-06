###写在前面
>fetch 同 XMLHttpRequest 非常类似，都是用来做网络请求。但是同复杂的XMLHttpRequest的API相比，fetch使用了Promise，这让它使用起来更加简洁，从而避免陷入”回调地狱”。

###两者比较
>比如，如果我们想要实现这样一个需求：请求一个URL地址，获取响应数据并将数据转换成JSON格式。使用fetch和XMLHttpRequest实现的方式是不同的。

###使用XMLHttpRequest实现
>使用XMLHttpRequest来实现改功能需要设置两个监听函数，分别用来处理成功和失败的情况，同时还需要依次调用open()和send()方法才能实现请求。


		function reqListener() {
		    var data = JSON.parse(this.responseText);
		    console.log(data);
		}
		function reqError(err) {
		    console.log('Fetch Error : %S', err);
		}
		var oReq = new XMLHttpRequest();
		oReq.onload = reqListener;
		oReq.onerror = reqError;
		oReq.open('get', './api/some.json', true);
		oReq.send();


###使用fetch实现
####使用fetch来实现是这样的：

	fetch('./api/some.json')
    .then(function(res) {
        if (res.status !== 200) {
            console.log('Looks like there was a problem. Status Code: ' + res.status);
            return;
        }

        // 处理响应中的文本信息 
        res.json().then(function(data) {
            console.log(data);
        });
    })
    .catch(function(err) {
        console.log('Fetch Error : %S', err);
    })
>在将响应的文本信息转换成JSON格式前，需要先确保响应的状态码为200。 
fetch()请求后返回的响应是一个stream对象，这就意味着我们在调用json()方法后会返回一个Promise，因为读取stream的过程是异步操作的。

###响应中的元数据
在上面的例子中，我们可以查看响应对象的状态码，也知道了如何将响应转换成JSON格式的数据。但其实我们可以访问的元数据还有以下这些：

		fetch("users.json").then(function(res) {
		    console.log(res.headers.get('Content-Type'));
		    console.log(res.headers.get('Data'));
		
		    console.log(res.status);
		    console.log(res.statusText);
		    console.log(res.type);
		    console.log(res.url);
		})

###响应类型
当我们发送`fetch`请求时，返回的`res.type`可能是`basic、cors`或`opaque`中的一种。这些类型可以告知我们资源从何而来，这样就能知道该如何处理响应对象。

当我们请求的是同一域下的资源时，响应返回的类型为`basic`，此时没有任何限制，我们可以查看响应中的任何数据。

如何请求的是跨域资源，那么会返回一个`CORS`类型的头部，并且响应类型为`cors`。这种类型跟上面的`basic`非常相似，只是它对响应头部的字段访问有限制，你只可以访问这些属性：
		
* Cache-Control
* Content-Language
* Content-Type
* Expires
* Last-Modified
* Pragma
>`Opaque` 类型的响应也是访问跨域资源的时候产生的，只是响应头不是CORS类型的响应头而已。如果是这种类型的响应，那么我们就不能读取返回的数据，也不能查看请求的状态码，这就是意味着我们将无法确定请求是成功了还是失败了。目前在`fetch()`的实现中，无法请求跨域的资源。

我们可以为fetch请求定义mode属性，来保证只有符合条件的请求才会被处理。可以设置的mode属性值如下：

* same-origin 只有请求相同域下的资源才能成功，其他请求均被拒绝。
* cors 允许请求同域或者跨域资源，但是跨域必须返回相应的跨域请求头部。
* cors-with-forced-preflight 在发出实际请求前先做preflight检查。
* no-cors 针对跨域资源做请求，但是不返回CORS的响应头，这是属于opaque类型的响应（window下无法使用）


在使用mode时，需要将fetch请求的第二个参数作为配置对象，并在其中配置具体的模式，如下代码：
		
		fetch("http://some-site.com/cors-enabled/some.json",{mode: 'cors'})
		    .then(function(res){
		        return res.text();
		    })
		    .then(function(text) {
		        console.log('Request successfully', text);
		    })
		    .catch(function(err) {
		        console.log('Request failed', error)
		    })

###Promise 方法链
`Promise` 的特性之一就是可以实现链式调用，`fetch`也可以使用该特性，同时，使用链式调用可以让请求的处理逻辑更加通用。

如果使用接口反馈的`JSON`格式数据，那么针对每次响应，我们都需要检查响应状态并做`JSON`格式转换。其实还能简化代码，那就是把状态监测和`JSON`转换的代码放到单独的函数中去。比如：

		function status(res) {
		    if (res.status >= 200 && res.status < 300) {
		        return Promise.resolve(response)
		    } else {
		        return Promise.reject(new Error(res.statusText))
		    }
		}
		
		function json(res) {
		    return res.json()
		}
		
		fetch('user.json')
		    .then(status)
		    .then(json)
		    .then(function(data) {
		        console.log('Request succeeded with JSON response', data);
		    })
		    .catch(function(err) {
		        console.log('Request failed', error);
		    })

在上面的代码中，我们定义了一个status方法来检查response.status 的状态，根据结果不同返回Promise.resolve() 或 Promise.reject() ，这是fetch()方法链中的第一个方法调用。如果返回resolve状态，我们会继续调用json()，从而返回response.json()的执行结果。经过这些处理完我们已经可以获取解析后的JSON对象。如果解析失败，Promise返回reject状态，执行catch里的代码进行错误处理。

这么编码更大的好处在于，你可以在所有的fetch请求中使用上面的逻辑代码，从而让代码变得更加容易阅读、维护和测试。

###使用fetch 请求发送凭证信息

如果我们想在fetch请求中带一些凭证信息，比如`cookie`等，我们应该将请求中的`credentials`设置为`include`：

		fetch(url, {
		    credentials: 'include'
		})

>注意： 服务器返回 400，500 错误码时并不会 reject，只有网络错误这些导致请求不能完成时，fetch 才会被 reject。

###Response相关属性及方法
####bodyUsed
1. 标记返回值是否被使用过
2. 这样设计的目的是为了之后兼容基于流的API，让应用一次消费data，这样就允许了JavaScript处理大文件例如视频，并且可以支持实时压缩和编辑。

		fetch('/test/content.json').then(function(res){
		    console.log(res.bodyUsed); // false
		    var data = res.json();
		    console.log(res.bodyUsed); //true
		    return data;
		}).then(function(data){
		    console.log(data);
		}).catch(function(error){
		    console.log(error);
		});

###headers
>返回Headers对象，该对象实现了Iterator，可通过for…of遍历

	fetch('/test/content.json').then(function(res){
	    var headers = res.headers;
	    console.log(headers.get('Content-Type')); // application/json
	    console.log(headers.has('Content-Type')); // true
	    console.log(headers.getAll('Content-Type')); // ["application/json"]
	    for(let key of headers.keys()){
	        console.log(key); // datelast-modified server accept-ranges etag content-length content-type
	    }
	    for(let value of headers.values()){
	        console.log(value);
	    }
	    headers.forEach(function(value, key, arr){
	        console.log(value); // 对应values()的返回值
	        console.log(key); // 对应keys()的返回值
	    });
	    return res.json();
	}).then(function(data){
	    console.log(data);
	}).catch(function(error){
	    console.log(error);
	});

###ok
1. 是否正常返回
2. 代表状态码在200-299之间

###status
状态码( 200表示 成功)

###statusText
状态描述 (‘OK’表示 成功)

###type
* basic：正常的，同域的请求，包含所有的headers。排除Set-Cookie和Set-Cookie2。
* cors：Response从一个合法的跨域请求获得，一部分header和body可读。
* error：网络错误。Response的status是0，Headers是空的并且不可写。当Response是从 Response.error()中得到时，就是这种类型。
* opaque： Response从"no-cors"请求了跨域资源。依靠Server端来做限制。
###url
返回完整的url字符串。如：’http://xxx.com/xx?a=1’

###arrayBuffer()
返回ArrayBuffer类型的数据的Promise对象

###blob()
返回Blob类型的数据的Promise对象

###clone()
* 生成一个Response的克隆
* body只能被读取一次，但clone方法就可以得到body的一个备份
* 克隆体仍然具有bodyUsed属性，如果被使用过一次，依然会失效

		fetch('/test/content.json').then(function(data){
		    var d = data.clone();
		    d.text().then(function(text){
		        console.log(JSON.parse(text));
		    });
		    return data.json();
		}).then(function(data){
		    console.log(data);
		}).catch(function(error){
		    console.log(error);
		});

###json()
返回JSON类型的数据的Promise对象

###text()
返回Text类型的数据的Promise对象

###formData()
返回FormData类型的数据的Promise对象

###fetch基本用法
####get方式
		fetch('/test/content.json').then(function(data){
		    return data.json();
		}).then(function(data){
		    console.log(data);
		}).catch(function(error){
		    console.log(error);
		});

####post方式
		fetch('/test/content.json', { // url: fetch事实标准中可以通过Request相关api进行设置
		    method: 'POST',
		    mode: 'same-origin', // same-origin|no-cors（默认）|cors
		    credentials: 'include', // omit（默认，不带cookie）|same-origin(同源带cookie)|include(总是带cookie)
		    headers: { // headers: fetch事实标准中可以通过Header相关api进行设置
		        'Content-Type': 'application/x-www-form-urlencoded' // default: 'application/json'
		    },
		    body: 'a=1&b=2' // body: fetch事实标准中可以通过Body相关api进行设置
		}).then(function(res){ res: fetch事实标准中可以通过Response相关api进行设置
		    return res.json();
		}).then(function(data){
		    console.log(data);
		}).catch(function(error){
		
		});

###fetch不足之处
####无法监控读取进度和中断请求
>Promises缺少了一些重要的XMLHttpRequest的使用场景。例如， 使用标准的ES6 Promise你无法收集进入信息或中断请求。而Fetch的狂热开发者更是试图提供Promise API的扩展用于取消一个Promise。 这个提议有点自挖墙角的意思，因为将这将让Promise变得不符合标准。但这个提议或许会导致未来出现一个可取消的Promise标准。 但另一方面，使用XMLHttpRequest你可以模拟进度（监听progress事件），也可以取消请求（使用abort()方法）。 但是，如果有必要你也可以使用Promise来包裹它