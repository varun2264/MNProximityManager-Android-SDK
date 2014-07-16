Android Proximity Manager SDK 0.1.0
==============

The Android Proximity Manager, _APM_ from now on, is a library dedicated to manage the interaction between an App and Mobiquity's beacons network. The integration of this SDK will enable your App to react based on the proximity to a beacon of yours or Mobiquity's. At the time of this writing the SDK version is **0.1.0**, so some deep changes to the api might come up in future versions.

## Release notes
####0.1.0  
* First release (alpha).



## Environment

Following you will find the requirements to start programming an App which benefits of Mobiquity's proximity network thanks to _APM_: 

* First, you will need to install Android support for your IDE of choice (Eclipse, Intellij, ..) usually in the form of a plugin.
* Second, you will need the Android SDK with api level 18 or higher installed on your IDE, usually from the Android SDK Manager.
* Third, you will need to have installed the Android SDK Tools which let you debug and develop with a real device, usually from the Android SDK Manager as well.  
* Finally, to develop you will also need a device with a Bluetooth Low Energy compatible chipset running Android 4.3 or higher.


## Project setup

All following instructions assume you have created an Android project in your IDE. The lines below explain how you must configure the project to start developing. As the SDK depends on some other jars you will also need to include them in your project together with the _APM_ jar file  **mnapm.jar**. 

You can go through this process by manually downloading the jars from the Internet or by using a dependency control manager like Maven or Gradle. No matter how but you must end up with all these jars on your project's build path:

* mnapm-0.1.0.jar
* retrofit-1.4.1.jar
* guava-17.0.jar
* gson-2.2.4.jar
* google play services jar

The next thing to do is to set up the AndroidManifest.xml file with some mandatory information that the _APM_ needs in order to work properly:

* Permissions required by the App: 

```
<permission 
	android:name="mobiquitynetworks.permission.SERVICE" 
	android:label="mobiquity service permission" 
	android:protectionLevel="signature" />
<permission 
	android:name="mobiquitynetworks.permission.BROADCAST" 
	android:label="mobiquity broadcast permission" 
	android:protectionLevel="signature" />
     
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<uses-permission android:name="android.permission.INTERNET"/> 
<uses-permission android:name="android.permission.GET_TASKS"/>
<uses-permission android:name="mobiquitynetworks.permission.SERVICE"/>
<uses-permission android:name="mobiquitynetworks.permission.BROADCAST"/>

```

* Receiver responsible of booting the underlying service up on several system conditions:

```
<receiver 
	android:name="com.mobiquitynetworks.receivers.MobiquityReceiver"
	android:exported="true">
	<intent-filter>
		<action android:name="android.intent.action.BOOT_COMPLETED"/>
		<action android:name="android.bluetooth.adapter.action.STATE_CHANGED"/>
	</intent-filter>
</receiver>

```
* Services to add:

```
<service 
	android:name="com.mobiquitynetworks.service.MobiquityService" 
	android:exported="false" />
         
<service 
	android:name="com.mobiquitynetworks.service.MonitoringService" 
	android:exported="false" />

```

To understand why you have to add all these components to your Manifest you should know that _APM_'s heavy duty is based on an underlying service capable of monitoring the environment on background, ranging on foreground, sendind statistics and downloading relevant content when a beacon is found. So, all these components assist the _APM_'s forementioned service to get the job done: booting, staying awake and scanning the environment with a minimal footprint on the device battery life.

