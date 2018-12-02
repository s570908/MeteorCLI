# Step 12: File Upload &amp; Images

In this step, we will be using `Ionic 2` to pick up some images from our device's gallery, and we will use them to send pictures, and to set our profile picture.

## Image Picker

First, we will a `Cordova` plug-in which will give us the ability to access the gallery:

    $ meteor add cordova:cordova-plugin-image-picker@1.1.3

## Meteor FS

Up next, would be adding the ability to store some files in our data-base. This requires us to add 2 `Meteor` packages, called `ufs` and `ufs-gridfs` (Which adds support for `GridFS` operations. See [reference](https://docs.mongodb.com/manual/core/gridfs/)), which will take care of FS operations:

    $ meteor add jalik:ufs
    $ meteor add jalik:ufs-gridfs

## Client Side

Before we proceed to the server, we will add the ability to select and upload pictures in the client. All our picture-related operations will be defined in a single service called `PictureService`; The first bit of this service would be picture-selection. The `UploadFS` package already supports that feature, **but only for the browser**, therefore we will be using the `Cordova` plug-in we've just installed to select some pictures from our mobile device:

[{]: <helper> (diff_step 12.3)
#### Step 12.3: Create PictureService with utils for files

##### Added client/imports/services/picture.ts
```diff
@@ -0,0 +1,80 @@
+┊  ┊ 1┊import { Injectable } from '@angular/core';
+┊  ┊ 2┊import { Platform } from 'ionic-angular';
+┊  ┊ 3┊import { ImagePicker } from 'ionic-native';
+┊  ┊ 4┊import { UploadFS } from 'meteor/jalik:ufs';
+┊  ┊ 5┊
+┊  ┊ 6┊@Injectable()
+┊  ┊ 7┊export class PictureService {
+┊  ┊ 8┊  constructor(private platform: Platform) {
+┊  ┊ 9┊  }
+┊  ┊10┊
+┊  ┊11┊  select(): Promise<Blob> {
+┊  ┊12┊    if (!this.platform.is('cordova') || !this.platform.is('mobile')) {
+┊  ┊13┊      return new Promise((resolve, reject) => {
+┊  ┊14┊        try {
+┊  ┊15┊          UploadFS.selectFile((file: File) => {
+┊  ┊16┊            resolve(file);
+┊  ┊17┊          });
+┊  ┊18┊        }
+┊  ┊19┊        catch (e) {
+┊  ┊20┊          reject(e);
+┊  ┊21┊        }
+┊  ┊22┊      });
+┊  ┊23┊    }
+┊  ┊24┊
+┊  ┊25┊    return ImagePicker.getPictures({maximumImagesCount: 1}).then((URL: string) => {
+┊  ┊26┊      return this.convertURLtoBlob(URL);
+┊  ┊27┊    });
+┊  ┊28┊  }
+┊  ┊29┊
+┊  ┊30┊  convertURLtoBlob(URL: string): Promise<Blob> {
+┊  ┊31┊    return new Promise((resolve, reject) => {
+┊  ┊32┊      const image = document.createElement('img');
+┊  ┊33┊
+┊  ┊34┊      image.onload = () => {
+┊  ┊35┊        try {
+┊  ┊36┊          const dataURI = this.convertImageToDataURI(image);
+┊  ┊37┊          const blob = this.convertDataURIToBlob(dataURI);
+┊  ┊38┊
+┊  ┊39┊          resolve(blob);
+┊  ┊40┊        }
+┊  ┊41┊        catch (e) {
+┊  ┊42┊          reject(e);
+┊  ┊43┊        }
+┊  ┊44┊      };
+┊  ┊45┊
+┊  ┊46┊      image.src = URL;
+┊  ┊47┊    });
+┊  ┊48┊  }
+┊  ┊49┊
+┊  ┊50┊  convertImageToDataURI(image: HTMLImageElement): string {
+┊  ┊51┊    // Create an empty canvas element
+┊  ┊52┊    const canvas = document.createElement('canvas');
+┊  ┊53┊    canvas.width = image.width;
+┊  ┊54┊    canvas.height = image.height;
+┊  ┊55┊
+┊  ┊56┊    // Copy the image contents to the canvas
+┊  ┊57┊    const context = canvas.getContext('2d');
+┊  ┊58┊    context.drawImage(image, 0, 0);
+┊  ┊59┊
+┊  ┊60┊    // Get the data-URL formatted image
+┊  ┊61┊    // Firefox supports PNG and JPEG. You could check image.src to
+┊  ┊62┊    // guess the original format, but be aware the using 'image/jpg'
+┊  ┊63┊    // will re-encode the image.
+┊  ┊64┊    const dataURL = canvas.toDataURL('image/png');
+┊  ┊65┊
+┊  ┊66┊    return dataURL.replace(/^data:image\/(png|jpg);base64,/, '');
+┊  ┊67┊  }
+┊  ┊68┊
+┊  ┊69┊  convertDataURIToBlob(dataURI): Blob {
+┊  ┊70┊    const binary = atob(dataURI);
+┊  ┊71┊
+┊  ┊72┊    // Write the bytes of the string to a typed array
+┊  ┊73┊    const charCodes = Object.keys(binary)
+┊  ┊74┊      .map<number>(Number)
+┊  ┊75┊      .map<number>(binary.charCodeAt.bind(binary));
+┊  ┊76┊
+┊  ┊77┊    // Build blob with typed array
+┊  ┊78┊    return new Blob([new Uint8Array(charCodes)], {type: 'image/jpeg'});
+┊  ┊79┊  }
+┊  ┊80┊}
```
[}]: #

In order to use the service we will need to import it in the app's `NgModule` as a `provider`:

[{]: <helper> (diff_step 12.4)
#### Step 12.4: Import PictureService

##### Changed client/imports/app/app.module.ts
```diff
@@ -13,6 +13,7 @@
 ┊13┊13┊import { ProfilePage } from '../pages/profile/profile';
 ┊14┊14┊import { VerificationPage } from '../pages/verification/verification';
 ┊15┊15┊import { PhoneService } from '../services/phone';
+┊  ┊16┊import { PictureService } from '../services/picture';
 ┊16┊17┊import { MyApp } from './app.component';
 ┊17┊18┊
 ┊18┊19┊@NgModule({
```
```diff
@@ -52,7 +53,8 @@
 ┊52┊53┊  ],
 ┊53┊54┊  providers: [
 ┊54┊55┊    { provide: ErrorHandler, useClass: IonicErrorHandler },
-┊55┊  ┊    PhoneService
+┊  ┊56┊    PhoneService,
+┊  ┊57┊    PictureService
 ┊56┊58┊  ]
 ┊57┊59┊})
 ┊58┊60┊export class AppModule {}🚫↵
```
[}]: #

Since now we will be sending pictures, we will need to update the message schema to support picture typed messages:

[{]: <helper> (diff_step 12.5)
#### Step 12.5: Added picture message type

##### Changed imports/models.ts
```diff
@@ -9,7 +9,8 @@
 ┊ 9┊ 9┊
 ┊10┊10┊export enum MessageType {
 ┊11┊11┊  TEXT = <any>'text',
-┊12┊  ┊  LOCATION = <any>'location'
+┊  ┊12┊  LOCATION = <any>'location',
+┊  ┊13┊  PICTURE = <any>'picture'
 ┊13┊14┊}
 ┊14┊15┊
 ┊15┊16┊export interface Chat {
```
[}]: #

In the attachments menu, we will add a new handler for sending pictures, called `sendPicture`:

[{]: <helper> (diff_step 12.6)
#### Step 12.6: Implement sendPicture method

##### Changed client/imports/pages/messages/messages-attachments.ts
```diff
@@ -1,5 +1,6 @@
 ┊1┊1┊import { Component } from '@angular/core';
 ┊2┊2┊import { AlertController, Platform, ModalController, ViewController } from 'ionic-angular';
+┊ ┊3┊import { PictureService } from '../../services/picture';
 ┊3┊4┊import { MessageType } from '../../../../imports/models';
 ┊4┊5┊import { NewLocationMessageComponent } from './location-message';
 ┊5┊6┊import template from './messages-attachments.html';
```
```diff
@@ -12,9 +13,19 @@
 ┊12┊13┊    private alertCtrl: AlertController,
 ┊13┊14┊    private platform: Platform,
 ┊14┊15┊    private viewCtrl: ViewController,
-┊15┊  ┊    private modelCtrl: ModalController
+┊  ┊16┊    private modelCtrl: ModalController,
+┊  ┊17┊    private pictureService: PictureService
 ┊16┊18┊  ) {}
 ┊17┊19┊
+┊  ┊20┊  sendPicture(): void {
+┊  ┊21┊    this.pictureService.select().then((file: File) => {
+┊  ┊22┊      this.viewCtrl.dismiss({
+┊  ┊23┊        messageType: MessageType.PICTURE,
+┊  ┊24┊        selectedPicture: file
+┊  ┊25┊      });
+┊  ┊26┊    });
+┊  ┊27┊  }
+┊  ┊28┊
 ┊18┊29┊  sendLocation(): void {
 ┊19┊30┊    const locationModal = this.modelCtrl.create(NewLocationMessageComponent);
 ┊20┊31┊    locationModal.onDidDismiss((location) => {
```
[}]: #

And we will bind that handler to the view, so whenever we press the right button, the handler will be invoked with the selected picture:

[{]: <helper> (diff_step 12.7)
#### Step 12.7: Bind click event for sendPicture

##### Changed client/imports/pages/messages/messages-attachments.html
```diff
@@ -1,6 +1,6 @@
 ┊1┊1┊<ion-content class="messages-attachments-page-content">
 ┊2┊2┊  <ion-list class="attachments">
-┊3┊ ┊    <button ion-item class="attachment attachment-gallery">
+┊ ┊3┊    <button ion-item class="attachment attachment-gallery" (click)="sendPicture()">
 ┊4┊4┊      <ion-icon name="images" class="attachment-icon"></ion-icon>
 ┊5┊5┊      <div class="attachment-name">Gallery</div>
 ┊6┊6┊    </button>
```
[}]: #

Now we will be extending the `MessagesPage`, by adding a method which will send the picture selected in the attachments menu:

[{]: <helper> (diff_step 12.8)
#### Step 12.8: Implement the actual send of picture message

##### Changed client/imports/pages/messages/messages.ts
```diff
@@ -6,6 +6,7 @@
 ┊ 6┊ 6┊import { Observable, Subscription, Subscriber } from 'rxjs';
 ┊ 7┊ 7┊import { Messages } from '../../../../imports/collections';
 ┊ 8┊ 8┊import { Chat, Message, MessageType, Location } from '../../../../imports/models';
+┊  ┊ 9┊import { PictureService } from '../../services/picture';
 ┊ 9┊10┊import { MessagesAttachmentsComponent } from './messages-attachments';
 ┊10┊11┊import { MessagesOptionsComponent } from './messages-options';
 ┊11┊12┊import template from './messages.html';
```
```diff
@@ -29,7 +30,8 @@
 ┊29┊30┊  constructor(
 ┊30┊31┊    navParams: NavParams,
 ┊31┊32┊    private el: ElementRef,
-┊32┊  ┊    private popoverCtrl: PopoverController
+┊  ┊33┊    private popoverCtrl: PopoverController,
+┊  ┊34┊    private pictureService: PictureService
 ┊33┊35┊  ) {
 ┊34┊36┊    this.selectedChat = <Chat>navParams.get('chat');
 ┊35┊37┊    this.title = this.selectedChat.title;
```
```diff
@@ -236,12 +238,25 @@
 ┊236┊238┊          const location = params.selectedLocation;
 ┊237┊239┊          this.sendLocationMessage(location);
 ┊238┊240┊        }
+┊   ┊241┊        else if (params.messageType === MessageType.PICTURE) {
+┊   ┊242┊          const blob: Blob = params.selectedPicture;
+┊   ┊243┊          this.sendPictureMessage(blob);
+┊   ┊244┊        }
 ┊239┊245┊      }
 ┊240┊246┊    });
 ┊241┊247┊
 ┊242┊248┊    popover.present();
 ┊243┊249┊  }
 ┊244┊250┊
+┊   ┊251┊  sendPictureMessage(blob: Blob): void {
+┊   ┊252┊    this.pictureService.upload(blob).then((picture) => {
+┊   ┊253┊      MeteorObservable.call('addMessage', MessageType.PICTURE,
+┊   ┊254┊        this.selectedChat._id,
+┊   ┊255┊        picture.url
+┊   ┊256┊      ).zone().subscribe();
+┊   ┊257┊    });
+┊   ┊258┊  }
+┊   ┊259┊
 ┊245┊260┊  getLocation(locationString: string): Location {
 ┊246┊261┊    const splitted = locationString.split(',').map(Number);
```
[}]: #

For now, we will add a stub for the `upload` method in the `PictureService` and we will get back to it once we finish implementing the necessary logic in the server for storing a picture:

[{]: <helper> (diff_step 12.9)
#### Step 12.9: Create stub method for upload method

##### Changed client/imports/services/picture.ts
```diff
@@ -27,6 +27,10 @@
 ┊27┊27┊    });
 ┊28┊28┊  }
 ┊29┊29┊
+┊  ┊30┊  upload(blob: Blob): Promise<any> {
+┊  ┊31┊    return Promise.resolve();
+┊  ┊32┊  }
+┊  ┊33┊
 ┊30┊34┊  convertURLtoBlob(URL: string): Promise<Blob> {
 ┊31┊35┊    return new Promise((resolve, reject) => {
 ┊32┊36┊      const image = document.createElement('img');
```
[}]: #

## Server Side

So as we said, need to handle storage of pictures that were sent by the client. First, we will create a `Picture` model so the compiler can recognize a picture object:

[{]: <helper> (diff_step 12.1)
#### Step 12.1: Add cordova plugin for image picker

##### Changed .meteor/cordova-plugins
```diff
@@ -1,6 +1,7 @@
 ┊1┊1┊cordova-plugin-console@1.0.5
 ┊2┊2┊cordova-plugin-device@1.1.4
 ┊3┊3┊cordova-plugin-geolocation@2.4.1
+┊ ┊4┊cordova-plugin-image-picker@1.1.3
 ┊4┊5┊cordova-plugin-splashscreen@4.0.1
 ┊5┊6┊cordova-plugin-statusbar@2.2.1
 ┊6┊7┊cordova-plugin-whitelist@1.3.1
```
[}]: #

If you're familiar with `Whatsapp`, you'll know that sent pictures are compressed. That's so the data-base can store more pictures, and the traffic in the network will be faster. To compress the sent pictures, we will be using an `NPM` package called [sharp](https://www.npmjs.com/package/sharp), which is a utility library which will help us perform transformations on pictures:

    $ meteor npm install --save sharp

Now we will create a picture store which will compress pictures using `sharp` right before they are inserted into the data-base:

[{]: <helper> (diff_step 12.12)
#### Step 12.12: Create pictures store

##### Added imports/collections/pictures.ts
```diff
@@ -0,0 +1,43 @@
+┊  ┊ 1┊import { MongoObservable } from 'meteor-rxjs';
+┊  ┊ 2┊import { UploadFS } from 'meteor/jalik:ufs';
+┊  ┊ 3┊import { Meteor } from 'meteor/meteor';
+┊  ┊ 4┊import { Picture, DEFAULT_PICTURE_URL } from '../models';
+┊  ┊ 5┊
+┊  ┊ 6┊export interface PicturesCollection<T> extends MongoObservable.Collection<T> {
+┊  ┊ 7┊  getPictureUrl(selector?: Object | string): string;
+┊  ┊ 8┊}
+┊  ┊ 9┊
+┊  ┊10┊export const Pictures =
+┊  ┊11┊  new MongoObservable.Collection<Picture>('pictures') as PicturesCollection<Picture>;
+┊  ┊12┊
+┊  ┊13┊export const PicturesStore = new UploadFS.store.GridFS({
+┊  ┊14┊  collection: Pictures.collection,
+┊  ┊15┊  name: 'pictures',
+┊  ┊16┊  filter: new UploadFS.Filter({
+┊  ┊17┊    contentTypes: ['image/*']
+┊  ┊18┊  }),
+┊  ┊19┊  permissions: new UploadFS.StorePermissions({
+┊  ┊20┊    insert: picturesPermissions,
+┊  ┊21┊    update: picturesPermissions,
+┊  ┊22┊    remove: picturesPermissions
+┊  ┊23┊  }),
+┊  ┊24┊  transformWrite(from, to) {
+┊  ┊25┊    // The transformation function will only be invoked on the server. Accordingly,
+┊  ┊26┊    // the 'sharp' library is a server-only library which will cause an error to be
+┊  ┊27┊    // thrown when loaded on the global scope
+┊  ┊28┊    const Sharp = Npm.require('sharp');
+┊  ┊29┊    // Compress picture to 75% from its original quality
+┊  ┊30┊    const transform = Sharp().png({ quality: 75 });
+┊  ┊31┊    from.pipe(transform).pipe(to);
+┊  ┊32┊  }
+┊  ┊33┊});
+┊  ┊34┊
+┊  ┊35┊// Gets picture's url by a given selector
+┊  ┊36┊Pictures.getPictureUrl = function (selector) {
+┊  ┊37┊  const picture = this.findOne(selector) || {};
+┊  ┊38┊  return picture.url || DEFAULT_PICTURE_URL;
+┊  ┊39┊};
+┊  ┊40┊
+┊  ┊41┊function picturesPermissions(userId: string): boolean {
+┊  ┊42┊  return Meteor.isServer || !!userId;
+┊  ┊43┊}
```
[}]: #

You can look at a store as some sort of a wrapper for a collection, which will run different kind of a operations before it mutates it or fetches data from it. Note that we used `GridFS` because this way an uploaded file is split into multiple packets, which is more efficient for storage. We also defined a small utility function on that store which will retrieve a profile picture. If the ID was not found, it will return a link for the default picture. To make things convenient, we will also export the store from the `index` file:

[{]: <helper> (diff_step 12.13)
#### Step 12.13: Export pictures collection

##### Changed imports/collections/index.ts
```diff
@@ -1,3 +1,4 @@
 ┊1┊1┊export * from './chats';
 ┊2┊2┊export * from './messages';
-┊3┊ ┊export * from './users';🚫↵
+┊ ┊3┊export * from './pictures';
+┊ ┊4┊export * from './users';
```
[}]: #

Now that we have the pictures store, and the server knows how to handle uploaded pictures, we will implement the `upload` stub in the `PictureService`:

[{]: <helper> (diff_step 12.14)
#### Step 12.14: Implement upload method

##### Changed client/imports/services/picture.ts
```diff
@@ -2,6 +2,9 @@
 ┊ 2┊ 2┊import { Platform } from 'ionic-angular';
 ┊ 3┊ 3┊import { ImagePicker } from 'ionic-native';
 ┊ 4┊ 4┊import { UploadFS } from 'meteor/jalik:ufs';
+┊  ┊ 5┊import { _ } from 'meteor/underscore';
+┊  ┊ 6┊import { PicturesStore } from '../../../imports/collections';
+┊  ┊ 7┊import { DEFAULT_PICTURE_URL } from '../../../imports/models';
 ┊ 5┊ 8┊
 ┊ 6┊ 9┊@Injectable()
 ┊ 7┊10┊export class PictureService {
```
```diff
@@ -28,7 +31,23 @@
 ┊28┊31┊  }
 ┊29┊32┊
 ┊30┊33┊  upload(blob: Blob): Promise<any> {
-┊31┊  ┊    return Promise.resolve();
+┊  ┊34┊    return new Promise((resolve, reject) => {
+┊  ┊35┊      const metadata = _.pick(blob, 'name', 'type', 'size');
+┊  ┊36┊
+┊  ┊37┊      if (!metadata.name) {
+┊  ┊38┊        metadata.name = DEFAULT_PICTURE_URL;
+┊  ┊39┊      }
+┊  ┊40┊
+┊  ┊41┊      const upload = new UploadFS.Uploader({
+┊  ┊42┊        data: blob,
+┊  ┊43┊        file: metadata,
+┊  ┊44┊        store: PicturesStore,
+┊  ┊45┊        onComplete: resolve,
+┊  ┊46┊        onError: reject
+┊  ┊47┊      });
+┊  ┊48┊
+┊  ┊49┊      upload.start();
+┊  ┊50┊    });
 ┊32┊51┊  }
 ┊33┊52┊
 ┊34┊53┊  convertURLtoBlob(URL: string): Promise<Blob> {
```
[}]: #

## View Picture Messages

We will now add the support for picture typed messages in the `MessagesPage`, so whenever we send a picture, we will be able to see them in the messages list like any other message:

[{]: <helper> (diff_step 12.15)
#### Step 12.15: Added view for picture message

##### Changed client/imports/pages/messages/messages.html
```diff
@@ -24,6 +24,7 @@
 ┊24┊24┊              <sebm-google-map-marker [latitude]="getLocation(message.content).lat" [longitude]="getLocation(message.content).lng"></sebm-google-map-marker>
 ┊25┊25┊            </sebm-google-map>
 ┊26┊26┊          </div>
+┊  ┊27┊          <img *ngIf="message.type == 'picture'" (click)="showPicture($event)" class="message-content message-content-picture" [src]="message.content">
 ┊27┊28┊
 ┊28┊29┊          <span class="message-timestamp">{{ message.createdAt | amDateFormat: 'HH:mm' }}</span>
 ┊29┊30┊        </div>
```
[}]: #

As you can see, we also bound the picture message to the `click` event, which means that whenever we click on it, a picture viewer should be opened with the clicked picture. Let's create the component for that picture viewer:

[{]: <helper> (diff_step 12.16)
#### Step 12.16: Create show picture component

##### Added client/imports/pages/messages/show-picture.ts
```diff
@@ -0,0 +1,14 @@
+┊  ┊ 1┊import { Component } from '@angular/core';
+┊  ┊ 2┊import { NavParams, ViewController } from 'ionic-angular';
+┊  ┊ 3┊import template from './show-picture.html';
+┊  ┊ 4┊
+┊  ┊ 5┊@Component({
+┊  ┊ 6┊  template
+┊  ┊ 7┊})
+┊  ┊ 8┊export class ShowPictureComponent {
+┊  ┊ 9┊  pictureSrc: string;
+┊  ┊10┊
+┊  ┊11┊  constructor(private navParams: NavParams, private viewCtrl: ViewController) {
+┊  ┊12┊    this.pictureSrc = navParams.get('pictureSrc');
+┊  ┊13┊  }
+┊  ┊14┊}
```
[}]: #

[{]: <helper> (diff_step 12.17)
#### Step 12.17: Create show picture template

##### Added client/imports/pages/messages/show-picture.html
```diff
@@ -0,0 +1,13 @@
+┊  ┊ 1┊<ion-header>
+┊  ┊ 2┊  <ion-toolbar color="whatsapp">
+┊  ┊ 3┊    <ion-title>Show Picture</ion-title>
+┊  ┊ 4┊
+┊  ┊ 5┊    <ion-buttons left>
+┊  ┊ 6┊      <button ion-button class="dismiss-button" (click)="viewCtrl.dismiss()"><ion-icon name="close"></ion-icon></button>
+┊  ┊ 7┊    </ion-buttons>
+┊  ┊ 8┊  </ion-toolbar>
+┊  ┊ 9┊</ion-header>
+┊  ┊10┊
+┊  ┊11┊<ion-content class="show-picture">
+┊  ┊12┊  <img class="picture" [src]="pictureSrc">
+┊  ┊13┊</ion-content>
```
[}]: #

[{]: <helper> (diff_step 12.18)
#### Step 12.18: Create show pictuer component styles

##### Added client/imports/pages/messages/show-picture.scss
```diff
@@ -0,0 +1,10 @@
+┊  ┊ 1┊.show-picture {
+┊  ┊ 2┊  background-color: black;
+┊  ┊ 3┊
+┊  ┊ 4┊  .picture {
+┊  ┊ 5┊    position: absolute;
+┊  ┊ 6┊    top: 50%;
+┊  ┊ 7┊    left: 50%;
+┊  ┊ 8┊    transform: translate(-50%, -50%);
+┊  ┊ 9┊  }
+┊  ┊10┊}🚫↵
```
[}]: #

[{]: <helper> (diff_step 12.19)
#### Step 12.19: Import ShowPictureComponent

##### Changed client/imports/app/app.module.ts
```diff
@@ -10,6 +10,7 @@
 ┊10┊10┊import { MessagesAttachmentsComponent } from '../pages/messages/messages-attachments';
 ┊11┊11┊import { MessagesOptionsComponent } from '../pages/messages/messages-options';
 ┊12┊12┊import { NewLocationMessageComponent } from '../pages/messages/location-message';
+┊  ┊13┊import { ShowPictureComponent } from '../pages/messages/show-picture';
 ┊13┊14┊import { ProfilePage } from '../pages/profile/profile';
 ┊14┊15┊import { VerificationPage } from '../pages/verification/verification';
 ┊15┊16┊import { PhoneService } from '../services/phone';
```
```diff
@@ -28,7 +29,8 @@
 ┊28┊29┊    NewChatComponent,
 ┊29┊30┊    MessagesOptionsComponent,
 ┊30┊31┊    MessagesAttachmentsComponent,
-┊31┊  ┊    NewLocationMessageComponent
+┊  ┊32┊    NewLocationMessageComponent,
+┊  ┊33┊    ShowPictureComponent
 ┊32┊34┊  ],
 ┊33┊35┊  imports: [
 ┊34┊36┊    IonicModule.forRoot(MyApp),
```
```diff
@@ -49,7 +51,8 @@
 ┊49┊51┊    NewChatComponent,
 ┊50┊52┊    MessagesOptionsComponent,
 ┊51┊53┊    MessagesAttachmentsComponent,
-┊52┊  ┊    NewLocationMessageComponent
+┊  ┊54┊    NewLocationMessageComponent,
+┊  ┊55┊    ShowPictureComponent
 ┊53┊56┊  ],
 ┊54┊57┊  providers: [
 ┊55┊58┊    { provide: ErrorHandler, useClass: IonicErrorHandler },
```
[}]: #

And now that we have that component ready, we will implement the `showPicture` method in the `MessagesPage` component, which will create a new instance of the `ShowPictureComponent`:

[{]: <helper> (diff_step 12.2)
#### Step 12.2: Add server side fs packages

##### Changed .meteor/packages
```diff
@@ -25,3 +25,5 @@
 ┊25┊25┊accounts-base
 ┊26┊26┊mys:accounts-phone
 ┊27┊27┊reywood:publish-composite
+┊  ┊28┊jalik:ufs
+┊  ┊29┊jalik:ufs-gridfs
```

##### Changed .meteor/versions
```diff
@@ -36,11 +36,14 @@
 ┊36┊36┊htmljs@1.0.11
 ┊37┊37┊http@1.1.8
 ┊38┊38┊id-map@1.0.9
+┊  ┊39┊jalik:ufs@0.7.1_1
+┊  ┊40┊jalik:ufs-gridfs@0.1.4
 ┊39┊41┊jquery@1.11.10
 ┊40┊42┊launch-screen@1.1.0
 ┊41┊43┊livedata@1.0.18
 ┊42┊44┊localstorage@1.0.12
 ┊43┊45┊logging@1.1.16
+┊  ┊46┊matb33:collection-hooks@0.8.4
 ┊44┊47┊meteor@1.6.0
 ┊45┊48┊meteor-base@1.0.4
 ┊46┊49┊minifier-css@1.2.15
```
[}]: #

## Profile Picture

We have the ability to send picture messages. Now we will add the ability to change the user's profile picture using the infrastructure we've just created. To begin with, we will define a new property to our `User` model called `pictureId`, which will be used to determine the belonging profile picture of the current user:

[{]: <helper> (diff_step 12.21)
#### Step 12.21: Add pictureId property to Profile

##### Changed imports/models.ts
```diff
@@ -5,6 +5,7 @@
 ┊ 5┊ 5┊export interface Profile {
 ┊ 6┊ 6┊  name?: string;
 ┊ 7┊ 7┊  picture?: string;
+┊  ┊ 8┊  pictureId?: string;
 ┊ 8┊ 9┊}
 ┊ 9┊10┊
 ┊10┊11┊export enum MessageType {
```
[}]: #

We will bind the editing button in the profile selection page into an event handler:

[{]: <helper> (diff_step 12.22)
#### Step 12.22: Add event for changing profile picture

##### Changed client/imports/pages/profile/profile.html
```diff
@@ -11,6 +11,7 @@
 ┊11┊11┊<ion-content class="profile-page-content">
 ┊12┊12┊  <div class="profile-picture">
 ┊13┊13┊    <img *ngIf="picture" [src]="picture">
+┊  ┊14┊    <ion-icon name="create" (click)="selectProfilePicture()"></ion-icon>
 ┊14┊15┊  </div>
 ┊15┊16┊
 ┊16┊17┊  <ion-item class="profile-name">
```
[}]: #

And we will add all the missing logic in the component, so the `pictureId` will be transformed into and actual reference, and so we can have the ability to select a picture from our gallery and upload it:

[{]: <helper> (diff_step 12.23)
#### Step 12.23: Implement pick, update and set of profile image

##### Changed client/imports/pages/profile/profile.ts
```diff
@@ -1,7 +1,9 @@
 ┊1┊1┊import { Component, OnInit } from '@angular/core';
 ┊2┊2┊import { AlertController, NavController } from 'ionic-angular';
 ┊3┊3┊import { MeteorObservable } from 'meteor-rxjs';
+┊ ┊4┊import { Pictures } from '../../../../imports/collections';
 ┊4┊5┊import { Profile } from '../../../../imports/models';
+┊ ┊6┊import { PictureService } from '../../services/picture';
 ┊5┊7┊import { ChatsPage } from '../chats/chats';
 ┊6┊8┊import template from './profile.html';
 ┊7┊9┊
```
```diff
@@ -14,13 +16,37 @@
 ┊14┊16┊
 ┊15┊17┊  constructor(
 ┊16┊18┊    private alertCtrl: AlertController,
-┊17┊  ┊    private navCtrl: NavController
+┊  ┊19┊    private navCtrl: NavController,
+┊  ┊20┊    private pictureService: PictureService
 ┊18┊21┊  ) {}
 ┊19┊22┊
 ┊20┊23┊  ngOnInit(): void {
 ┊21┊24┊    this.profile = Meteor.user().profile || {
 ┊22┊25┊      name: ''
 ┊23┊26┊    };
+┊  ┊27┊
+┊  ┊28┊    MeteorObservable.subscribe('user').subscribe(() => {
+┊  ┊29┊      this.picture = Pictures.getPictureUrl(this.profile.pictureId);
+┊  ┊30┊    });
+┊  ┊31┊  }
+┊  ┊32┊
+┊  ┊33┊  selectProfilePicture(): void {
+┊  ┊34┊    this.pictureService.select().then((blob) => {
+┊  ┊35┊      this.uploadProfilePicture(blob);
+┊  ┊36┊    })
+┊  ┊37┊      .catch((e) => {
+┊  ┊38┊        this.handleError(e);
+┊  ┊39┊      });
+┊  ┊40┊  }
+┊  ┊41┊
+┊  ┊42┊  uploadProfilePicture(blob: Blob): void {
+┊  ┊43┊    this.pictureService.upload(blob).then((picture) => {
+┊  ┊44┊      this.profile.pictureId = picture._id;
+┊  ┊45┊      this.picture = picture.url;
+┊  ┊46┊    })
+┊  ┊47┊      .catch((e) => {
+┊  ┊48┊        this.handleError(e);
+┊  ┊49┊      });
 ┊24┊50┊  }
 ┊25┊51┊
 ┊26┊52┊  updateProfile(): void {
```
[}]: #

We will also define a new hook in the `Meteor.users` collection so whenever we update the profile picture, the previous one will be removed from the data-base. This way we won't have some unnecessary data in our data-base, which will save us some precious storage:

[{]: <helper> (diff_step 12.24)
#### Step 12.24: Add after hook for user modification

##### Changed imports/collections/users.ts
```diff
@@ -1,5 +1,15 @@
 ┊ 1┊ 1┊import { MongoObservable } from 'meteor-rxjs';
 ┊ 2┊ 2┊import { Meteor } from 'meteor/meteor';
 ┊ 3┊ 3┊import { User } from '../models';
+┊  ┊ 4┊import { Pictures } from './pictures';
 ┊ 4┊ 5┊
-┊ 5┊  ┊export const Users = MongoObservable.fromExisting<User>(Meteor.users);🚫↵
+┊  ┊ 6┊export const Users = MongoObservable.fromExisting<User>(Meteor.users);
+┊  ┊ 7┊
+┊  ┊ 8┊// Dispose unused profile pictures
+┊  ┊ 9┊Meteor.users.after.update(function (userId, doc, fieldNames, modifier, options) {
+┊  ┊10┊  if (!doc.profile) return;
+┊  ┊11┊  if (!this.previous.profile) return;
+┊  ┊12┊  if (doc.profile.pictureId == this.previous.profile.pictureId) return;
+┊  ┊13┊
+┊  ┊14┊  Pictures.collection.remove({ _id: doc.profile.pictureId });
+┊  ┊15┊}, { fetchPrevious: true });
```
[}]: #

Collection hooks are not part of `Meteor`'s official API and are added through a third-party package called `matb33:collection-hooks`. This requires us to install the necessary type definition:

    $ npm install --save-dev @types/meteor-collection-hooks

Now we need to import the type definition we've just installed in the `tsconfig.json` file:

[{]: <helper> (diff_step 12.26)
#### Step 12.26: Import @types/meteor-collection-hooks

##### Changed tsconfig.json
```diff
@@ -20,7 +20,8 @@
 ┊20┊20┊      "meteor-typings",
 ┊21┊21┊      "@types/underscore",
 ┊22┊22┊      "@types/meteor-accounts-phone",
-┊23┊  ┊      "@types/meteor-publish-composite"
+┊  ┊23┊      "@types/meteor-publish-composite",
+┊  ┊24┊      "@types/meteor-collection-hooks"
 ┊24┊25┊    ]
 ┊25┊26┊  },
 ┊26┊27┊  "include": [
```
[}]: #

We now add a `user` publication which should be subscribed whenever we initialize the `ProfilePage`. This subscription should fetch some data from other collections which is related to the user which is currently logged in; And to be more specific, the document associated with the `profileId` defined in the `User` model:

[{]: <helper> (diff_step 12.27)
#### Step 12.27: Add user publication

##### Changed server/publications.ts
```diff
@@ -1,6 +1,6 @@
 ┊1┊1┊import { Meteor } from 'meteor/meteor';
 ┊2┊2┊import { Mongo } from 'meteor/mongo';
-┊3┊ ┊import { Chats, Messages, Users } from '../imports/collections';
+┊ ┊3┊import { Chats, Messages, Pictures, Users } from '../imports/collections';
 ┊4┊4┊import { Chat, Message, User } from '../imports/models';
 ┊5┊5┊
 ┊6┊6┊Meteor.publishComposite('users', function(
```
```diff
@@ -73,4 +73,16 @@
 ┊73┊73┊      }
 ┊74┊74┊    ]
 ┊75┊75┊  };
-┊76┊  ┊});🚫↵
+┊  ┊76┊});
+┊  ┊77┊
+┊  ┊78┊Meteor.publish('user', function () {
+┊  ┊79┊  if (!this.userId) {
+┊  ┊80┊    return;
+┊  ┊81┊  }
+┊  ┊82┊
+┊  ┊83┊  const profile = Users.findOne(this.userId).profile || {};
+┊  ┊84┊
+┊  ┊85┊  return Pictures.collection.find({
+┊  ┊86┊    _id: profile.pictureId
+┊  ┊87┊  });
+┊  ┊88┊});
```
[}]: #

We will also modify the `users` and `chats` publication, so each user will contain its corresponding picture document as well:

[{]: <helper> (diff_step 12.28)
#### Step 12.28: Added images to users publication

##### Changed server/publications.ts
```diff
@@ -1,7 +1,7 @@
 ┊1┊1┊import { Meteor } from 'meteor/meteor';
 ┊2┊2┊import { Mongo } from 'meteor/mongo';
 ┊3┊3┊import { Chats, Messages, Pictures, Users } from '../imports/collections';
-┊4┊ ┊import { Chat, Message, User } from '../imports/models';
+┊ ┊4┊import { Chat, Message, Picture, User } from '../imports/models';
 ┊5┊5┊
 ┊6┊6┊Meteor.publishComposite('users', function(
 ┊7┊7┊  pattern: string
```
```diff
@@ -24,7 +24,17 @@
 ┊24┊24┊        fields: { profile: 1 },
 ┊25┊25┊        limit: 15
 ┊26┊26┊      });
-┊27┊  ┊    }
+┊  ┊27┊    },
+┊  ┊28┊
+┊  ┊29┊    children: [
+┊  ┊30┊      <PublishCompositeConfig1<User, Picture>> {
+┊  ┊31┊        find: (user) => {
+┊  ┊32┊          return Pictures.collection.find(user.profile.pictureId, {
+┊  ┊33┊            fields: { url: 1 }
+┊  ┊34┊          });
+┊  ┊35┊        }
+┊  ┊36┊      }
+┊  ┊37┊    ]
 ┊28┊38┊  };
 ┊29┊39┊});
```
[}]: #

[{]: <helper> (diff_step 12.29)
#### Step 12.29: Add images to chats publication

##### Changed server/publications.ts
```diff
@@ -79,7 +79,16 @@
 ┊79┊79┊          }, {
 ┊80┊80┊            fields: { profile: 1 }
 ┊81┊81┊          });
-┊82┊  ┊        }
+┊  ┊82┊        },
+┊  ┊83┊        children: [
+┊  ┊84┊          <PublishCompositeConfig2<Chat, User, Picture>> {
+┊  ┊85┊            find: (user, chat) => {
+┊  ┊86┊              return Pictures.collection.find(user.profile.pictureId, {
+┊  ┊87┊                fields: { url: 1 }
+┊  ┊88┊              });
+┊  ┊89┊            }
+┊  ┊90┊          }
+┊  ┊91┊        ]
 ┊83┊92┊      }
 ┊84┊93┊    ]
 ┊85┊94┊  };
```
[}]: #

Since we already set up some collection hooks on the users collection, we can take it a step further by defining collection hooks on the chat collection, so whenever a chat is being removed, all its corresponding messages will be removed as well:

[{]: <helper> (diff_step 12.3)
#### Step 12.3: Create PictureService with utils for files

##### Added client/imports/services/picture.ts
```diff
@@ -0,0 +1,80 @@
+┊  ┊ 1┊import { Injectable } from '@angular/core';
+┊  ┊ 2┊import { Platform } from 'ionic-angular';
+┊  ┊ 3┊import { ImagePicker } from 'ionic-native';
+┊  ┊ 4┊import { UploadFS } from 'meteor/jalik:ufs';
+┊  ┊ 5┊
+┊  ┊ 6┊@Injectable()
+┊  ┊ 7┊export class PictureService {
+┊  ┊ 8┊  constructor(private platform: Platform) {
+┊  ┊ 9┊  }
+┊  ┊10┊
+┊  ┊11┊  select(): Promise<Blob> {
+┊  ┊12┊    if (!this.platform.is('cordova') || !this.platform.is('mobile')) {
+┊  ┊13┊      return new Promise((resolve, reject) => {
+┊  ┊14┊        try {
+┊  ┊15┊          UploadFS.selectFile((file: File) => {
+┊  ┊16┊            resolve(file);
+┊  ┊17┊          });
+┊  ┊18┊        }
+┊  ┊19┊        catch (e) {
+┊  ┊20┊          reject(e);
+┊  ┊21┊        }
+┊  ┊22┊      });
+┊  ┊23┊    }
+┊  ┊24┊
+┊  ┊25┊    return ImagePicker.getPictures({maximumImagesCount: 1}).then((URL: string) => {
+┊  ┊26┊      return this.convertURLtoBlob(URL);
+┊  ┊27┊    });
+┊  ┊28┊  }
+┊  ┊29┊
+┊  ┊30┊  convertURLtoBlob(URL: string): Promise<Blob> {
+┊  ┊31┊    return new Promise((resolve, reject) => {
+┊  ┊32┊      const image = document.createElement('img');
+┊  ┊33┊
+┊  ┊34┊      image.onload = () => {
+┊  ┊35┊        try {
+┊  ┊36┊          const dataURI = this.convertImageToDataURI(image);
+┊  ┊37┊          const blob = this.convertDataURIToBlob(dataURI);
+┊  ┊38┊
+┊  ┊39┊          resolve(blob);
+┊  ┊40┊        }
+┊  ┊41┊        catch (e) {
+┊  ┊42┊          reject(e);
+┊  ┊43┊        }
+┊  ┊44┊      };
+┊  ┊45┊
+┊  ┊46┊      image.src = URL;
+┊  ┊47┊    });
+┊  ┊48┊  }
+┊  ┊49┊
+┊  ┊50┊  convertImageToDataURI(image: HTMLImageElement): string {
+┊  ┊51┊    // Create an empty canvas element
+┊  ┊52┊    const canvas = document.createElement('canvas');
+┊  ┊53┊    canvas.width = image.width;
+┊  ┊54┊    canvas.height = image.height;
+┊  ┊55┊
+┊  ┊56┊    // Copy the image contents to the canvas
+┊  ┊57┊    const context = canvas.getContext('2d');
+┊  ┊58┊    context.drawImage(image, 0, 0);
+┊  ┊59┊
+┊  ┊60┊    // Get the data-URL formatted image
+┊  ┊61┊    // Firefox supports PNG and JPEG. You could check image.src to
+┊  ┊62┊    // guess the original format, but be aware the using 'image/jpg'
+┊  ┊63┊    // will re-encode the image.
+┊  ┊64┊    const dataURL = canvas.toDataURL('image/png');
+┊  ┊65┊
+┊  ┊66┊    return dataURL.replace(/^data:image\/(png|jpg);base64,/, '');
+┊  ┊67┊  }
+┊  ┊68┊
+┊  ┊69┊  convertDataURIToBlob(dataURI): Blob {
+┊  ┊70┊    const binary = atob(dataURI);
+┊  ┊71┊
+┊  ┊72┊    // Write the bytes of the string to a typed array
+┊  ┊73┊    const charCodes = Object.keys(binary)
+┊  ┊74┊      .map<number>(Number)
+┊  ┊75┊      .map<number>(binary.charCodeAt.bind(binary));
+┊  ┊76┊
+┊  ┊77┊    // Build blob with typed array
+┊  ┊78┊    return new Blob([new Uint8Array(charCodes)], {type: 'image/jpeg'});
+┊  ┊79┊  }
+┊  ┊80┊}
```
[}]: #

We will now update the `updateProfile` method in the server to accept `pictureId`, so whenever we pick up a new profile picture the server won't reject it:

[{]: <helper> (diff_step 12.31)
#### Step 12.31: Allow updating pictureId

##### Changed server/methods.ts
```diff
@@ -61,7 +61,8 @@
 ┊61┊61┊      'User must be logged-in to create a new chat');
 ┊62┊62┊
 ┊63┊63┊    check(profile, {
-┊64┊  ┊      name: nonEmptyString
+┊  ┊64┊      name: nonEmptyString,
+┊  ┊65┊      pictureId: Match.Maybe(nonEmptyString)
 ┊65┊66┊    });
 ┊66┊67┊
 ┊67┊68┊    Meteor.users.update(this.userId, {
```
[}]: #

Now we will update the users fabrication in our server's initialization, so instead of using hard-coded URLs, we will insert them as new documents to the `PicturesCollection`:

[{]: <helper> (diff_step 12.32)
#### Step 12.32: Update creation of users stubs

##### Changed server/main.ts
```diff
@@ -1,8 +1,7 @@
 ┊1┊1┊import { Accounts } from 'meteor/accounts-base';
 ┊2┊2┊import { Meteor } from 'meteor/meteor';
-┊3┊ ┊import * as Moment from 'moment';
 ┊4┊3┊import { Chats, Messages, Users } from '../imports/collections';
-┊5┊ ┊import { MessageType } from '../imports/models';
+┊ ┊4┊import { MessageType, Picture } from '../imports/models';
 ┊6┊5┊
 ┊7┊6┊Meteor.startup(() => {
 ┊8┊7┊  if (Meteor.settings) {
```
```diff
@@ -14,43 +13,74 @@
 ┊14┊13┊    return;
 ┊15┊14┊  }
 ┊16┊15┊
+┊  ┊16┊  let picture = importPictureFromUrl({
+┊  ┊17┊    name: 'man1.jpg',
+┊  ┊18┊    url: 'https://randomuser.me/api/portraits/men/1.jpg'
+┊  ┊19┊  });
+┊  ┊20┊
 ┊17┊21┊  Accounts.createUserWithPhone({
 ┊18┊22┊    phone: '+972540000001',
 ┊19┊23┊    profile: {
 ┊20┊24┊      name: 'Ethan Gonzalez',
-┊21┊  ┊      picture: 'https://randomuser.me/api/portraits/men/1.jpg'
+┊  ┊25┊      pictureId: picture._id
 ┊22┊26┊    }
 ┊23┊27┊  });
 ┊24┊28┊
+┊  ┊29┊  picture = importPictureFromUrl({
+┊  ┊30┊    name: 'lego1.jpg',
+┊  ┊31┊    url: 'https://randomuser.me/api/portraits/lego/1.jpg'
+┊  ┊32┊  });
+┊  ┊33┊
 ┊25┊34┊  Accounts.createUserWithPhone({
 ┊26┊35┊    phone: '+972540000002',
 ┊27┊36┊    profile: {
 ┊28┊37┊      name: 'Bryan Wallace',
-┊29┊  ┊      picture: 'https://randomuser.me/api/portraits/lego/1.jpg'
+┊  ┊38┊      pictureId: picture._id
 ┊30┊39┊    }
 ┊31┊40┊  });
 ┊32┊41┊
+┊  ┊42┊  picture = importPictureFromUrl({
+┊  ┊43┊    name: 'woman1.jpg',
+┊  ┊44┊    url: 'https://randomuser.me/api/portraits/women/1.jpg'
+┊  ┊45┊  });
+┊  ┊46┊
 ┊33┊47┊  Accounts.createUserWithPhone({
 ┊34┊48┊    phone: '+972540000003',
 ┊35┊49┊    profile: {
 ┊36┊50┊      name: 'Avery Stewart',
-┊37┊  ┊      picture: 'https://randomuser.me/api/portraits/women/1.jpg'
+┊  ┊51┊      pictureId: picture._id
 ┊38┊52┊    }
 ┊39┊53┊  });
 ┊40┊54┊
+┊  ┊55┊  picture = importPictureFromUrl({
+┊  ┊56┊    name: 'woman2.jpg',
+┊  ┊57┊    url: 'https://randomuser.me/api/portraits/women/2.jpg'
+┊  ┊58┊  });
+┊  ┊59┊
 ┊41┊60┊  Accounts.createUserWithPhone({
 ┊42┊61┊    phone: '+972540000004',
 ┊43┊62┊    profile: {
 ┊44┊63┊      name: 'Katie Peterson',
-┊45┊  ┊      picture: 'https://randomuser.me/api/portraits/women/2.jpg'
+┊  ┊64┊      pictureId: picture._id
 ┊46┊65┊    }
 ┊47┊66┊  });
 ┊48┊67┊
+┊  ┊68┊  picture = importPictureFromUrl({
+┊  ┊69┊    name: 'man2.jpg',
+┊  ┊70┊    url: 'https://randomuser.me/api/portraits/men/2.jpg'
+┊  ┊71┊  });
+┊  ┊72┊
 ┊49┊73┊  Accounts.createUserWithPhone({
 ┊50┊74┊    phone: '+972540000005',
 ┊51┊75┊    profile: {
 ┊52┊76┊      name: 'Ray Edwards',
-┊53┊  ┊      picture: 'https://randomuser.me/api/portraits/men/2.jpg'
+┊  ┊77┊      pictureId: picture._id
 ┊54┊78┊    }
 ┊55┊79┊  });
-┊56┊  ┊});🚫↵
+┊  ┊80┊});
+┊  ┊81┊
+┊  ┊82┊function importPictureFromUrl(options: { name: string, url: string }): Picture {
+┊  ┊83┊  const description = { name: options.name };
+┊  ┊84┊
+┊  ┊85┊  return Meteor.call('ufsImportURL', options.url, description, 'pictures');
+┊  ┊86┊}
```
[}]: #

To avoid some unexpected behaviors, we will reset our data-base so our server can re-fabricate the data:

    $ meteor reset

We will now update the `ChatsPage` to add the belonging picture for each chat during transformation:

[{]: <helper> (diff_step 12.33)
#### Step 12.33: Fetch user image from server

##### Changed client/imports/pages/chats/chats.ts
```diff
@@ -3,8 +3,8 @@
 ┊ 3┊ 3┊import { MeteorObservable } from 'meteor-rxjs';
 ┊ 4┊ 4┊import * as Moment from 'moment';
 ┊ 5┊ 5┊import { Observable, Subscriber } from 'rxjs';
-┊ 6┊  ┊import { Chats, Messages, Users } from '../../../../imports/collections';
-┊ 7┊  ┊import { Chat, Message, MessageType } from '../../../../imports/models';
+┊  ┊ 6┊import { Chats, Messages, Users, Pictures } from '../../../../imports/collections';
+┊  ┊ 7┊import { Chat, Message } from '../../../../imports/models';
 ┊ 8┊ 8┊import { ChatsOptionsComponent } from './chats-options';
 ┊ 9┊ 9┊import { MessagesPage } from '../messages/messages';
 ┊10┊10┊import template from './chats.html';
```
```diff
@@ -50,7 +50,7 @@
 ┊50┊50┊
 ┊51┊51┊        if (receiver) {
 ┊52┊52┊          chat.title = receiver.profile.name;
-┊53┊  ┊          chat.picture = receiver.profile.picture;
+┊  ┊53┊          chat.picture = Pictures.getPictureUrl(receiver.profile.pictureId);
 ┊54┊54┊        }
 ┊55┊55┊
 ┊56┊56┊        // This will make the last message reactive
```
[}]: #

And we will do the same in the `NewChatComponent`:

[{]: <helper> (diff_step 12.34)
#### Step 12.34: Use the new pictureId field for new chat modal

##### Changed client/imports/pages/chats/new-chat.html
```diff
@@ -26,7 +26,7 @@
 ┊26┊26┊<ion-content class="new-chat">
 ┊27┊27┊  <ion-list class="users">
 ┊28┊28┊    <button ion-item *ngFor="let user of users | async" class="user" (click)="addChat(user)">
-┊29┊  ┊      <img class="user-picture" [src]="user.profile.picture">
+┊  ┊29┊      <img class="user-picture" [src]="getPic(user.profile.pictureId)">
 ┊30┊30┊      <h2 class="user-name">{{user.profile.name}}</h2>
 ┊31┊31┊    </button>
 ┊32┊32┊  </ion-list>
```
[}]: #

[{]: <helper> (diff_step 12.35)
#### Step 12.35: Implement getPic

##### Changed client/imports/pages/chats/new-chat.ts
```diff
@@ -3,7 +3,7 @@
 ┊3┊3┊import { MeteorObservable } from 'meteor-rxjs';
 ┊4┊4┊import { _ } from 'meteor/underscore';
 ┊5┊5┊import { Observable, Subscription, BehaviorSubject } from 'rxjs';
-┊6┊ ┊import { Chats, Users } from '../../../../imports/collections';
+┊ ┊6┊import { Chats, Pictures, Users } from '../../../../imports/collections';
 ┊7┊7┊import { User } from '../../../../imports/models';
 ┊8┊8┊import template from './new-chat.html';
 ┊9┊9┊
```
```diff
@@ -107,4 +107,8 @@
 ┊107┊107┊
 ┊108┊108┊    alert.present();
 ┊109┊109┊  }
-┊110┊   ┊}🚫↵
+┊   ┊110┊
+┊   ┊111┊  getPic(pictureId): string {
+┊   ┊112┊    return Pictures.getPictureUrl(pictureId);
+┊   ┊113┊  }
+┊   ┊114┊}
```
[}]: #

[{]: <helper> (nav_step next_ref="https://angular-meteor.com/tutorials/whatsapp2/meteor/native-mobile" prev_ref="https://angular-meteor.com/tutorials/whatsapp2/meteor/google-maps")
| [< Previous Step](https://angular-meteor.com/tutorials/whatsapp2/meteor/google-maps) | [Next Step >](https://angular-meteor.com/tutorials/whatsapp2/meteor/native-mobile) |
|:--------------------------------|--------------------------------:|
[}]: #

