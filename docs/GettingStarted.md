## Starting with Android

### Android Configuration

First, make sure you setup everthing on Firebase portal

Add this permission:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

Add google-services.json to Android project. Make sure build action is GoogleServicesJson

![ADD JSON](https://github.com/CrossGeeks/FirebasePushNotificationPlugin/blob/master/images/android-googleservices-json.png?raw=true)

Must compile against 26+ as plugin is using API 26 specific things. Here is a great breakdown: http://redth.codes/such-android-api-levels-much-confuse-wow/ (Android project must be compiled using 9.0+ target framework)

### Android Initialization

You should initialize the plugin on an Android Application class if you don't have one on your project, should create an application class. Then call **PushNotificationManager.Initialize** method on OnCreate.

There are 3 overrides to **PushNotificationManager.Initialize**:

- **PushNotificationManager.Initialize(Context context, bool resetToken,bool createNotificationChannel, bool autoRegistration)** : Default method to initialize plugin without supporting any user notification categories. Uses a DefaultPushHandler to provide the ui for the notification.

- **PushNotificationManager.Initialize(Context context, NotificationUserCategory[] categories, bool resetToken,bool createNotificationChannel, bool autoRegistration)**  : Initializes plugin using user notification categories. Uses a DefaultPushHandler to provide the ui for the notification supporting buttons based on the action_click send on the notification

- **PushNotificationManager.Initialize(Context context,IPushNotificationHandler pushHandler, bool resetToken,bool createNotificationChannel, bool autoRegistration)** : Initializes the plugin using a custom push notification handler to provide custom ui and behaviour notifications receipt and opening.


**Important: While debugging set resetToken parameter to true.**

Example of initialization:

```csharp

    [Application]
    public class MainApplication : Application
    {
        public MainApplication(IntPtr handle, JniHandleOwnership transer) :base(handle, transer)
        {
        }

        public override void OnCreate()
        {
            base.OnCreate();
	    
	    //Set the default notification channel for your app when running Android Oreo
            if (Build.VERSION.SdkInt >= Android.OS.BuildVersionCodes.O)
            {
                /*
                 * For a single Notification channel
                 */
                //Change for your default notification channel id here
                PushNotificationManager.DefaultNotificationChannelId = "DefaultChannel";

                //Change for your default notification channel name here
                PushNotificationManager.DefaultNotificationChannelName = "General";

                /*
                 * Or to work with multiple notification channels
                 * e.g. to enable multiple importance level messages or different notification sounds...etc
                 * Note: Once NotificationChannels contains at least one element, DefaultNotificationChannelId, DefaultNotificationChannelName, 
                 * and DefaultNotificationChannelImportanceLevel are ignored.
                 */
                PushNotificationManager.NotificationChannels = new List<NotificationChannelProps>()
                {
                    new NotificationChannelProps("infoMessagesId", "Informations"),
                    new NotificationChannelProps("warningMessagesId", "Warnings", NotificationImportance.High),
                    new NotificationChannelProps("reminderMessagesId", "Reminders", NotificationImportance.Min)
                };

            }

            
            //If debug you should reset the token each time.
            #if DEBUG
              PushNotificationManager.Initialize(this,true);
            #else
              PushNotificationManager.Initialize(this,false);
            #endif

              //Handle notification when app is closed here
              CrossPushNotification.Current.OnNotificationReceived += (s,p) =>
              {


              };	  
         }
    }

```

By default the plugin launches the activity where **ProcessIntent** method is called when you tap at a notification, but you can change this behaviour by setting the type of the activity you want to be launch on **PushNotificationManager.NotificationActivityType**

If you set **PushNotificationManager.NotificationActivityType** then put the following call on the **OnCreate** of activity of the type set. If not set then put it on your main launcher activity **OnCreate** method (On the Activity you got MainLauncher= true set)

```csharp
        protected override void OnCreate(Bundle bundle)
        {
            base.OnCreate(bundle);

			//Other initialization stuff

            PushNotificationManager.ProcessIntent(this,Intent);
        }

 ```

**Note: When using Xamarin Forms do it just after LoadApplication call.**

By default the plugin launches the activity when you tap at a notification with activity flags: **ActivityFlags.ClearTop | ActivityFlags.SingleTop**.

You can change this behaviour by setting **PushNotificationManager.NotificationActivityFlags**. 
 
If you set **PushNotificationManager.NotificationActivityFlags** to ActivityFlags.SingleTop  or using default plugin behaviour then make this call on **OnNewIntent** method of the same activity on the previous step.
       
 ```csharp
	    protected override void OnNewIntent(Intent intent)
        {
            base.OnNewIntent(intent);
            PushNotificationManager.ProcessIntent(this,intent);
        }
 ```

 More information on **PushNotificationManager.NotificationActivityType** and **PushNotificationManager.NotificationActivityFlags** and other android customizations here:

 [Android Customization](../docs/AndroidCustomization.md)

## Starting with iOS 

### iOS Configuration

On Info.plist enable remote notification background mode

![Remote notifications](https://github.com/CrossGeeks/FirebasePushNotificationPlugin/blob/master/images/iOS-enable-remote-notifications.png?raw=true)

### iOS Initialization

There are 3 overrides to **PushNotificationManager.Initialize**:

- **PushNotificationManager.Initialize(NSDictionary options,bool autoRegistration)** : Default method to initialize plugin without supporting any user notification categories. Auto registers for push notifications if second parameter is true.

- **PushNotificationManager.Initialize(NSDictionary options, NotificationUserCategory[] categories,bool autoRegistration)**  : Initializes plugin using user notification categories to support iOS notification actions.

- **PushNotificationManager.Initialize(NSDictionary options,IPushNotificationHandler pushHandle,bool autoRegistration)** : Initializes the plugin using a custom push notification handler to provide native feedback of notifications event on the native platform.


Call  **PushNotificationManager.Initialize** on AppDelegate FinishedLaunching
```csharp

PushNotificationManager.Initialize(options,true);

```
 **Note: When using Xamarin Forms do it just after LoadApplication call.**

Also should override these methods and make the following calls:
```csharp
        public override void RegisteredForRemoteNotifications(UIApplication application, NSData deviceToken)
        {
             PushNotificationManager.DidRegisterRemoteNotifications(deviceToken);
        }

        public override void FailedToRegisterForRemoteNotifications(UIApplication application, NSError error)
        {
            PushNotificationManager.RemoteNotificationRegistrationFailed(error);

        }
        // To receive notifications in foregroung on iOS 9 and below.
        // To receive notifications in background in any iOS version
        public override void DidReceiveRemoteNotification(UIApplication application, NSDictionary userInfo, Action<UIBackgroundFetchResult> completionHandler)
        {
            
            PushNotificationManager.DidReceiveMessage(userInfo);
        }

      
```


## Using Push Notification APIs
It is drop dead simple to gain access to the PushNotification APIs in any project. All you need to do is get a reference to the current instance of IPushNotification via `CrossPushNotification.Current`:

### On Demand Registration

When plugin initializes by default auto registers the device for push notifications. If needed you can do on demand registration by turning off auto registration when initializing the plugin.

Use the following method for on demand registration:

```csharp
   CrossPushNotification.Current.RegisterForPushNotifications();
```


### Events

Once token is registered/refreshed you will get it on **OnTokenRefresh** event.


```csharp
   /// <summary>
   /// Event triggered when token is refreshed
   /// </summary>
    event PushNotificationTokenEventHandler OnTokenRefresh;
```

```csharp        
  /// <summary>
  /// Event triggered when a notification is received
  /// </summary>
  event PushNotificationResponseEventHandler OnNotificationReceived;
```


```csharp        
  /// <summary>
  /// Event triggered when a notification is opened
  /// </summary>
  event PushNotificationResponseEventHandler OnNotificationOpened;
```

```csharp
  /// <summary>
  /// Event triggered when a notification is opened by tapping an action
  /// </summary>
  event PushNotificationResponseEventHandler OnNotificationAction;
```

```csharp 
   /// <summary>
   /// Event triggered when a notification is deleted (Android Only)
   /// </summary>
   event PushNotificationDataEventHandler OnNotificationDeleted;
```

```csharp        
  /// <summary>
  /// Event triggered when there's an error
  /// </summary>
  event PushNotificationErrorEventHandler OnNotificationError;
```

Token event usage sample:
```csharp

  CrossPushNotification.Current.OnTokenRefresh += (s,p) =>
  {
        System.Diagnostics.Debug.WriteLine($"TOKEN : {p.Token}");
  };

```

Push message received event usage sample:
```csharp

  CrossPushNotification.Current.OnNotificationReceived += (s,p) =>
  {
 
        System.Diagnostics.Debug.WriteLine("Received");
    
  };

```

Push message opened event usage sample:
```csharp
  
  CrossPushNotification.Current.OnNotificationOpened += (s,p) =>
  {
                System.Diagnostics.Debug.WriteLine("Opened");
                foreach(var data in p.Data)
                {
                    System.Diagnostics.Debug.WriteLine($"{data.Key} : {data.Value}");
                }

                if(!string.IsNullOrEmpty(p.Identifier))
                {
                    System.Diagnostics.Debug.WriteLine($"ActionId: {p.Identifier}");
                }
             
 };
```

Push message action tapped event usage sample:
**OnNotificationAction**
```csharp
  
  CrossPushNotification.Current.OnNotificationAction += (s,p) =>
  {
                System.Diagnostics.Debug.WriteLine("Action");
           
                if(!string.IsNullOrEmpty(p.Identifier))
                {
                    System.Diagnostics.Debug.WriteLine($"ActionId: {p.Identifier}");
				    foreach(var data in p.Data)
					{
						System.Diagnostics.Debug.WriteLine($"{data.Key} : {data.Value}");
					}

                }
             
 };
```

Push message deleted event usage sample: (Android Only)
```csharp

  CrossPushNotification.Current.OnNotificationDeleted+= (s,p) =>
  {
 
        System.Diagnostics.Debug.WriteLine("Deleted");
    
  };

```

Plugin by default provides some notification customization features for each platform. Check out the [Android Customization](AndroidCustomization.md) and [iOS Customization](iOSCustomization.md) sections.

<= Back to [Table of Contents](../README.md)
