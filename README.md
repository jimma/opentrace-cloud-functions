# OpenTrace Cloud Functions

![OpenTrace Logo](OpenTrace.png)

----

OpenTrace is the open source reference implementation of BlueTrace.

BlueTrace is a privacy-preserving protocol for community-driven contact tracing across borders. It allows participating devices to log Bluetooth encounters with each other, in order to facilitate epidemiological contact tracing while protecting users’ personal data and privacy. Visit https://bluetrace.io to learn more.

The OpenTrace reference implementation comprises:
- Android app: [opentrace-community/opentrace-android](https://github.com/opentrace-community/opentrace-android)
- iOS app: [opentrace-community/opentrace-ios](https://github.com/opentrace-community/opentrace-ios)
- Cloud functions: [opentrace-community/opentrace-cloud-functions](https://github.com/opentrace-community/opentrace-cloud-functions) _(this repo)_
- Calibration: [opentrace-community/opentrace-calibration](https://github.com/opentrace-community/opentrace-calibration)

----

# Setup of Cloud Functions


## Prerequisites:
+ [Node.js](https://nodejs.org/)


## Create Firebase Project
1. Create a new Firebase Project from [Firebase console](https://console.firebase.google.com/).
2. Enable Google Analytics for the project, to be used for Firebase Crashlytics and Firebase Remote Config.


## Encryption Key
#### Generate the key
An encryption key is required to encrypt and decrypt all Temporary Identifiers (TempIDs).
The key's size depends on the algorithm use, recommended size is 256 bits (i.e., 32 bytes).
It needs to be converted to Base64 for storage in GCP Secret Manager.

A simple method to generate a random key and encode it in Base64 is:
```shell script
head -c32 /dev/urandom | base64
```

#### Store the key in Secret Manager
Create a new secret in [Secret Manager](https://console.cloud.google.com/security/secret-manager) and add a new version with the key generated above. Note that this requires Billing enabled.


#### Firebase Secret Access for Cloud Functions
The default cloud function IAM user is `<project-id>@appspot.gserviceaccount.com`, it needs to be given the **Secret Manager Secret Accessor** role in order to read data from Secret Manager.
This can be done at [IAM Admin](https://console.cloud.google.com/iam-admin/iam) page.


##Firebase Storage Buckets
Set up 2 Storage Buckets from Firebase Console:
1. **upload bucket**: allow Android/iOS apps to upload files here, block read access using the rule below.
```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /{allPaths=**} {
      allow create: if request.auth != null; // Only allow write, Cloud Functions have read/write access by default.
    }
  }
}
```

2. **archive bucket**: store processed uploaded files, block read/write access from all users using the rule below.
```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /{allPaths=**} {
      allow read, write: if false; // Disable access to all users, Cloud Functions have read/write access by default.
    }
  }
}
```


##Firebase CLI and login
Install the Firebase CLI via npm and connect to the account:
```shell script
npm install -g firebase-tools
firebase login
```


## Initialize Project
_Note: Do not use `firebase init` as it may overwrite some of the existing files._


#### `.firebaserc`
Create the file `.firebaserc` at the root directory, replacing `project-short-name` with a project name such as `dev`, `stg` or `prd`, and `project-id` with the id of the Firebase Project created above:
```json
{
  "projects": {
    "<project-short-name>": "<project-id>"
  }
}
```


#### Set the working project
Run the following to set the working project:
```shell script
firebase use <project-short-name>
```
Verify that the correct project is selected:
```shell script
firebase projects:list
```

#### Install dependencies
Run the following to install dependencies:
```shell script
npm --prefix functions install
```


#### Create project configuration file
Copy `functions/src/config.example.ts` to `functions/src/config.ts` and update all values accordingly. The most important configs are:
+ `projectId`: Project ID

+ `regions`: All regions to deploy the functions to, possible values can be found in: `functions/src/opentrace/types/FunctionConfig.ts` or at Google's [Cloud locations page](https://cloud.google.com/about/locations/).

+ `encryption.defaultAlgorithm`: The default cipher algorithm used for encrypting TempIDs, e.g., `aes-256-gcm`, `aes-256-cbc`. The full list can be found on Mac/Linux by running `openssl enc -ciphers`.

+ `encryption.keyPath`: The name of the secret created in [Encryption Key](#encryption-key) section.

+ `upload.bucket` and `upload.bucketForArchive`: The names of the buckets set up in [Firebase Storage Buckets](#firebase-storage-buckets) section.


#### Pin Generator
The class `PinGenerator` uses a plain substring to generate a pin from user uid. It should be subclassed with a secure implementation.


## Test
+ To prepare for the test, create a new service account key from the Firebase Service account. Refer to https://cloud.google.com/iam/docs/creating-managing-service-account-keys

+ Download the json credential file and set the path to `GOOGLE_APPLICATION_CREDENTIALS` environment variable. More info: https://cloud.google.com/docs/authentication/production

+ Once setup, run the test with:
```shell script
npm --prefix functions test
```


## Deploy the functions
Run the following to deploy the functions:
```shell script
firebase deploy
```
Once deployed, view the Functions in [Firebase console](https://console.firebase.google.com/) or at [GCP Cloud Functions](https://console.cloud.google.com/functions/list).

If you have set up either the [Android app](https://github.com/opentrace-community/opentrace-android) or [iOS app](https://github.com/opentrace-community/opentrace-ios), you can test the functions by opening the app, going through the registration and verify that the app displays a pin code in the Upload page.
