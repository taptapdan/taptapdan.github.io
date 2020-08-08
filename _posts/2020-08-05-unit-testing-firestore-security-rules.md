---
layout: post
title: Unit Testing Firestore Security Rules
---

Notes taken from Firebase Live 2020 presentation.

- [Watch Video](https://www.youtube.com/watch?v=8CR86yFmY2I)
- [Firebase Live 2020](https://firebaseonair.withgoogle.com/events/firebase-live20)
- [Firebase Blog](https://firebase.googleblog.com/)

## Introduction

- Develop locally against emulators first before touching instances in the cloud.
    - Speed
    - Safety
    - Cost
- [VSCode Firebase Extension](https://marketplace.vscode.pro/vsfire) with Firestore Security Rules syntax highlighting
- `npm -i g firebase-tools` Install latest Firebase CLI tools.
- Security rules aren't a filter / a way to query.
- Reads are done on a document basis, can't return fields within a doc.

### Examples

```
rules_version = 2;

service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if false;
    }
    
    // Read-Only Example
    match /readonly/{docId} {
      allow write: if false;
      allow read: if true;
    }
    
    // User-Specific Example
    match /users/{userId} {
      allow write: if (request.auth.uid == userId);
    }
    
    // Post Author Example
    match /posts/{postId} {
      allow read: if (resource.data.visibility == 'public') ||
        (resource.data.authorId == request.auth.uid)
    }
  }
}
```

```js
const asserts = require('assert');
const firebase = require('@firebase/testing');

const MY_PROJECT_ID = 'my-social-app-d12345';
const myUid = 'user_abc';
const theirUid = 'user_xyz';
const myAuth = { uid: myUid, email: 'abc@example.org' };

function getFirestore(auth = null) {
  return firebase.initializeTestApp({
    projectId: MY_PROJECT_ID,
    // We can simulate the user authorization, which would come though on request.
    auth: auth,
  }).firestore();
}

function getAdminFirestore() {
  return firebase.initializeAdminApp({ projectId: MY_PROJECT_ID });
}

describe('Firestore Rules', () => {
  beforeEach(async () => {
    await firebase.clearFirestoreData({ projectId: MY_PROJECT_ID });
  });

  after(async () => {
    await firebase.clearFirestoreData({ projectId: MY_PROJECT_ID });
  });

  // Read-Only Example
  it("Can read docs in the read-only collection", async () => {
    const db = getFirestore();
    const testDoc = db.collection('readonly').doc('testDoc');
    await firebase.assertSucceeds(testDoc.get());
  });

  it("Can't write to docs in the read-only collection", async () => {
    const db = getFirestore();
    const testDoc = db.collection('readonly').doc('anotherTestDoc');
    await firebase.assertFails(testDoc.set({ foo: 'bar' }));
  });
  
  // User-Specific Example
  it("Can write to a user document with the same ID as our user", async () => {
    const db = getFirestore(myAuth);
    const testDoc = db.collection('users').doc(myId);
    await firebase.assertSucceeds(testDoc.set({ foo: 'bar' }));
  });
  
  it("Can't write to a user document with different ID as our user", async () => {
    const db = getFirestore(myAuth);
    const testDoc = db.collection('users').doc(theirId);
    await firebase.assertFails(testDoc.set({ foo: 'bar' }));
  });
  
  // Post Author Example
  it("Can read posts marked public", async () => {
    const db = getFirestore();
    const testQuery = db.collection('posts').where('visibility', '==', 'public');
    await firebase.assertSucceeds(testQuery.get());
  });
  
  it("Can query personal posts", async () => {
    const db = getFirestore(myAuth);
    const testQuery = db.collection('posts').where('authorId', '==', myId);
    await firebase.assertSucceeds(testQuery.get());
  });
  
  it("Can't query all posts", async () => {
    // If there's a possibility that the document data could cause this to fail
    // then the security rules don't allow you to perform this query. The determination
    // is made without looking at all of the data.
    const db = getFirestore(myAuth);
    const testQuery = db.collection('posts');
    await firebase.assertFails(testQuery.get());
  });
  
  it("Can read a single public post", async () => {
    const postId = 'public_post';
  
    const admin = getAdminFirestore();
    const setupDoc = admin.collection('posts').doc(postId);
    await setupDoc.set({ authorId: theirId, visibility: 'public' });
  
    const db = getFirestore();
    const testRead = db.collection('posts').doc(postId);
    await firebase.assertSucceeds(testQuery.get());
  });
  
  it("Can read private post belonging to self", async () => {
    const postId = 'public_post';
  
    const admin = getAdminFirestore();
    const setupDoc = admin.collection('posts').doc(postId);
    await setupDoc.set({ authorId: myId, visibility: 'private' });
    
    const db = getFirestore();
    const testRead = db.collection('posts').doc(postId);
    await firebase.assertSucceeds(testQuery.get());
  });
  
  
  it("Can't read private post belonging to other user", async () => {
    const postId = 'public_post';
  
    const admin = getAdminFirestore();
    const setupDoc = admin.collection('posts').doc(postId);
    await setupDoc.set({ authorId: theirId, visibility: 'private' });
    
    const db = getFirestore();
    const testRead = db.collection('posts').doc(postId);
    await firebase.assertFails(testQuery.get());
  });
});
```