The final step of the setup is to create a properties file in **assets/properties/mobiquity.properties** where you will place your api key and secret that you got from the [management panel](http://panel.mobiquitynetworks.com). 


```
	mobiquity.apikey=a90a26bab9868d071c1e4325a9623df284efd384
	mobiquity.secret=e6c9d0f81fc25d4f11950153d37b44e39d0ba6b9
	mobiquity.cache.eviction_time=3
	mobiquity.monitoring.on_period=5000
	mobiquity.monitoring.off_period=25000
	mobiquity.boot.retry_period=1800000

```

As you can see in this file you not only set which is your api key and secret, but also may set many other optional parameters that configure how _APM_ behaves.


---

## Api usage


The key principle that has driven the api design has been that the integration required the minimum code possible on the App's side, making the integration a painless process. In order to accomplish this goal, a lot of hard work is done by _APM_ under the hood. Focussing on the visible side of the api, there exists a first-class citizen in the SDK which internally forwards to other components your api requests and orchestrates their outputs, the _MobiquityManager_ facade class. This is the class you will be dealing with most of the time and next sections show off it's main responsabilities and method calls.


###Connecting to the underlying service

The very first thing to do when developing with _APM_ is to connect to the underlying running service. A suitable place to make this happen is the Activity's _onCreate_ method. Depending on your application flow and internal states you may decide to make the connection somewhere else and it's fine. Moreover, your business flow could force your App to connect from several entry points, you can accomplish that with no further effort since _APM_ is able to handle the situation transparently.


```
@Override
protected void onCreate(Bundle savedInstanceState) {
	
	...

	MobiquityManager.connect(this, new On.ServiceReady() {
			
		@Override
		public void onReady() {
				
			MobiquityManager.startRanging();
			...
		}
			
		@Override
		public void onError(State errorState) {
				
			Toast errorToast = Toast.makeText(MainActivity.this, "Cannot start proximity service.", Toast.LENGTH_LONG);
			errorToast.show();
		}
	});
}

```

###Scanning the environment

When the service has finished booting with no errors, it will be ready to accept requests and at that time the _onReady_ callback  will be called, otherwise the error callback will be called. Once the internal connection is stablished you may start ranging the environment. As you can see in the example above, this is the first action we take when the service is connected as the App is in foreground and it's fine to start _ranging_ at this Activity's lifecycle point. 

When the App goes to background you can change the scanning mode to monitoring to minimize the battery drain of the ble scanning operation. A perfect point to perform this change is _onPause_ or _onStop_ since at this time the App is about to go to background: 

```
@Override
protected void onPause(){
	
	super.onPause();		
	MobiquityManager.startMonitoring();
	...	
}

```
And the other way around when the App comes into foreground:

```
@Override
protected void onResume(){
		
	super.onResume();
	if( MobiquityManager.isConnected() )
		MobiquityManager.startRanging();
	...	
}

```

Don't need to call _stopMonitoring_ or _stopRanging_ before _startRanging_ or _startMonitoring_ because this is done internally. In addition, if you App's flow needs at some point to stop scanning the environment just call _stopMonitoring_ or _stopRanging_.

Last but not least, you can set up via the properties config file the ON and OFF cycle durations of the monitoring background operation. The default values are 5 and 15 seconds respectively. 

### Handling beacon events


One of the design principles of _APM_'s api has been the seamless integration with Android. Unlike other beacon frameworks that make your Activities implement an interface, all events will get into your Activity as _Intents_, by doing so, _APM_ moves out of the way from your class hierarchy. These are the different _Broadcast Intents_ that _APM_ will broadcast:

*  MobiquityManager.ACTION_BEACON_DISCOVERY
*  MobiquityManager.ACTION_BEACON_UPDATE
*  MobiquityManager.ACTION_BEACON_DROPPING_PACKETS
*  MobiquityManager.ACTION_BEACON_DROPPING_PACKETS2
*  MobiquityManager.ACTION_BEACON_TIMEOUT


The meaning is very straightforward, 

Below this line you have a simple example of how to create a _Broadcast Receiver_ to deal with beacon events.

```

private BroadcastReceiver beaconReceiver = new BroadcastReceiver(){

	@Override
	public void onReceive(Context context, Intent intent) {
		
		final Beacon beacon = (Beacon)intent.getSerializableExtra(MobiquityManager.EXTRAS_BEACON_KEY);
		
		if( intent.getAction().equals( MobiquityManager.ACTION_BEACON_DISCOVERY ) ){
			
			...
			
		}else if( intent.getAction().equals( MobiquityManager.ACTION_BEACON_ANNOUNCEMENT_TIMEOUT )){
			
			...
		}
	}
};

```

Some event types are delivered as _Broadcast Intent_ while others are delivered as both _Local Broadcast Intent_ and _Broadcast Intent_. Below you can check out several examples of the registration process.

* Dynamically registering and unregistering from the Local Broadcast Manager:

```
@Override
public void onResume(){
			
	super.onResume();		
	IntentFilter filterDiscovery = new IntentFilter(MobiquityManager.ACTION_BEACON_DISCOVERY);
	IntentFilter filterTimeout = new IntentFilter(MobiquityManager.ACTION_BEACON_TIMEOUT);
	IntentFilter filterUpdate = new IntentFilter(MobiquityManager.ACTION_BEACON_UPDATE);
	LocalBroadcastManager.getInstance(getActivity()).registerReceiver(beaconReceiver, filterDiscovery);
	LocalBroadcastManager.getInstance(getActivity()).registerReceiver(beaconReceiver, filterTimeout);
	LocalBroadcastManager.getInstance(getActivity()).registerReceiver(beaconReceiver, filterUpdate);
	...
}

@Override
public void onPause(){
			
	super.onPause();
	LocalBroadcastManager.getInstance(getActivity()).unregisterReceiver(beaconReceiver);
	...	
}

```
* Dynamically registering and unregistering a _Broadcast Receiver_:

```
@Override
public void onResume(){
			
	super.onResume();		
	IntentFilter filter = new IntentFilter();
	filter.addAction(MobiquityManager.ACTION_BEACON_DISCOVERY);
	filter.addAction(MobiquityManager.ACTION_BEACON_TIMEOUT);
	registerReceiver(beaconReceiver, filter);
	...
}

@Override
public void onPause(){
			
	super.onPause();
	unregisterReceiver(beaconReceiver);
	...	
}

```

* Statically registering a _Broadcast Receiver_:

```
<receiver
	android:name=".receivers.BeaconsReceiver"
	android:exported="false">
	<intent-filter>
		<action android:name="com.mobiquitynetworks.action.BEACON_DISCOVERY"/>
		<action android:name="com.mobiquitynetworks.action.BEACON_TIMEOUT"/>
	</intent-filter>
</receiver>

```


###Getting proximity content

Once you get a beacon event and unwrap the beacon object from the _Intent_, you may want to request the deployed data for this beacon by calling the asynchronous method _requestBeaconData_. Below you can find a _Broadcast Receiver_ snippet with a content request:

```
@Override
public void onReceive(Context context, Intent intent) {
	
	...
	final Beacon beacon = (Beacon)intent.getSerializableExtra(MobiquityManager.EXTRAS_BEACON_KEY);
	MobiquityManager.requestBeaconData(beacon, new On.RequestResolution<BeaconData>() {
						
		@Override
		public void onRequestSuccess(BeaconData data) 
			
			...
			mAdapter.notifyDataSetChanged();
			
		}
	
		@Override
		public void onRequestError( RequestError err ) {
			
		}
	
		@Override
		public void onRequestOngoing() {
			
		}
	});
	...

}
```
The _BeaconData_ object that is returned from the success callback exposes a set of methods that lets you get the campaigns, resources and notifications deployed on the beacon. A thing to highlight is that all callback methods are called within the main UI thread.

###Getting Beacon information

You can get a lot of intrinsic information about the emitting device from the _Beacon_ object, to point out a few:

* Name
* Mac address
* Minor, major and UUID
* Distance
* Last rssi measure
* Creation timestamp 
* Update timestamp
* Idle time
* Elapsed time
* Estimated announcement period
* Announcements counter

###Tracking


As the user walks around the venue and encounters beacons, _APM_ will automatically send tracking information (device information  and user information if available) when a beacon is first discovered and also when it becomes out of range. Besides these automatic events you can manually track other events, following you have some examples:

```
@Override
public void onListItemClick(ListView l, View v, int position, long id ){
			
	Campaign campaign = DataManager.getAllCampaigns().get(position);
	MobiquityManager.sendCampaignViewedEvent(campaign);
	...
}	
```

```
@Override
public void onListItemClick(ListView l, View v, int position, long id ){
			
	Resource resource = mCampaign.getResources().get(position);
	MobiquityManager.sendResourceViewedEvent(resource);
	...
}	
```

If you want to append the user information on every tracking request you can do so by **informing** the SDK about the user.

```		
@Override
protected void onCreate(Bundle savedInstanceState) {
	
	...

	MobiquityManager.connect(this, new On.ServiceReady() {
			
		@Override
		public void onReady() {
				
			TrackingUser trackingUser = new TrackingUser.Builder("developer")
				.provider("facebook.com")
				.username("enric@mobiquitynetworks.com")
				.audience(new TrackingAudience.Builder()
					.education(Education.GRAD_SCHOOL)
					.gender(Gender.MALE)
					.maritalStatus(MaritalStatus.SINGLE)
					.build())
				.build();						
			MobiquityManager.setTrackingUserInformation(trackingUser);
			MobiquityManager.startRanging();
			
		}
			
		...
	});
}	
```
The last piece of code appends the user information to the overall tracking info, so when beacon events show up the user structure will be sent together with the device and event structures. A serialized event might look like something like this for an exit beacon:

```
{ device: 
   { idDev: '70c1dc255de67868',
     idFA: '2de252cf-566e-4292-a9d3-532255dfa520',
     type: 'samsung GT-I9505',
     os: 'Android',
     osVersion: '4.4.2' },
  event: 
   { timestamp: '2014-07-14T15:19:02.809Z',
     type: 'EXBC',
     value: 
      { minor: 210,
        major: 1,
        uuid: '03BBAC2B-46ED-8B5A-51D5-79AB39DE6526' } },
  sdk: { version: '0.1.0' },
  user: 
   { audience: { education: 'GS', gender: 'M', maritalStatus: 'SG', kids: 0 },
     provider: 'facebook.com',
     role: 'developer',
     username: 'enric@mobiquitynetworks.com' } 
}
```

At any moment the user tracking info can be removed by calling _MobiquityManager.removeTrackingUserInformation_, from that point onwards no user info will be appended to the requests.

---

##Author

Enric Cecilla, [enric@mobiquitynetworks.com](mailto:enric@mobiquitynetworks.com)

##License

Mobiquity Technologies, Inc. All rights reserved. See the LICENSE file for more info.





