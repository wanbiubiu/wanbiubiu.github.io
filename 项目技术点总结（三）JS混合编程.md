##项目技术点总结（三）JS混合编程

* WebView加载JS网页
	* webView.loadUrl(url)即可以加载网页（http网页或本地assets路径下网页）
				
			webView.loadUrl("file:///android_asset/../xx.html");
			webView.loadUrl("http://www.baidu.com");
	* webView.getSetings().setJavaScriptEnable(true)设置允许webview执行JS操作
	    * WebViewClient事件监听，在打开webView内链接时直接在webView内进行跳转，而不是跳转到系统浏览器打开
        
* JS交互
	* 参数交互
		* 同JS端进行交互，需要新建JSInterface接口类
		* JS端调用android函数
			* webView.addJavaScriptInterface(jsInterface,"webView_name")，添加该接口，并设置JS端对应于此webView的名称
			* JSInterface内设置被JS端调用的函数，函数前需要声明“**@JavascriptInterface**”才能被调用

					@JavascriptInterface
	    			public void showToast(){
	    				Toast.makeText(mContext, "JS调用android端", Toast.LENGTH_SHORT).show();
	    			}
     * android端调用JS端函数，通过loadUrl直接调用javascript端function并输入对应参数即可

    	 		webView.loadUrl("javascript:function("+param+")");
 
 
			**注意：**
			
			1. 必须保证在网页加载完成后再进行此交互操作，否则会显示*android Uncaught ReferenceError*，若无法保证加载完成后进行（如接收广播时）则需要为webView添加WebViewClient，重写onPageFinished()，在页面加载完成后在进行。（或使用EventBus的Sticky Event）
			
					locationMap.setWebViewClient(new LocationWebViewClient());
					class LocationWebViewClient extends WebViewClient{
				        @Override
				        public void onPageFinished(WebView view, String url) {
				            super.onPageFinished(view, url);
				            locationMap.loadUrl("javascript:LocateAddress("+currentlocation.getLongitude()+","+currentlocation.getLatitude()+")");
			        	}
			    	}
			  
		 2. 传入的param为String格式，需根据JS的写法注意是否需要加引号
		 
	* 权限请求
		* JS端操作需要请求权限时，调用android端JSInterface内函数检查权限，对应的onRequestPermissionResult(...)要重写在承载webview的activity或fragment中。
		* android6.0以上时fragment中的onRequestPermissionResult(...)被activity拦截，需要在activity中接收并传递过来
				
				@Override
			    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
			        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
			        if (Build.VERSION.SDK_INT >= 23) {
			            fragment.onRequestPermissionsResult(requestCode, permissions, grantResults);
			        }
			    }
			    
		* 注意JSInterface中chechPermission与activity中onRequestPermissionResult都要加上权限请求的后续操作
	* Activityresult
		* JS端需要跳转到其它activity或应用中时，可在JSInterface中直接通过((Activity)mContext).startActivityForResult(intent,requestCode)进行跳转
		* 在接收onActivityResult时，需要在webview所属的activity接收
		* 若webview处于fragment中，此时fragment的onActivityResult被所属Activity的onAcctivityResult拦截，需要手动从activity中传递给fragment

				@Override
			   	protected void onActivityResult(int requestCode, int resultCode, Intent data) {
			        super.onActivityResult(requestCode, resultCode, data);
			
			        switch (requestCode) {
			            case StaticValue.CAMERA_REQUEST_CODE:
			                handleDatatoBlock(requestCode,resultCode, data);
			                break;
			            case StaticValue.GALLERY_REQUEST_CODE:
			                handleDatatoBlock(requestCode, resultCode, data);
			                break;
			        }
			    }
			
			    private void handleDatatoBlock(int requestCode, int resultCode, Intent data){
			        Fragment fragment = fragmentManager.findFragmentByTag("block");
			        if (fragment != null) {
			            handleResult(fragment, requestCode, resultCode, data);
			        }
			    }
			
			    private void handleResult(Fragment fragment, int requestCode, int resultCode, Intent data){
			        fragment.onActivityResult(requestCode,resultCode,data);
			    }
			     
* [WebView缓存机制](https://www.jianshu.com/p/5e7075f4875f)
	* android自带的webview缓存机制
		* **浏览器缓存**：通过系统浏览器内核，由WebView内置自动实现
		* **Application Cache机制**：html文件在文件头中指定需要缓存的文件，webView.getSettings()设置缓存路径、大小等参数
		* **Dom Storage机制**：key-value存储简单数据，webView.getSettings().setDomStorageEnable(true)即可
		* **Web SQL Database机制**：基于SQL数据库，目前已不推荐使用
		* **Indexed Database机制**：4.4后取代Web SQL Database，基于NoSQL键值对存储，开启setJavaScriptEnable(true)支持执行JS即可
		* **File System机制**：提供虚拟文件系统存储h5页面数据，android WebView暂不支持
* 异步WebView：针对WebView页面数据量大、内存溢出、崩溃等问题
* [参考1](https://www.jianshu.com/p/d2f5ae6b4927)
* [参考2](http://blog.csdn.net/carson_ho/article/details/52693322)
* [其它坑](https://www.zhihu.com/question/31316646)