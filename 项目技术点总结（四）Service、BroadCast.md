##项目技术点总结（四）Service、BroadCast

* [Service](http://blog.csdn.net/guolin_blog/article/details/11952435/)
	* Service的常用启动方法
		* 使用时首先在Manifest中注册，继承Service并重写onCreate()、onStartCommand()和onDestroy()方法
		
			> 生命周期：onCreate() onStartCommand() onDestory()
		
		* 从activity或Fragment通过Intent跳转开启服务。开启后，Service通过Binder、重写onBind()与activity通信

			> 生命周期：onCreate() onBind() onUnBind() onDestory()
		
		* 开启后Service与开启者独立，开启者不能调用Service内方法
	* [Service的另一种启动方式](https://www.jianshu.com/p/2fb6eb14fdec)
		* Manifest中注册过后，在context中通过bindService()与unbindService()开启或结束服务 
		* 重写onCreate(),onBind(),onUnbind(),onDestory()方法
		* 此方法中Service活动依赖于开启者，开启者可以调用Service中的方法
	* **Service在主线程中运行**，但是运行完全不依赖于UI，因此只要进程存在，即使Activity销毁或程序关闭也能继续运行。
	* 若需要进行复制操作，需要在Service中开辟子线程，以防出现主线程ANR。即使原activity被销毁，其它activity也可以通过Binder与其进行通信
	* Service优先级较低，可能会被系统杀死，为保持Service一直运行，可以设置为前台服务*startForeground()；*并添加状态栏通知
	* [远程Service](http://blog.csdn.net/guolin_blog/article/details/9797169) 在Manifest对应Service中设置*android:process=":remote"*即可，开启的Service相当于在另一进程的主线程中运行，此时与程序主线程的通信较难实现，需要借助AIDL跨进程通信进行
* IntentService
	* 重写onCreate(),onHandleIntent(), onDestory()
	* 在onHandleIntent()内通过HandlerThread进行异步操作，因此可以执行耗时操作，任务完成后自动停止
	* 优先级较高，不易被杀死
	* 当前任务未完成时，新的任务要等待其完成后再开始，不会再开辟一个线程进行新任务

* [BroadCast](https://www.jianshu.com/p/ca3d87a4cdf3)
	* 应用内及不同应用间组件的通信、线程间通信等
	* 采用了观察者模式（信息发布／事件订阅模型），发送端与接收端解耦，通过AMS对全局发送端广播的要求，在已注册的列表中寻找符合的接收端，调用接收端onReceive()处理广播信息
	![](https://upload-images.jianshu.io/upload_images/944365-9fca9fd3978cef10.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)
	* 发送端
		* 发送intent，将intent作为消息载体广播出去
		* Intent携带String action（作为广播接收条件，与IntentFilter相匹配）与消息主题extra
		* getApplicationContext().sendBroadcast(intent)发送
		* 其他发送模式：
			* 系统广播：针对系统的操作如监听网络、飞行模式、插入耳机等，使用系统内置的intent，只需指定intent的特定action即可
			* 有序广播：接收端按顺序接收，优先级较高的接收端可以对后续广播进行截断或修改
			* app内部广播：普通广播在不同进程内直接通信，而app内部广播仅在app内传播或仅接收app内部广播，安全性及通信效率更高
	* 接收端BroadCastReceiver
		* 四大组件之一，需要进行注册：
			* 静态：直接在manifest中添加<receiver>标签，耗电占内存，需要持续监听时使用
			* 动态：在代码中调用mContext.registerReceiver(receiver,filter) ，并在不需要时手动注销mContext.unregiserReceiver(receiver)，否则会导致内存泄漏。在仅特定时刻需要监听时使用
		* 构造receiver实例时需要继承BroadcastReceiver并重写onReceive(Context context, Intent intent)，在其中对获取到的intent进行处理
		* 注意不能重复注册，在activity中说明全局变量，判断是否已注销后再注册
		* 在onPause中注销，onDestroy与onStop不一定会执行（内存不够被系统销毁时），在其中注销可能不成功导致内存泄漏，而onPause则一定会执行。注意不能重复注销
	* AMS 

* 位置服务
	* 使用IntentService获取位置信息，在一个activity与一个Java类中获取并处理后台服务广播的位置信息
	* LocationService
	
			public class LocationService extends IntentService{
			
			    LocationManager locationManager;
			    Location location;
			    Context mContext;
			
			   public LocationService(){
			       super("LocationService");
			   }
			
			    @Override
			    public void onCreate() {
			        super.onCreate();
			        mContext = this;
			
			    }
			
			    @Override
			    protected void onHandleIntent(Intent intent) {
			        getCurrentLocation(new Criteria());//获取定位信息
			    }
				
			    @Override
			    public void onDestroy() {
			        super.onDestroy();
			
			    }
			}
			
	* activity中接收广播
	
			class LocationReceiver extends BroadcastReceiver {
		        @Override
		        public void onReceive(Context context, Intent intent) {
		            currentlocation = intent.getParcelableExtra("newLocation");
		            System.out.println("locationReceived:" + currentlocation);
		
		            try{
		                longitude = currentlocation.getLongitude();
		                latitude = currentlocation.getLatitude();
		
						//是否为网页初始化完成后的重新获取的数据
		                if (webFlag){
		                    locationMap.loadUrl("javascript:LocateAddress("+longitude+","+latitude+")");
		                }
		                System.out.println("javascript:LocateAddress("+longitude+","+latitude+")");
		            }catch (Exception e){
		                e.printStackTrace();
		            }
		
		        }
		    }

		由于接收到广播的时间不确定，可能在网页加载完成前就接收到，因此直接在onReceive中调用loadUrl会导致js端找不到对应函数并报undefine的错误，因此需要采用全局变量获取广播数据，通过WebViewClient在页面加载完成后再调用loadUrl
		
	* Java类中接收广播
	 	
	 		//JSInterface
	 		
	 		@JavascriptInterface
    		public void blockLocate(){
    			//请求位置权限
    			 ((MainActivity)mContext).startLocationService();
    		}
    		
    		//MainActivity
    		
    		public void startLocationService(){
		       if (!flag) {
		       		MainActivity.this.registerReceiver(locReceiver, intentFilter);
               		flag = true;
		       }
		        //启动后台服务获取实时位置
		        locManager = (LocationManager) MainActivity.this.getSystemService(Context.LOCATION_SERVICE);
		        Intent intent = new Intent(MainActivity.this, LocationService.class);
		        MainActivity.this.startService(intent);
		    }
		    
			//接收地块定位服务返回位置数据
		    class LocationReceiver extends BroadcastReceiver {
		        @Override
		        public void onReceive(Context context, Intent intent) {
		            Location currentlocation = intent.getParcelableExtra("newLocation");
		            System.out.println("locationReceived:" + currentlocation);
		            lon = currentlocation.getLongitude();
		            lat = currentlocation.getLatitude();
		            //locationInfo.setText("经度：" + +"\n纬度："+ );
		            //定位信息传入blockfragment中
		            LocInterface locInterface = (LocInterface)fragmentManager.findFragmentByTag("block");
		            locInterface.onReceiveLoc(lon,lat);
		
		        }
		    }
				    
			public interface LocInterface{
		        public void onReceiveLoc(double lon, double lat);
		    }
		
		    @Override
		    protected void onPause() {
		        super.onPause();
		        if (flag) {
		           MainActivity.this.unregisterReceiver(locReceiver);
		           MainActivity.this.stopService(new Intent(MainActivity.this, LocationService.class));
		           Log.d("MainActivity","unregisterReceiver,stopService");
		           flag = false;
		        }
		    }
		    
		    //BlockFragment
		    
		    @Override
		    public void onReceiveLoc(double lon, double lat) {
		      	block.loadUrl("javascript:getLocation("+lat+","+lon+")");
		    }
	 
	 在网页端请求定位服务，在JSInterface中调用传入的Context（即MainActivity）中函数注册接收端并开启服务，在JS页面BlockFragmeent中调用MainActivity中接口监听
* 文件下载服务