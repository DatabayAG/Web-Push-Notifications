## Implementation of asynchronous push-notifications in progressive web-apps

Using [Push API](https://www.w3.org/TR/push-api/)
and [Firebase](https://firebase.google.com/docs)

Works for Chrome, EDGE and Firefox. For the following examples Google Chrome in version 76 was used.

### Requirements


1. The web-app server communication expects an HTTPS connection
2. All web-certificates must be valid
   * otherwise the browser has to be started explicitly unsecured like
    
    `google-chrome --user-data-dir=/tmp/foo --ignore-certificate-errors --unsafely-treat-insecure-origin-as-secure=<your-host>`

3. Ensure browser-notifications are not globally disabled

### Implementation

#### Create a firebase project

To create a firebase account, you need a google account first. 
With that you can log in on [console.firebase.google.com](https://console.firebase.google.com).
Now enter an existing firebase project or create a new project.
After entering the project, click on the gear right next to "Project Overview" on the top of the left sidebar and enter "Project settings".
You should now see some general settings for your project and below a list of all your apps.
Choose a web-app out of the list of your apps (those which are prefixed with </>) or create a new web-app by clicking the "Add app" button.
After selecting an app you see multiple information on the right.
Copy the CDN html-code, which is shown directly under "Firebase SDK snippet". It should look similar to this:

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
    
If you do not want to use CDN you can also download the js.

#### Integration

To integrate push notifications in your web-app you have to paste the SDK snippet into your index.html and add a few things.
First of all you have to register a service worker.
To do that you have to create a `firebase-messaging-sw.js` in your root directory.
In this javascript you can define all event listeners your push-notifications should react to.
After this you need the `firebase-messaging.js`.
You should add it right after the `firebase-app.js` in your SDK snippet

     <script src="https://www.gstatic.com/firebasejs/6.5.0/firebase-messaging.js"></script>
     
Now you should be able to create a messaging object below the `initializeApp` in the SDK snippet.

    const messaging = firebase.messaging();

You have to connect the service worker to your firebase project server.

In order to do that you need your public VAPID key.
- VAPID is a secure protocol communication between the server and the service worker.
- It is published by Google and a proposed standard for [IETF](https://www.rfc-editor.org/info/rfc8292).

To get the key, go to your firebase project setting and enter the tab "Cloud Messaging".
You have to scroll down to "Web configuration".
In box on the right (below "Web Push certificates") you should see a key called "Key pair".
If not, create one.
Then copy the key and inject it into your SDK snippet right below the creation of the messaging object.

    messaging.usePublicVapidKey('<your-key>');

For a public web app its necessary that you ask the user for permission to send notifications.
Therefore, you have to add the function `messaging.requestPermission()` and wrap the connection process.

    messaging.requestPermission();
    if(Notification.permission === 'granted'){
        messaging.usePublicVapidKey('<your-key>');
    } else {
        console.error('Notifications disabled!');
    }

#### Sending

To send a message to all registered service workers you need to get the token from one of the service workers.
For this you have to call the promise-function `messaging.getToken()` after your integration.

    messaging.getToken().then((currentToken) => {});

The promises response is the token.
When you combine the token with the default firebase cloud messaging send url (https://fcm.googleapis.com/fcm/send/) you get your personal endpoint for your push notifications.

    messaging.getToken().then((currentToken) => {
        var endpoint = 'https://fcm.googleapis.com/fcm/send/' + currentToken;
    }).catch((err) => {
        console.error('Subscription failed!');
    });

You can now send notification requests to that endpoint.
To authenticate the request you need one of your firebase server keys.
You can find those in your firebase project settings in the tab cloud messaging.
You can use the Legacy Server key, but its recommended to add/use a new Server key.
It is possible to create multiple Server keys for different purposes.

With the endpoint and the Server key you are now able to trigger push notifications on all active service workers.
Therefore, you can use any valid communication method.
In the following example, I use CURL on my CLI.

     curl "<your-endpoint>" --request POST --header "TTL: 60" --header "Content-Length: 0" --header "Authorization: key=<your-server-key>"

This may be a little bit different between different browsers.
This one is for Chrome. For Firefox, for example, you do not have to send the Authorization header.
With this the communication of the server and service worker is finished.

To react to an incoming request you have to define an event listener in your `firebase-messaging-sw.js`.
This works just like normal event listeners.
The following example listen to a push request and displays a notification:

    self.addEventListener('push', function(event) {
        self.registration.showNotification('Push Notification', {})
    });

The second parameter of the `showNotification` is an option array. With That you can change the notification behavior (just like mobile vibration, etc. all options can be tested [here](https://tests.peter.sh/notification-generator/)).
To update the registered service worker just refresh your Browser.

With this you should always see a visual push notification as long as you have a browser with an active service worker running.


#### Common problems

###### Insufficient permission

*  In the most cases, if you can not get a token for your endpoint this is caused by insufficient notification permissions.
   * You can prove your permission with Notification.permission (should be "granted").
   * You can reset your permission with right click on the prefix in your browsers URL bar or in your browser settings.

###### Wrong push request authentication
If your (for Example CURL) request is responded with something like "bad request" or "wrong authentication" in most cases your endpoint-token has changed.
This can happen if you updated your service worker but did not replace it with a new one.
In this case just update your endpoint-token in your request.

###### Missing compatibility
For some Browser there are minor differences in the implementation or in the accepted request.
Push notifications do not work in Safari or IE.

#### Alternatives

There are some possible variations in the implementation progress

* There are ways to implement notifications without the firebase API or even without firebase.
   * An example using the web-push node.js library can be found [here](https://developers.google.com/web/ilt/pwa/introduction-to-push-notifications#vapid).
* Its possible to send push notifications without VAPID, but it is not recommended.
* The Sending progress can be done with the firebase API, too. But i personally would recommend it to keep layers of incoming request abstract.








