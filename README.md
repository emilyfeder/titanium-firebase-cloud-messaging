# Firebase Cloud Messaging - Titanium Module

Use the native Firebase SDK (iOS/Android) in Axway Titanium. This repository is part of the [Titanium Firebase](https://github.com/hansemannn/titanium-firebase) project.

## Requirements
- [x] The [Firebase Core](https://github.com/hansemannn/titanium-firebase-core) module. 
The options `googleAppID` and `GCMSenderID` are required for Android, or `file` (e.g. `GoogleService-Info.plist`) for iOS.
- [x] iOS: Titanium SDK 6.3.0+
- [x] Android: Titanium SDK 7.0.0+, [Ti.PlayServices](https://github.com/appcelerator-modules/ti.playservices) module

## Download
- [x] [Stable release](https://github.com/hansemannn/titanium-firebase-cloud-messaging/releases)
- [x] [![gitTio](http://hans-knoechel.de/shields/shield-gittio.svg)](http://gitt.io/component/firebase.cloudmessaging)

## iOS Notes

To register for push notifications on iOS, first register for notification settings, then for the push notifications
and finally for the Firebase messaging (thanks to [@garycrook](https://github.com/garycrook) for the snippet):
```js
var FirebaseCloud = require('firebase.cloudmessaging');

// Listen to the notification settings event
Ti.App.iOS.addEventListener('usernotificationsettings', function eventUserNotificationSettings() {
  // Remove the event again to prevent duplicate calls through the Firebase API
  Ti.App.iOS.removeEventListener('usernotificationsettings', eventUserNotificationSettings);
  
  // Register for push notifications
  Ti.Network.registerForPushNotifications({
    success: function () { ... },
    error: function () { ... },
    callback: function () { ... } // Fired for all kind of notifications (foreground, background & closed)
  });
  
  // Register for Firebase Cloud Messaging
	FirebaseCloud.registerForPushNotifications();
});

// Register for the notification settings event
Ti.App.iOS.registerUserNotificationSettings({
  types: [
    Ti.App.iOS.USER_NOTIFICATION_TYPE_ALERT,
    Ti.App.iOS.USER_NOTIFICATION_TYPE_SOUND,
    Ti.App.iOS.USER_NOTIFICATION_TYPE_BADGE
  ]
});
```

## Setting the Notification Icon
Same as the way [ti.goosh](https://github.com/caffeinalab/ti.goosh) does it. From their docs:

The module sets the notification tray icon taking it from `/platform/android/res/drawable-*/notificationicon.png`.

It should be flat (no gradients), white and face-on perspective.

**NB: You have to generate the icon with all resolutions.**

```
22 × 22 area in 24 × 24 (mdpi)
33 × 33 area in 36 × 36 (hdpi)
44 × 44 area in 48 × 48 (xhdpi)
66 × 66 area in 72 × 72 (xxhdpi)
88 × 88 area in 96 × 96 (xxxhdpi)
```

You can use this script to generate it **once you put** the icon in `drawable-xxxhdpi/notificationicon.png`.

```sh
#!/bin/sh

ICON_SOURCE="app/platform/android/res/drawable-xxxhdpi/notificationicon.png"
if [ -f "$ICON_SOURCE" ]; then
	mkdir -p "app/platform/android/res/drawable-xxhdpi"
	mkdir -p "app/platform/android/res/drawable-xhdpi"
	mkdir -p "app/platform/android/res/drawable-hdpi"
	mkdir -p "app/platform/android/res/drawable-mdpi"
	convert "$ICON_SOURCE" -resize 72x72 "app/platform/android/res/drawable-xxhdpi/notificationicon.png"
	convert "$ICON_SOURCE" -resize 48x48 "app/platform/android/res/drawable-xhdpi/notificationicon.png"
	convert "$ICON_SOURCE" -resize 36x36 "app/platform/android/res/drawable-hdpi/notificationicon.png"
	convert "$ICON_SOURCE" -resize 24x24 "app/platform/android/res/drawable-mdpi/notificationicon.png"
else
	echo "No notificationicon.png found"
fi
```

## API's

### `FirebaseCloudMessaging`

#### Methods

##### `registerForPushNotifications()`

##### `appDidReceiveMessage(parameters)`  - iOS only
  - `parameters` (Object)

Note: Only call this method if method swizzling is disabled (enabled by default). Messages are received via the native delegates instead,
so receive the `gcm.message_id` key from the notification payload instead.

##### `sendMessage(parameters)`
  - `parameters` (Object)
    - `messageID` (String)
    - `to` (String)
    - `timeToLive` (Number)
    - `data` (Object)

##### `subscribeToTopic(topic)`
  - `topic` (String)

##### `unsubscribeFromTopic(topic)`
  - `topic` (String)

##### `createNotificationChannel(parameters)`
https://developer.android.com/reference/android/app/NotificationChannel
  - `parameters` (Object)
    - `sound` (String) - refers to a sound file (without extension) at `platform/android/res/raw`
    either this or channelId is required. If sound == "default" or not passed in, will use the default sound. If sound == "silent" the channel will have no sound
    - `channelId` (String)
    - `channelName` (String) optional, defaults to `channelId`
    - `importance` (String) optional

#### Properties

##### `fcmToken` (String, get)

##### `apnsToken` (String, set) - iOS only

##### `shouldEstablishDirectChannel` (Number, get/set)

#### Events

##### `didReceiveMessage`
  - `message` (Object)

iOS Note: This method is only called on iOS 10+ and only for direct messages sent by Firebase. Normal Firebase push notifications
are still delivered via the Titanium notification events, e.g. 
```js
Ti.App.iOS.addEventListener('notification', function(event) {
  // Handle foreground notification
});

Ti.App.iOS.addEventListener('remotenotificationaction', function(event) {
  // Handle background notification action click
});
```

##### `didRefreshRegistrationToken`
  - `fcmToken` (String)

## Example
```js
var core = require('firebase.core');
var fcm = require('firebase.cloudmessaging');

// Configure core module (required for all Firebase modules).
core.configure({
    GCMSenderID: '...',
    googleAppID: '...', // Differs between Android and iOS.
    // file: 'GoogleService-Info.plist' // If using a plist (iOS only).
});

// Called when the Firebase token is registered or refreshed.
fcm.addEventListener('didRefreshRegistrationToken', function(e) {
    Ti.API.info('Token', e.fcmToken);
});

// Called when direct messages arrive. Note that these are different from push notifications.
fcm.addEventListener('didReceiveMessage', function(e) {
    Ti.API.info('Message', e.message);
});

//For configuring custom sounds and importance for the generated system notifications when app is in the background
fcm.createNotificationChannel({
    sound: 'warm_sound',
    channelId: 'general',
    channelName: 'General Notifications',
    importance: 'high' //will pop in from the top and make a sound
})

// Register the device with the FCM service.
fcm.registerForPushNotifications();

// Check if token is already available.
if (fcm.fcmToken) {
    Ti.API.info('FCM-Token', fcm.fcmToken);
} else {
    Ti.API.info('Token is empty. Waiting for the token callback ...');
}

// Subscribe to a topic.
fcm.subscribeToTopic('testTopic');
```

## Send FCM messages with PHP
To test your app you can use this PHP script to send messages to the device/topic:

```php
<?php
    $url = 'https://fcm.googleapis.com/fcm/send';

    $fields = [
        'to' => '/topics/testTopic', // or device token
        'notification' => [
            'title' => 'TiFirebaseMessaging',
            'body' => 'Message received'
        ],
        'data' => [
            'key1' => 'value1',
            'key2' => 'value2'
        ]
    ];

    $headers = [
        'Authorization: key=SERVER_ID_FROM_FIREBASE_SETTIGNS_CLOUD_MESSAGING', 'Content-Type: application/json'
    ];
    $ch = curl_init();

    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($fields));

    $result = curl_exec($ch);

    echo $result;
    curl_close($ch);
?>
```

Run it locally with `php filelane.php` or put it on a webserver where you can execute PHP files.

## Build

### iOS

```js
cd ios
appc ti build -p ios --build-only
```

### Android

```js
cd android
appc ti build -p android --build-only
```

## Legal

(c) 2017-Present by Hans Knöchel & Michael Gangolf
