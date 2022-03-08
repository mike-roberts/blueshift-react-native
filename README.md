# blueshift-react-native

React native plugin for Blueshift iOS and Android SDK.

## Installation

```sh
npm install blueshift-react-native
```

## Usage

```js
import { multiply } from "blueshift-react-native";

// ...

const result = await multiply(3, 7);
```


## Deep links 
Blueshift Plugin will deliver Push, in-app and universal links deep links to react native using `url` event. You can add event listener as below in you react project to get the deep link.

```javascript
// Add event listener to get the deep link url
Linking.addEventListener('url', this.handleDeepLink);

handleDeepLink(event) {
  // handle the deep link
  console.log("deep link url = "+ event.url);
}
```

# iOS integration

### Prerequisites

Following permissions needs to be enabled in your Xcode project to send push notifications to the user’s device. 
* To send push notifications, [add **Push Notifications** capability to your app target](https://developer.apple.com/documentation/usernotifications/registering_your_app_with_apns). 
* To send silent push notifications, [add **Background modes** capability to your App target and enable **Remote notifications** background mode for your app target](https://developer.apple.com/documentation/usernotifications/setting_up_a_remote_notification_server/pushing_background_updates_to_your_app).

After adding the **Push Notifications** capability and enabling **Remote notifications** background mode, it should look like below.

![alt text](https://files.readme.io/6b23055-capability.png)

## 1. SDK integration

You can integrate Blueshift React plugin for your iOS project using Automatic integration, where Blueshift Plugin can take care of handling the device token, push notification and deep link callbacks. 

Automatic integration is not recommended if you are using Firebase SDK or any other SDK with auto integration along with Blueshift. The push notification and other OS callback methods do conflict with each other, so in this case you can use the manual way of SDK integration. Please reach out to our support team by sending an email to support@getblueshift.com for any integration related queries.

Follow below steps for SDK integration:

#### Setup AppDelegate.h 

To get started, include the Plugin’s header file in the `AppDelegate.h` file of the app’s Xcode project.

Include the Plugin’s header `BlueshiftPluginManager.h` in `AppDelegate.h` and also implement the `UNUserNotificationCenterDelegate` protocol on `AppDelegate` class. The `AppDelegate.h` should like:

```objective-c
#import <React/RCTBridgeDelegate.h>
#import <UIKit/UIKit.h>

#import "BlueshiftPluginManager.h"

@interface AppDelegate : UIResponder <UIApplicationDelegate, RCTBridgeDelegate, UNUserNotificationCenterDelegate>

@property (nonatomic, strong) UIWindow *window;

@end
```

#### Setup AppDelegate.m 

Now open `AppDelegate.m` file and add the following function in the `AppDelegate` class. Initialise the Blueshift react native plugin using `BlueshiftPluginManager` class method `intialisePluginWithConfig: autoIntegrate:`. Pass `autoIntegrate` as `YES` to opt in for automatic integration.

To know more about the other optional configurations, please check [this document](https://developer.blueshift.com/docs/include-configure-initialize-the-ios-sdk-in-the-app#initialize-the-sdk).


```objective-c

- (void)initialiseBlueshiftWithLaunchOptions:(NSDictionary*)launchOptions {
  // Create config object
  BlueShiftConfig *config = [[BlueShiftConfig alloc] init];
  
  // Set Blueshift API key to SDK
  config.apiKey = @"5dfe3c9aee8b375bcc616079b08156d9";
 
  // Set launch options to track the push click from killed app state
  config.applicationLaunchOptions = launchOptions;
    
  // Delay push permission by setting NO, by default push permission is displayed on app launch.
  config.enablePushNotification = NO;
  
  // Set userNotificationDelegate to self to get the push notification callbacks.
  config.userNotificationDelegate = self;
  
  // Initialise the Plugin and SDK using the Automatic integration.
  [[BlueshiftPluginManager sharedInstance] intialisePluginWithConfig:config autoIntegrate:YES];
}

```

Now call above function inside the `application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions` method of the `AppDelegate` class. The `AppDelegate.m` file will look like:

```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{

  /*
  //   React Native initialisation code.
  //
  */
  
  // Initialise the Plugin & SDK by calling the `initialiseBlueshiftWithLaunchOptions` method before the return statment. 
  [self initialiseBlueshiftWithLaunchOptions:launchOptions];
  
  return YES;
}

```

The SDK setup with automatic integration is complete over here. Using this setup you will be able to send events to Blueshift, send basic push notifications (title+content) to the iOS device. And also you will get the deep links in your react app under event `url`.
Refer section to enable Rich push notifications, section to enable in-app notifications and section to enable Blueshift email deep links. 

### Manual Integration
You will need to follow above mentioned steps to create the Blueshift Config and then initialise the Plugin by passing `autoIntegrate` as `NO`. 

```objective-c
  [[BlueshiftPluginManager sharedInstance] intialisePluginWithConfig:config autoIntegrate:NO];
```

Now, as you are doing manual integration, follow below steps to integrate the Blueshift SDK manually to handle push notification and deep link callbacks. 

#### Configure AppDelegate for push notifications
Add the following to the `AppDelegate.m` file of your app’s Xcode project to support the push notifications. Refer [this section](https://developer.blueshift.com/docs/include-configure-initialize-the-ios-sdk-in-the-app#configure-appdelegate-for-push-notifications) for more information.

```objective-c
#pragma mark - remote notification delegate methods
- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(nonnull NSData *)deviceToken {
    [[BlueShift sharedInstance].appDelegate registerForRemoteNotification:deviceToken];
}

- (void)application:(UIApplication*)application didFailToRegisterForRemoteNotificationsWithError:(NSError*)error {
    [[BlueShift sharedInstance].appDelegate failedToRegisterForRemoteNotificationWithError:error];
}

- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))handler {
    [[BlueShift sharedInstance].appDelegate handleRemoteNotification:userInfo forApplication:application fetchCompletionHandler:handler];
}

#pragma mark - UserNotificationCenter delegate methods
-(void)userNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(UNNotificationPresentationOptions options))completionHandler{
  NSDictionary* userInfo = notification.request.content.userInfo;
  if([[BlueShift sharedInstance]isBlueshiftPushNotification:userInfo]) {
    [[BlueShift sharedInstance].userNotificationDelegate handleUserNotificationCenter:center willPresentNotification:notification withCompletionHandler:completionHandler];
  } else {
    //Handle Notifications other than Blueshift
  }
}

- (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)(void))completionHandler {
  NSDictionary* userInfo = response.notification.request.content.userInfo;
  
  // Optional : Call Plugin method to send the push notification payload to react native under event `PushNotificationClickedEvent`.
  [[BlueshiftPluginManager sharedInstance] sendPushNotificationDataToRN:userInfo];
  
  if([[BlueShift sharedInstance]isBlueshiftPushNotification:userInfo]) {
  // Call Blueshift method to handle the push notification click 
    [[BlueShift sharedInstance].userNotificationDelegate handleUserNotification:center didReceiveNotificationResponse:response withCompletionHandler:completionHandler];
  } else {
    //Handle Notifications other than Blueshift
  }
}
```
#### Configure AppDelegate for Batch uploads
Add following to the `AppDelegate.m` file of your app’s Xcode project to support batch uploads. Refer [this section](https://developer.blueshift.com/docs/include-configure-initialize-the-ios-sdk-in-the-app#batch-upload-interval) for more information.
```objective-c
#pragma mark - Lifecycle methods

- (void)applicationDidEnterBackground:(UIApplication *)application {
  [[BlueShift sharedInstance].appDelegate appDidEnterBackground:application];
}

- (void)applicationDidBecomeActive:(UIApplication *)application {
  [[BlueShift sharedInstance].appDelegate appDidBecomeActive:application];
}

```

#### Handle the push and in-app deep links manually
The Blueshift iOS SDK supports deep links on push notifications and in-app messages. If a deep-link URL is present in the push or in-app message payload, the Blueshift SDK triggers `AppDelegate` class `application:openURL:options:` method on notification click/tap action and delivers the deep link there.

```objective-c
/// Override the open url method for handling deep links
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options {
 // Check if the received link is from Blueshift, then pass it to Blueshift plugin to handle it.
  if ([[BlueshiftPluginManager sharedInstance] isBlueshiftOpenURLLink:url options:options] == YES) {
    return [[BlueshiftPluginManager sharedInstance] application:application openURL:url options:options];
  } else {// If the link is not from Blueshift, write custom logic to handled it in your own way.
    return [RCTLinkingManager application:application openURL:url options:options];
  }
}
```
In case of Automatic integration, the deep links for push and in-app are handled by the plugin automatically. You can always override this functionality by manually implementing the method as mentioned above.

## 2. Enable Rich push notifications
Blueshift supports Image and Carousel based push notifications.
- Support Rich image based push notifications - you will need to add the `Notification service extension` to you project and integrate the `Blueshift-iOS-Extension-SDK`. Follow [this document](https://developer.blueshift.com/docs/integrate-your-ios-apps-notifications-with-blueshift#set-up-notification-service-extension) for step by step guide to enable Rich image push notifications.  
- Support Carousel push notifications - You will need to integrate the `Notification service extension` and then add `Notification content extension`. Follow [this document](https://developer.blueshift.com/docs/integrate-your-ios-apps-notifications-with-blueshift#set-up-notification-content-extension) for step by step guide to enable carousel push notifications. Make sure you set the App group id, `App group id` is mandatory to set when you use carousel push notifications. For [this document](https://developer.blueshift.com/docs/integrate-your-ios-apps-notifications-with-blueshift#add-an-app-group) to create and set up app group id in your project.

## 3. Enable In-App Messages
By default, In-app messages are disabled in the SDK. You will need to enable it explicitly from the Blueshift config.

#### Enable In-App messages from Blueshift Config
During the SDK initialisation in `AppDelegate.m` file, we have set the values to the config. You need to set `enableInAppNotification` property of config to `YES` to enable in-app messages from Blueshift iOS SDK. 
```objective-c 
[config setEnableInAppNotification:YES];

```
#### Enable Background Modes
We highly recommend enabling Background fetch and Remote notifications background modes from the Signing & Capabilities. This will enable the app to fetch the in-app messages if the app is in the background state.

![alt_text](https://files.readme.io/31d15b6-Screenshot_2020-07-14_at_7.02.38_PM.png)

#### Register screens for in-app messages
Once you enable the In-app messages, you will need to register the react native screens for receiving the in-app messages. You can register the screens in two ways.

1. Register all screens to receive in-app messages
You need to add `registerForInAppMessage` line in the `AppDelegate.m` file immediately after the SDK initialisation line. Refer below code snippet for reference. 
```objective-c
  [[BlueshiftPluginManager sharedInstance] intialisePluginWithConfig:config autoIntegrate:NO];
  [[BlueShift sharedInstance] registerForInAppMessage:@"ReactNative"];
```
2. Register and unregister each screen of your react native app for in-app messages. If you don’t register a screen for in-app messages, the in-app messages will stop showing up for screens that are not registered. You will need to add in-app registration and unregistration code on the `componentDidMount` and `componentWillUnmount` respectively inside your react native screens. Refer below code snipper for reference. 
```Javascript
 componentDidMount() { 
    // Register for in-app notification
    Blueshift.registerForInAppMessage("HomeScreen");
  }
  
 componentWillUnmount() {
    // Unregister for in-app notification
    Blueshift.unregisterForInAppMessage();
 }
```
## 3. Enable Blueshift email deep links
Blueshift’s deep links are usual https URLs that take users to a page in the app or launch them in a browser. If an email or text message that we send as a part of your campaign contains a Blueshift deep link and a user clicks on it, iOS will launch the installed app and Blueshift SDK will deliver the deep link to the app so that app can navigate the user to the respective screen.

- Complete the CNAME and AASA configuration as mentioned in the `Prerequisites` section of [this document](https://developer.blueshift.com/docs/integrate-blueshifts-universal-links-ios#prerequisites).

- Add associated domains to your Xcode project as mentioned in the `Integration` section of [this document](https://developer.blueshift.com/docs/integrate-blueshifts-universal-links-ios#integration).

- Follow below steps for to enable Blueshift deep links from the SDK.

Implement protocol `BlueshiftUniversalLinksDelegate` on the AppDelegate class to get the deep links callbacks from the SDK. You `AppDelegate.h` will look like below,
```objective-c
#import "BlueshiftPluginManager.h"

@interface AppDelegate : UIResponder <UIApplicationDelegate,BlueshiftUniversalLinksDelegate, UNUserNotificationCenterDelegate, RCTBridgeDelegate>

@property (nonatomic, strong) UIWindow *window;

@end
```
Now set the `blueshiftUniversalLinksDelegate` config variable to `true` to enable the Blueshift deep links during the Blueshift Plugin initialisation in `AppDelegate.m` file. 
```objective-c
  // If you want to use the Blueshift universal links, then set it as below.
  config.blueshiftUniversalLinksDelegate = self;
```

### Automatic Integration 
If you have integrated the Plugin and SDK using the automatic integration, your setup is completed here. You will receive the deep link on the react native using event `url`.

### Manual Integration
If you have opted for Manual integration, you will need to follow below steps to integrated the Blueshift Plugin.

#### Configure continueUserActivity method
Pass the URL/activity from the `continueUserActivity` method to the Plugin, so that the Plugin can process the URL and perform the click tracking. After processing the URL, the SDK sends the original URL in the `BlueshiftUniversalLinksDelegate` method.

```objective-c
// Override the `application:continueUserActivity:` method for handling the universal links
- (BOOL)application:(UIApplication *)application continueUserActivity:(nonnull NSUserActivity *)userActivity
 restorationHandler:(nonnull void (^)(NSArray<id<UIUserActivityRestoring>> * _Nullable))restorationHandler {
   // Check if the received URL is Blueshift universal link URL, then pass it to Blueshift plugin to handle it.
  if ([[BlueshiftPluginManager sharedInstance] isBlueshiftUniversalLink:userActivity] == YES) {
    return  [[BlueshiftPluginManager sharedInstance] application:application continueUserActivity:userActivity restorationHandler:restorationHandler];
  } else { // If the link is not from Blueshift, write custom logic to handled it in your own way.
    return [RCTLinkingManager application:application continueUserActivity:userActivity restorationHandler:restorationHandler];
  }
}
```
In case of Automatic integration, the email deep links are handled by the plugin automatically. You can always override this functionality by manually implementing the method as mentioned above.

#### Implement BlueshiftUniversalLinksDelegate
Now, implement the `BlueshiftUniversalLinksDelegate` delegate methods to get the success and failure callbacks. `BlueshiftPluginManager` will take care of delivering this deep link under event `url` to the react native.

```objective-c
// Deep link processing success callback
- (void)didCompleteLinkProcessing:(NSURL *)url {
  if (url) {
    [[BlueshiftPluginManager sharedInstance] application:UIApplication.sharedApplication openURL:url options:@{}];
  }
}

// Deep link processing failure callback
- (void)didFailLinkProcessingWithError:(NSError *)error url:(NSURL *)url {
  if (url) {
    [[BlueshiftPluginManager sharedInstance] application:UIApplication.sharedApplication openURL:url options:@{}];
  }
}
```

Refer to the Troubleshooting section of [this document](https://developer.blueshift.com/docs/integrate-blueshifts-universal-links-ios#troubleshooting) to 
troubleshoot the integration issues.

# Android Integration

## Depend on Blueshift Android SDK

To install the Blueshift Android SDK, add the following line to the app level `build.gradle` file. To know the latest version, check the [releases](https://github.com/blueshift-labs/Blueshift-Android-SDK/releases) page in Github. 

```groovy
implementation "com.blueshift:android-sdk-x:$sdkVersion"
```

## Depend on Firebase Messaging

Blueshift uses Firebase Messaging for sending push messages. If not already done, please integrate Firebase Messaging to the project.

If this is the first time that you are integrating FCM with your application, add the following lines of code into the `AndroidManifest.xml` file. This will enable the Blueshift Android SDK to receive the push notification sent from Blueshift servers via Firebase.

```xml
<service android:name="com.blueshift.fcm.BlueshiftMessagingService">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
```

If you have an existing FCM integration, let your `FirebaseMessagingService` class to extend `BlueshiftMessagingService` as mentioned below. This will enable the Blueshift Android SDK to receive the push notification sent from Blueshift servers via Firebase.

```java
public class AwesomeAppMessagingService extends BlueshiftMessagingService {

    @Override
    public void onMessageReceived(RemoteMessage remoteMessage) {
        if (BlueshiftUtils.isBlueshiftPushMessage(remoteMessage)) {
            super.onMessageReceived(remoteMessage);
        } else {
            /*
             * The push message does not belong to Blueshift. Please handle it here.
             */
        }
    }

    @Override
    public void onNewToken(String newToken) {
        super.onNewToken(newToken);

        /*
         * Use the new token in your app. the super.onNewToken() call is important
         * for the SDK to do the analytical part and notification rendering.
         * Make sure that it is present when you override onNewToken() method.
         */
    }
}
```

## Grant Permissions

Add the following permissions to the `AndroidManifest.xml` file.

```xml
<!-- Internet permission is required to send events, 
get notifications and in-app messages. -->
<uses-permission android:name="android.permission.INTERNET" />

<!-- Network state access permission is required to detect changes 
in network connection to schedule sync operations. -->
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

<!-- Location access permission is required if you want to track the 
location of the user. You can skip this step if you don't want to 
track the user location. -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```

## Initialize the Native SDK

Open the `MainApplication` class and add the following lines to its `onCreate()` method.

```java
Configuration configuration = new Configuration();

// == Mandatory Settings ==
configuration.setAppIcon(R.mipmap.ic_launcher);
configuration.setApiKey("BLUESHIFT_EVENT_API_KEY");

Blueshift.getInstance(this).initialize(configuration);
```

To know more about the other optional configurations, please check [this document](https://developer.blueshift.com/docs/get-started-with-the-android-sdk#optional-configurations).

Also, add update the `onCreate()` and `onNewIntent()` methods of your `MainActivity` class as given below to handle push deep links.

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
 super.onCreate(savedInstanceState);
 BlueshiftReactNativeModule.processBlueshiftPushUrl(getIntent());
}

@Override
public void onNewIntent(Intent intent) {
 super.onNewIntent(intent);
 BlueshiftReactNativeModule.processBlueshiftPushUrl(getIntent());
}
```

## Contributing

See the [contributing guide](CONTRIBUTING.md) to learn how to contribute to the repository and the development workflow.

## License

MIT
