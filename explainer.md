# WebXR Device API - Hand Input

This document describes a design giving developers access to hand-tracking XR systems, building on top of the [WebXR device API](https://immersive-web.github.io/webxr/)

## Use cases and scope

This API primarily exposes the poses of hand skeleton joints. It can be used to render a hand model in VR scenarios, as well perform gesture detection with the hands. It does not provide access to a full hand mesh.

## Accessing this API

This API will only be accessible if a `"hand-tracking"` [XR feature](https://immersive-web.github.io/webxr/#feature-dependencies) is requested.

This API presents itself as an additional field on `XRInputSource`, `hand`. The `hand` attribute will be non-null if the input source supports hand tracking and the feature has been requested.

```js
navigator.xr.requestSession({optionalFeatures: ["hand-tracking"]}).then(...);

function renderFrame(session, frame) {
   // ...

   for (inputSource of session.inputSources) {
      if (inputSource.hand) {
         // render a hand model
         // perform gesture detection
      }
   }
}


```

## Hands and joints

Each hand is made up many bones, connected by _joints_. We name them with their connected bone, for example `index-finger-phalanx-distal` is the joint closer to the wrist connected to the distal phalanx bone of the index finger. The `*-phalanx-tip` "joints" locate the tips of the fingers. The `wrist` joint is located at the composite joint between the wrist and forearm.

The joint spaces can be accessed via indexing, for example to access the middle knuckle joint one would use:

```js
let joint = inputSource.hand["middle-finger-phalanx-distal"];
```

All devices which support hand tracking will support or emulate all joints, so this indexing operation will always return a valid object as long as it is supplied with a valid joint index. If a joint is supported but not currently being tracked, the getter will still produce the `XRJointSpace`, but it will return `null` when run through `getPose` (etc).

Each joint space is an `XRSpace`, with its `-Y` direction pointing perpendicular to the skin, outwards from the palm, and `-Z` direction pointing along their associated bone, away from the wrist. This space will return null poses when the joint loses tracking.

For `*-tip` joints where there is no associated bone, the `-Z` direction is the same as that for the associated `distal` joint, i.e. the direction is along that of the previous bone.


## Obtaining radii

If you wish to obtain a radius ("distance from skin") for a joint, instead of using `getPose()`, you can use `getJointPose()` on the joint space. The `radius` can be accessed on the joint pose.

```js
let radius = frame.getJointPose(joint, referenceSpace).radius;
```


## Displaying hand models using this API

Ideally, most of this will be handled by an external library or the framework being used by the user.

A simple skeleton can be displayed as follows:

```js
const orderedJoints = [
   ["thumb-metacarpal", "thumb-phalanx-proximal", "thumb-phalanx-distal", "thumb-phalanx-tip"],
   ["index-finger-metacarpal", "index-finger-phalanx-proximal", "index-finger-phalanx-intermediate", "index-finger-phalanx-distal", "index-finger-phalanx-tip"]
   ["index-finger-metacarpal", "index-finger-phalanx-proximal", "index-finger-phalanx-intermediate", "index-finger-phalanx-distal", "index-finger-phalanx-tip"]
   ["ring-finger-metacarpal", "ring-finger-phalanx-proximal", "ring-finger-phalanx-intermediate", "ring-finger-phalanx-distal", "ring-finger-phalanx-tip"]
   ["pinky-finger-metacarpal", "pinky-finger-phalanx-proximal", "pinky-finger-phalanx-intermediate", "pinky-finger-phalanx-distal", "pinky-finger-phalanx-tip"]
];

function renderSkeleton(inputSource, frame, renderer) {
   let wrist = inputSource.hand["wrist"];
   if (!wrist) {
      // this code is written to assume that the wrist joint is exposed
      return;
   }
   let wristPose = frame.getJointPose(wrist, renderer.referenceSpace);
   renderer.drawSphere(frame, wristPose.transform, wristPose.radius);
   for (finger of orderedJoints) {
      let previous = wristPose;
      for (joint of finger) {
         let joint = inputSource.hand[joint];
         if (joint) {
            let pose = frame.getJointPose(joint, renderer.referenceSpace);
            drawSphere(frame, pose.transform, pose.radius);
            drawCylinder(frame,
                         /* from */ previous.transform,
                         /* to */ pose.transform,
                         /* specify a thinner radius */ pose.radius / 3);
            previous = pose;
         }
      }
   }
}
```

## Hand interaction using this API

It's useful to be able to use individual fingers for interacting with objects. For example, it's possible to have fingers interacting with spherical buttons in space:

```js
const buttons = [
   {position: [1, 0, 0, 1], radius: 0.1, pressed: false,
    onpress: function() { ... }, onrelease: function() { ... }},
   // ...
];

function checkInteraction(button, inputSource, frame, renderer) {
   let tip = frame.getPose(inputSource.hand["index-finger-phalanx-tip"], renderer.referenceSpace);
   let distance = calculateDistance(tip.transform.position, button.position);
   if (distance < button.radius) {
      if (!button.pressed) {
         button.pressed = true;
         button.onpress();
      }
   } else {
      if (button.pressed) {
         button.pressed = false;
         button.onrelease();
      }
   }
}

function onFrame(frame, renderer) {
   // ...
   for (button of buttons) {
      for (inputSource of frame.session.inputSources) {
         checkInteraction(button, inputSource, frame, renderer);
      }
   }
}

```

## Gesture detection using this API

One can do gesture detection using the position and orientation values of the various fingers. This can get pretty complicated and stateful, but a straightforward example below would be simplistically detecting that the user has made a fist gesture:

```js
function checkFistGesture(inputSource, frame, renderer) {
   for (finger of [["index-finger-phalanx-tip", "index-finger-metacarpal"],
                  ["index-finger-phalanx-tip", "index-finger-metacarpal"],
                  ["ring-finger-phalanx-tip", "ring-finger-metacarpal"],
                  ["pinky-finger-phalanx-tip", "pinky-finger-metacarpal"]]) {
      let tip = finger[0];
      let metacarpal = finger[1];
      let tipPose = frame.getPose(inputSource.hand[tip], renderer.referenceSpace);
      let metacarpalPose = frame.getPose(inputSource.hand[metacarpal], renderer.referenceSpace)
      if (calculateDistance(tipPose.position, metacarpalPose.position) > minimumDistance ||
          !checkOrientation(tipPose.orientation, metacarpalPose.orientation)) {
         return false
      }
   }
   return true;
}

function checkOrientation(tipOrientation, metacarpalOrientation) {
   let tipDirection = applyOrientation(tipOrientation, [0, 0, -1]); // -Z axis of tip
   let palmDirection = applyOrientation(metacarpalOrientation, [0, -1, 0]) // -Y axis of metacarpal

   if (1 - dotProduct(tipDirection, palmDirection) < minimumDeviation) {
      return true;
   } else {
      return false;
   }
}
```

## Efficiently obtaining hand poses

Each use of  `getPose()` allocates one short-lived `XRPose` object, one `XRRigidTransform` object, and at least one `DOMPointReadOnly` or `Float32Array`. For 25 joints per hand and two hands, this is 150-250 objects created per frame. This can have noticeable performance implications, especially around garbage collection.

To avoid that, we provide a `fillPoses()` (and `fillJointRadii`) API which can be used to efficiently obtain all the transforms for a hand at once.

```js
let poses1 = new Float32Array(16 * 25);
let radii1 = new Float32Array(25);
function onFrame(frame, renderer) {
  let hand1 = frame.session.inputSources[0].hand;
  frame.fillPoses(hand1.values(), renderer.referenceSpace, poses1);
  frame.fillJointRadii(hand1.values(), radii1);
  renderer.drawHand(poses1, radii1);
  // do something similar for second hand
}
```

## Privacy and Security Considerations

The concept of exposing hand input could pose a risk to users’ privacy. For example, data produced by some hand-tracking systems could potentially enable sites to infer users’ gestural behaviors or approximate hand size, make it apparent to sites that a user is missing fingers or parts of fingers, or allow detection of tremors or other medical conditions.

Implementations are required to employ strategies to mitigate these risks, such as:
- Reducing the precision and sampling rate of data
- Adding noise or rounding data
- Return the same hand geometry/size for all users
- Emulating values for joints if the implementation isn’t capable of detecting them or the user does not have them.

This specification requires implementations to include sufficient mitigations to protect users’ privacy.

## Appendix: Proposed IDL

```webidl
partial interface XRInputSource {
   readonly attribute XRHand? hand;
}

partial interface XRFrame {
   XRJointPose? getJointPose(XRJointSpace joint, XRSpace relativeTo);
   boolean fillJointRadii(sequence<XRJointSpace> jointSpaces, Float32Array radii);
   boolean fillPoses(sequence<XRSpace> spaces, XRSpace baseSpace, Float32Array transforms);
}

interface XRJointPose: XRPose {
   readonly attribute float? radius;
}

interface XRJointSpace: XRSpace {
   readonly attribute XRHandJoint jointName;
}

enum XRHandJoint {
   "wrist",

   "thumb-metacarpal",
   "thumb-phalanx-proximal",
   "thumb-phalanx-distal",
   "thumb-tip",

   "index-finger-metacarpal",
   "index-finger-phalanx-proximal",
   "index-finger-phalanx-intermediate",
   "index-finger-phalanx-distal",
   "index-finger-tip",

   "middle-finger-metacarpal",
   "middle-finger-phalanx-proximal",
   "middle-finger-phalanx-intermediate",
   "middle-finger-phalanx-distal",
   "middle-finger-tip",

   "ring-finger-metacarpal",
   "ring-finger-phalanx-proximal",
   "ring-finger-phalanx-intermediate",
   "ring-finger-phalanx-distal",
   "ring-finger-tip",

   "pinky-finger-metacarpal",
   "pinky-finger-phalanx-proximal",
   "pinky-finger-phalanx-intermediate",
   "pinky-finger-phalanx-distal",
   "pinky-finger-tip"
};

interface XRHand {
   iterable<XRHandJoint, XRJointSpace>;

   readonly attribute unsigned long size;
   XRJointSpace get(XRHandJoint key);
};


```
