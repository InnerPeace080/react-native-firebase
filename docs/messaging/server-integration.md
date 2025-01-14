---
title: Server Integration
description: Integrate Firebase Cloud Messaging with your backend server.
next: /storage/usage
previous: /messaging/notifications
---

The Cloud Messaging module provides the tools required to enable you to send custom messages directly from your own servers.
For example, you could send a FCM message to a specific device when a new chat message is saved to your database
and display a [notification](/messaging/notifications) or update local device storage so the message is instantly available.

Firebase provides a number of SDKs in different languages such as [Node.JS](https://www.npmjs.com/package/firebase-admin),
[Java](https://firebase.google.com/docs/reference/admin/java/reference/com/google/firebase/messaging/package-summary),
[Python](https://firebase.google.com/docs/reference/admin/python/firebase_admin.messaging),
[C#](https://firebase.google.com/docs/reference/admin/dotnet/namespace/firebase-admin/messaging) and
[Go](https://godoc.org/firebase.google.com/go/messaging). It also supports sending messages over
[HTTP](https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages). These methods allow you to send messages
directly to your user's devices via the FCM servers.

# Device tokens

To send a message to a device, you must access its unique token. A token is automatically generated by the device and
can be accessed using the Cloud Messaging module. The token should be saved inside of your systems data-store and should
be easily accessible when required.

The examples below use a [Cloud Firestore](/firestore) database to store and manage the tokens, and [Authentication](/auth)
to manage the users identity. You can however use any datastore or authentication method of your choice.

> If using iOS, ensure you have read and followed the steps in [registered with FCM](/messaging#ios---registering-devices-with-fcm) and [requested user permission](#ios---requesting-permissions) before trying to receive messages!

## Saving tokens

Once your application has started, you can call the `getToken` method on the Cloud Messaging module to get the unique
device token:

```jsx
import React, { useEffect } from 'react';
import messaging from '@react-native-firebase/messaging';
import auth from '@react-native-firebase/auth';
import firestore from '@react-native-firebase/firestore';

async function saveTokenToDatabase(token) {
  // Assume user is already signed in
  const userId = auth().currentUser.uid;

  // Add the token to the users datastore
  await firestore()
    .collection('users')
    .doc(userId)
    .update({
      tokens: firestore.FieldValue.arrayUnion(token),
    });
}

function App() {
  useEffect(() => {
    // Get the device token
    messaging()
      .getToken()
      .then((token) => {
        return saveTokenToDatabase(token);
      });

    // Listen to whether the token changes
    return messaging()
      .onTokenRefresh((token) => {
        saveTokenToDatabase(token);
      });
  }, []);
}
```

The above code snippet has a single purpose; storing the device FCM token on a remote database. The `useEffect` is fired
when the `App` component runs and immediately gets the token. It also listens to any events when the device automatically refreshes
the token.

Inside of the `saveTokenToDatabase` method, we store the token on a record specifically relating to the current user. You may also
notice that the token is being added via the `FieldValue.arrayUnion` method. A user can have more than one token (for example using 2 devices)
so it's important to ensure that we store all tokens in the database.

## Using tokens

With the tokens stored in a secure datastore, we now have the ability to send messages via FCM to those devices.

> The following example uses the Node.JS `firebase-admin` package to send messages to our devices, however any SDK (listed above)
can be used.

Go ahead and setup the [`firebase-tools`](https://www.npmjs.com/package/firebase-admin) library on your server environment.
Once setup, our script needs to perform two actions:

1. Fetch the tokens required to send the message.
2. Send a data payload to the devices that the tokens are registered to.

Imagine our application being similar to Instagram. Users are able to upload pictures, and other users can "like" those pictures.
Each time a post is liked, we want to send a message to the user that uploaded the picture. The code below simulates a function
which is called with all of the information required when a picture is liked:

```js
// Node.js
var admin = require('firebase-admin');

// ownerId - who owns the picture someone liked
// userId - id of the user who liked the picture
// picture - metadata about the picture

async function onUserPictureLiked(ownerId, userId, picture) {
  // Get the owners details
  const owner = admin
    .firestore()
    .collection('users')
    .doc(ownerId)
    .get();

  // Get the users details
  const user = admin
    .firestore()
    .collection('users')
    .doc(userId)
    .get();

  await admin.messaging().sendToDevice(
    owner.tokens, // ['token_1', 'token_2', ...]
    {
      data: {
        owner: JSON.stringify(owner),
        user: JSON.stringify(user),
        picture: JSON.stringify(picture),
      },
    }, {
      // Required for background/quit data-only messages on iOS
      contentAvailable: true,
      // Required for background/quit data-only messages on Android
      priority: 'high',
    });
}
```

Data-only messages are sent as low priority on both Android and iOS and will not trigger the `setBackgroundMessageHandler`
by default. To enable this functionality, you must set the "priority" to `high` on Android and enable the
`content-available` flag for iOS in the message payload.

> If using the FCM REST API, see the [following documentation](https://firebase.google.com/docs/cloud-messaging/http-server-ref) on setting `priority` and `content-available`!

The `data` property can send an object of key-value pairs totaling 4KB as string values (hence the `JSON.stringify`).

Back within our application, as explained in the [Usage](/messaging) documentation, our message handlers will receive a
[`RemoteMessage`](/reference/messaging/remotemessage) payload containing the message details sent from the server:

```jsx
function App() {
  useEffect(() => {
    const unsubscribe = messaging().onMessage(async remoteMessage => {
      const owner = JSON.parse(remoteMessage.data.owner);
      const user = JSON.parse(remoteMessage.data.user);
      const picture = JSON.parse(remoteMessage.data.picture);

      console.log(`The user "${user.name}" liked your picture "${picture.name}"`);
    });

    return unsubscribe;
  }, []);
}
```

Your application code can then handle messages as you see fit; updating local cache, displaying a [notification](/messaging/notifications)
or updating UI. The possibilities are endless!

# Send messages to topics

When devices [subscribe to topics](/messaging/usage#topics), you can send messages without specifying/storing any device
tokens. 

Using the `firebase-admin` Admin SDK as an example, we can send a message to devices subscribed to a topic:

```js
const admin = require('firebase-admin');

const message = {
  data: {
    type: 'warning',
    content: 'A new weather warning has been created!',
  },
  topic: 'weather',
};

admin.messaging().send(message)
  .then((response) => {
    console.log('Successfully sent message:', response);
  })
  .catch((error) => {
    console.log('Error sending message:', error);
  });
```

## Conditional topics

To send a message to a combination of topics, specify a condition, which is a boolean expression that specifies the target
topics. For example, the following condition will send messages to devices that are subscribed to `weather` and either `news`
or `traffic`:

```
'weather' in topics && ('news' in topics || 'traffic' in topics)
```

To send a message to this condition, replace the `topic` key with `condition`:

```js
const admin = require('firebase-admin');

const message = {
  data: {
    content: 'New updates are available!',
  },
  condition: "'weather' in topics && ('news' in topics || 'traffic' in topics)",
};

admin.messaging().send(message)
  .then((response) => {
    console.log('Successfully sent message:', response);
  })
  .catch((error) => {
    console.log('Error sending message:', error);
  });
```
