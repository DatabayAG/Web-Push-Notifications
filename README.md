## Implementation of asyncron push-notifications in progressive web-apps

Using [Push API](https://www.w3.org/TR/push-api/)
and [Firebase](https://firebase.google.com/docs)

Works for Chrome, EDGE and Firefox. All Examples are for Chrome.

### Requirements

The web-app server communication expects a https connection

All web-certificates must be valid
 * otherwise the browser has to be started explicitly unsecured like
    
    `google-chrome --user-data-dir=/tmp/foo --ignore-certificate-errors --unsafely-treat-insecure-origin-as-secure=<your-host>`

Browser-notification arent global disabled

### Implementation

#### Create a firebase project

To create a firebase account you need a google account first. 
With that you can log in under [console.firebase.google.com](https://console.firebase.google.com).
Now enter an existing firebase project or create a new firebase project.
After entering the project click on the gear right next to "Project Overview" on the top of left sidebar and enter "Project settings".
Now you should the see some general setting for your project an below that list of all your apps.
Choose a web out of the list off your apps (those are prefixed with </>) or create a new app by clicking the "Add app" button.
After selecting an app you see al sorts of information on the right.
Copy the CDN html-code on right under "Firebase SDK snippet". It should look like this:

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
    
If you dont want to use CDN you can also download the js.

#### Integration

To integrate push notifications in your web-app you have to paste the SDK snippet into the index.html and add a few things.
First of all you have to register a ServiceWorker.
To do that you create a firebase-messaging-sw.js in your root directory.
- In this js you can define all Eventslisteners your push-notifications should react to. later more.

After this you need the firebase-messaging.js.
You can should it right after the firebase-app.js in your SDK snippet

     <script src="https://www.gstatic.com/firebasejs/6.5.0/firebase-messaging.js"></script>
     
Following you can create an messaging object below the initializeApp in the SDK snippet.

    const messaging = firebase.messaging();

Now you have to connect the ServiceWorker to your firebase project server.
For that you need your public VAPID key.
- VAPID is an secure protocol communication between the server and the ServiceWorker.
- Its published by Google and a proposed standart for [IETF](https://www.rfc-editor.org/info/rfc8292).

To get the key go to your firebase project setting and enter the tab "Cloud Messaging".
Now scroll down to "Web configuration".
In the right box (below "Web Push certificates") you should see a key called "Key pair".
If not create one.
Then copy the Key and iject it into your SDK snippet right below the creation of the messaging object inseide the function messaging.usePublicVapidKey().

    messaging.usePublicVapidKey('<your-key>');

For an public web app its necessary that you ask the user for permission to send notifications.
Therefore you have to add the function messaging.requestPermission() and wrap the connection process.

    messaging.requestPermission();
    if(Notification.permission === 'granted'){
        messaging.usePublicVapidKey('<your-key>');
    } else {
        console.error('Notifications disabled!');
    }

#### Sending

To send a message to all registered ServiceWorkers you need to get the token form one of the ServiceWorker.
For this you have to call the promise-function messaging.getToken() after your Integration.

    messaging.getToken().then((currentToken) => {});

The promises response is the token.
When you combine the token with the default firebase cloud messaging send url (https://fcm.googleapis.com/fcm/send/) you get your personal endpoint for your push notifications.

    messaging.getToken().then((currentToken) => {
        var endpoint = 'https://fcm.googleapis.com/fcm/send/' + currentToken;
    }).catch((err) => {
        console.error('Subscription failed!');
    });

You can now send notification request to that endpoint.
To authenticate the request you need one of your firebase server keys.
You can find those in your firebase project settings in the tab cloud messaging.
You can use the Legacy Server key but its recommended to add/use a new Server key.
You can create multiple Server keys for different purposes.

With the endpoint and the Server key you are now able to trigger push notifications on all active ServiceWorkers.
Therefore you can use any valid communication method.
In the following example i use CURL on my CLI.

     curl "<your-endpoint>" --request POST --header "TTL: 60" --header "Content-Length: 0" --header "Authorization: key=<your-server-key>"

The may be tiny differences for between different browsers.
This one is for Chrome. For Firefox for example you dont have to send the Authorization header.
Of course you can add meta-information to your request.
With this the communication of the server and ServiceWorker is finished.

To react on an incoming request you have to define an EventListener in your firebase-messaging-sw.js.
This works just like normal EventListeners.
The following example listen to the push request and displays a notification:

    self.addEventListener('push', function(event) {
        self.registration.showNotification('Push Notification', {})
    });

The second parameter of the showNotification is an option array with what you can change the notification behavior (just like mobile vibration etc. all options can be testet [here](https://tests.peter.sh/notification-generator/)).
To update the registered ServiceWorker just refresh your Browser.

With that you should always see a visual push notification as long as you have a browser with an active service worker running.


#### Common Problems

###### Insufficient Permission
Mostly if you cant get an token for your endpoint this is cause by insufficient notification permission.
You can prove your permission with Notification.permission (should be "granted").
You can reset your permission with rightclick on the prefix in your browsers URL bar or in your browser settings.

###### Wrong push request authentication
If you (for Example CURL) request responding with something like "bad request" or "wrong authentication" in the most cases you endpoint-token changed.
This can happen if you dont updated you service worker but replace it with a new one.
In this case just update your endpoint-token in your request.

###### Missing Compatibility
For some Browser there are minor differences in the implementation or in the accepted request.
Push notifications dont work in Safari or IE.

#### Alternatives

There are some possible variation in the implementation progress

- There are ways to implement notifications without the firebase API or even without firebase.
An example using the the web-push node.js library can be found [here](https://developers.google.com/web/ilt/pwa/introduction-to-push-notifications#vapid).
- Its possible to send push notification without VAPID, but not recommended.
- The Sending progress can be done with the firebase API, too. But i personally would recommend it to keep layer of incoming request abtract.








