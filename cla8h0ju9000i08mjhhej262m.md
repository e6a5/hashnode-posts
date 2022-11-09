# Solving CORS errors

There are a lot of solutions for CORS errors such as disabling CORS on the browser, disabling browser web security, or enabling CORS on the server. Disabling something to solve CORS errors is a really bad idea, it will increase the risks of cross-origin HTTP requests. For this reason, I will only focus on solving CORS errors by enabling CORS on the server.
## No 'Access-Control-Allow-Origin' header is present on the requested resource
![allow-origin.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667926444932/IHIxkwy3A.png align="left")
The first error that I want to mention is `No 'Access-Control-Allow-Origin' header is present on the requested resource`. This error occurs when calling the other server (cross-origin request) which does not set access control from your origin, the browser will restrict that cross-origin request.  To solve this problem, your server must set access control to allow the origin that you want to make a request.  In my example, I have a request from the origin `http://localhost:8080` to the origin `http://localhost:8000/users` to get a list of users.

```jsx
async function getUsers() {
 // a cross-origin request from origin http://localhost:8080
	let response = await fetch("http://localhost:8000/users")
	let data = await response.json()
	return data
}
getUsers().then((data) => {
	let name = document.getElementById("name")
	name.innerHTML = data[0].name
	let age = document.getElementById("age")
	age.innerHTML = data[0].age
}).catch((err) => {
	let errortag = document.getElementById("error")
	errortag.innerHTML = err.message
})
```

To be able to show the list of users on the document, I must set access control to allow access from the origin `http://localhost:8080` at the origin `http://localhost:8000`

```go
func handleMyList(w http.ResponseWriter, res *http.Request) {
	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:8080")
	....
}
```

After setting “Access-Control-Allow-Origin” on my server, my client is able to see the list of available users.

## Request header field content-type is not allowed by Access-Control-Allow-Headers in preflight response

I want to use the data responses as JSON data on the server, so I use the `Content-Type: application/json` header when calling API at the client.

```jsx
async function getUsers() {
 // a cross-origin request from origin http://localhost:8080
	let response = await fetch("http://localhost:8000/users", {
			headers: {
				"content-type": "application/json"
			}
	})
	let data = await response.json()
	return data
}
```

But the browser shows an error `Request header field content-type is not allowed by Access-Control-Allow-Headers in preflight response`

![allow-header.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667926582565/1yDC2o_Yr.png align="left")
According to [preflight requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#preflighted_requests), a preflight request is sent by the browser if:

- **The method is:**
    - PUT
    - DELETE
    - CONNECT
    - OPTIONS
    - TRACE
    - PATCH
- **Or if it has a header other than:**
    - Accept
    - Accept-Language
    - Content-Language
    - Content-Type
    - DPR
    - Downlink
    - Save-Data
    - Viewport-Width
    - Width
- **Or if it has a `Content-Type` header other than:**
    - application/x-www-form-urlencoded
    - multipart/form-data
    - text/plain

If any of the conditions above are met, the browser sends first an HTTP request using the `[OPTIONS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/OPTIONS)` method to the resource on the other origin, in order to determine if the actual request is safe to send.

![missing-content-type.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667926664886/kEriblrKG.png align="left")
In the image above, the browser sent a preflight request to ask the server that the request headers `content-type` and the request method `GET` are allowed to access. But in response headers, the server does not respond with any information relating to the request headers and the request method. Therefore, the browser will block the actual request and throw an error `Request header field content-type is not allowed by Access-Control-Allow-Headers in preflight response.` to the browser console.

To fix this issue, you must set `Access-Control-Allow-Methods` and `Access-Control-Allow-Headers` to accept the value which is used by the client. For my case, I will set `Access-Control-Allow-Methods: GET, OPTIONS`, and `Access-Control-Allow-Headers: "Content-Type"`

```go
func handleMyList(w http.ResponseWriter, res *http.Request) {
	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:8080")
	w.Header().Set("Access-Control-Allow-Methods", "GET, OPTIONS")
	w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
	....
}
```

After enabling `Access-Control-Allow-Methods` and `Access-Control-Allow-Headers` in my server, the returned origin is matching to all access control requests in Request Headers in the preflight request. So the actual request will be made.

![after-add-allow-header.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1667926706595/-QCQ-M4e_.png align="left")
## The value of the 'Access-Control-Allow-Credentials' header in the response is '' which must be 'true' when the request's credentials mode is 'include’

I want to use some data from cookies on the client. To do that, I set the credentials mode in the request is ‘include’ like the following code.

```jsx
async function getUsers() {
 // a cross-origin request from origin http://localhost:8080
	let response = await fetch("http://localhost:8000/users", {
			headers: {
				"content-type": "application/json"
			},
			credentials: "include"
	})
	let data = await response.json()
	return data
}
```

When I try to fetch data from the server, the browser throws an error `The value of the 'Access-Control-Allow-Credentials' header in the response is '' which must be 'true' when the request's credentials mode is 'include’`. When a request's credentials mode is `include`, browsers will only expose the response to the frontend JavaScript code if the `Access-Control-Allow-Credentials`value is `true`. The error is similar to the preceding errors, we set `Access-Control-Allow-Credentials` to be true in response headers to solve that error.

```go
func handleMyList(w http.ResponseWriter, res *http.Request) {
	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:8080")
	w.Header().Set("Access-Control-Allow-Methods", "GET, OPTIONS")
	w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
	w.Header().Set("Access-Control-Allow-Credentials", "true")
	....
}
```

After setting `Access-Control-Allow-Credentials` value equal `true`, I want to validate a value from cookies in the request. If the server can not that value from cookies, it will return a response with an HTTP status code is 400 (Bad Request).

To test it, I set my expected value to document.cookie

```html
<script>
      document.cookie = "test1=Hello; SameSite=None; Secure"
 </script>
```

And I get the value from cookies in the request

```go
func handleMyList(w http.ResponseWriter, res *http.Request) {
	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:8080")
	w.Header().Set("Access-Control-Allow-Methods", "GET, OPTIONS")
	w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
	w.Header().Set("Access-Control-Allow-Credentials", "true")
	test1, err := res.Cookie("test1")
	if err != nil || test1 == nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}
	....
}
```

But the browser throws an error to the console Response to the preflight request doesn't pass the access control check: It does not have HTTP ok status. I check the request in the networking tab in my browser, and the preflight request is returning with status code 400. It means the server does not get the value from cookies in the request. According to this [spec](https://fetch.spec.whatwg.org/#cors-protocol-and-credentials), a [CORS-preflight request](https://fetch.spec.whatwg.org/#cors-preflight-request) never includes [credentials](https://fetch.spec.whatwg.org/#credentials). Therefore, the server should not validate the expected value in the preflight request and return status code success (2xx).

```go
func handleMyList(w http.ResponseWriter, res *http.Request) {
	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:8080")
	w.Header().Set("Access-Control-Allow-Methods", "GET, OPTIONS")
	w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
	w.Header().Set("Access-Control-Allow-Credentials", "true")
	if res.Method == "OPTIONS" {
		w.WriteHeader(http.StatusNoContent)
		return
	}
	test1, err := res.Cookie("test1")
	if err != nil || test1 == nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}
	....
}
```

## Conclusion

In my example, I solved the CORS errors for a specific path handler, it is not a best practice. Because your server does not have only one path, there are lots of path handlers. You could not handle all CORS errors for each path handler. You should handle them in the middleware before you handle the request in the handlers. My code is available [here](https://github.com/tranhiepqna/solving-cors-erros).
