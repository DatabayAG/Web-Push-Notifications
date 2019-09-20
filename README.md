## Implementation of Asynchronous Push Notifications in Progressive Web Applications

Using [Push API](https://www.w3.org/TR/push-api/)
and [Firebase](https://firebase.google.com/docs).

The `Push API` is currently [supported](https://caniuse.com/#feat=push-api)
by Chrome, Edge and Firefox. Chrome v. 76 was used for the examples and prototypes described in this document.

### Requirements

1. The web application server servers it's documents via a HTTP connection.
2. All certificates used for transport encryption (HTTPS/SSL) must be valid.
    * Otherwise (e.g. for development purposes) the browser has to be started explicitly unsecured like:

```bash
    google-chrome --user-data-dir=/tmp/foo --ignore-certificate-errors --unsafely-treat-insecure-origin-as-secure=<your-host>
```

3. Ensure browser notifications are not globally disabled in your browser settings.

### Implementation

#### Create a Firebase Project

To create a firebase account, you need a google account first. 
With that you can log in on [console.firebase.google.com](https://console.firebase.google.com).
Now enter an existing firebase project by selecting the respective
project on your dashboard or create a new project via
the `Create a project` button.
After entering the project, click on the gear symbol right next to `Project Overview`
at the top of the left sidebar and click on the `Project settings` link.
You should now see some general settings for your project and a `Your apps` section
at the bottom of the page.

* If you have not created an app at all: Click on the a web application
symbol `</>` in the `Your apps` list at the bottom of the page to create a new web application.
* If you have already created at least one app: Click on the `Add app` button to create an further app,
or choose the respective app from the `Web apps` list.
 
After selecting an app you see multiple information on the right.
Copy the CDN HTML code, which is shown directly below `Firebase SDK snippet`.
It should look similar to this:

```html
    <!-- The core Firebase JS SDK is always required and must be listed first -->
    <script src="https://www.gstatic.com/firebasejs/6.5.0/firebase-app.js"></script>
    
    <!-- TODO: Add SDKs for Firebase products that you want to use
         https://firebase.google.com/docs/web/setup#config-web-app -->
    
    <script>
      // Your web app's Firebase configuration
      var firebaseConfig = {
        apiKey: "AIzaSyBqyAcC_NCX3hmqah3WEq8oCTRttHYxYy8",
        authDomain: "push-notifications-d6997.firebaseapp.com",
        databaseURL: "https://push-notifications-d6997.firebaseio.com",
        projectId: "push-notifications-d6997",
        storageBucket: "",
        messagingSenderId: "112067727973",
        appId: "1:112067727973:web:d82edfa3dc7a06bb"
      };
      // Initialize Firebase
      firebase.initializeApp(firebaseConfig);
    </script>
```
    
If you do not want to use CDN you can also download the JavaScript file.

#### Integration

To enable your web application to handle push notifications you have
to integrate the SDK snippet into your frontend HTML documents and
add a few more things.

First of all you have to register a [`Service Worker`](https://caniuse.com/#feat=serviceworkers).
If you do not have any `Service Worker` yet, you can add the `firebase-messaging-sw.js`
file from the ([`Firebase SDK`] (you can find useful links for integration [here](https://firebase.google.com/docs/cloud-messaging/js/receive) or
[here](https://firebase.google.com/docs/cloud-messaging/js/receive)).
to your application's root directory.
In ILIAS the integration or implementation of a `Service Worker` will have
strong correlations to the **Concept ILIAS Offline**, because each web application
can only register one `Service Worker`.
If your application already implements a `Service Worker` it has to be open
for extended push notification JavaScript code by offering respective slots
for other components. 
In this `firebase-messaging-sw.js` file you can define all event listeners
your push notifications should react to. After this you need
the `firebase-messaging.js` to be included in your HTML document.

You should include it right after the `firebase-app.js` in your SDK snippet:

```html
    <script src="https://www.gstatic.com/firebasejs/6.5.0/firebase-messaging.js"></script>
```

Now you should be able to create a messaging object below the `initializeApp` in the SDK snippet.

```js
    const messaging = firebase.messaging();
```

You have to connect the service worker to your firebase project server.

In order to do that you need your public [`VAPID`](https://www.rfc-editor.org/info/rfc8292) key.
- VAPID is a secure protocol communication between the server and the service worker.
- It is published by Google and a proposed standard for [IETF](https://www.rfc-editor.org/info/rfc8292).

To get the key, go to your firebase project setting and enter the tab `Cloud Messaging`.
You have to scroll down to `Web configuration`.
In the box on the right (below `Web Push certificates`) you should see a key called `Key pair`.
If not, create one.
Then copy the key and inject it into your SDK snippet right below the creation of the messaging object.

```js
    messaging.usePublicVapidKey('<your-key>');
```

For a public web app it is necessary that you ask the user for permission to send notifications.
Therefore, you have to add the function `messaging.requestPermission()` and wrap the connection process.

```js
    messaging.requestPermission();
    if(Notification.permission === 'granted'){
        messaging.usePublicVapidKey('<your-key>');
    } else {
        console.error('Notifications disabled!');
    }
```

#### Sending

To send a message to all registered service workers you need to get the token
from one of the service workers.
For this you have to call the promise-function `messaging.getToken()` after your integration.

```js
    messaging.getToken().then((currentToken) => {});
```

The response of the promise passed as argument is the actual token.
When you combine the token with the default firebase cloud messaging
send url (https://fcm.googleapis.com/fcm/send/) you get your personal
endpoint for your push notifications.

```js
    messaging.getToken().then((currentToken) => {
        var endpoint = 'https://fcm.googleapis.com/fcm/send/' + currentToken;
    }).catch((err) => {
        console.error('Subscription failed!');
    });
```

You can now send notification requests to that endpoint.
To authenticate the request you need one of your firebase server keys.
You can find those in your firebase project settings in the tab `Cloud Messaging`.
You can use the `Legacy Server key`, but its recommended to add/use a new `Server key`.
It is possible to create multiple `Server keys` for different purposes.

With the endpoint and the `Server key` you are now able to trigger push notifications
on all active service workers.
Therefore, you can use any valid communication method.
In the following example, CURL is used on a command line interface.

```bash
     curl "<your-endpoint>" --request POST --header "TTL: 60" --header "Content-Length: 0" --header "Authorization: key=<your-server-key>"
```

This may be a little bit different between different browsers.
This one is for Chrome. For Firefox, for example, you do not have to send the `Authorization header`.
With this the communication of the server and service worker is finished.

To react to an incoming request you have to define an event listener in your `firebase-messaging-sw.js`.
This works just like regular event listeners.
The following example listens to a push request and displays a notification:

```js
    self.addEventListener('push', function(event) {
        self.registration.showNotification('Push Notification', {})
    });
```

The second parameter of the `showNotification` is an option array.
With that you can change the notification behavior (just like mobile vibration, etc.
all options can be tested [here](https://tests.peter.sh/notification-generator/)).
To update the registered service worker just refresh your browser.

With this you should always see a visual push notification as long as you
have a browser with an active service worker running.

#### Common Problems

###### Insufficient Permission

*  In most cases, if you cannot get a token for your endpoint, this is caused by insufficient notification permissions.
   * You can check the permission with `Notification.permission` (should be `'granted'`).
   * You can reset the permission with a right click on the prefix in your browsers URL bar or in your browser settings.

###### Wrong Push Request Authentication

If your (for Example CURL) request is responded with something like "bad request" or "wrong authentication" in most cases your endpoint-token has changed.
This can happen if you updated your service worker but did not replace it with a new one.
In this case just update your endpoint-token in your request.

###### Missing Compatibility

For some browsers there are minor differences in the implementation or in the accepted request.
Push notifications do not work in Safari or IE.

#### Alternatives

There are some possible variations in the implementation progress:

* There are ways to implement notifications without the `Firebase API` or even without `Firebase`.
   * An example using the web-push node.js library can be found [here](https://developers.google.com/web/ilt/pwa/introduction-to-push-notifications#vapid).
* It's possible to send push notifications without VAPID, but it is not recommended.
* The sending progress can be done with the `Firebase API`, too. But it is not recommended because the layers of incoming request should be kept abstract.