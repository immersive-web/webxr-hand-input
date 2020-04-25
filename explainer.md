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

Each hand is made up many bones, connected by _joints_. We name them with their connected bone, for example `INDEX_PHALANX_DISTAL` is the joint closer to the wrist connected to the distal phalanx bone of the index finger. The `*_PHALANX_TIP` "joints" locate the tips of the fingers. The `WRIST` joint is located at the composite joint between the wrist and forearm.

The joint spaces can be accessed via indexing, for example to access the middle knuckle joint one would use:

```js
let joint = inputSource.hand[XRHand.MIDDLE_PHALANX_PROXIMAL];
```

Not all devices support all joints, this indexing getter will return `null` when accessing a joint that is not supported by the current user agent or device. This will not change for a given input source. If a joint is supported but not currently being tracked, the getter will still produce the `XRJointSpace`, but it will return `null` when run through `getPose` (etc).

Each joint space is an `XRSpace`, with its `-Y` direction pointing perpendicular to the skin, outwards from the palm, and `-Z` direction pointing along their associated bone, away from the wrist. This space will return null poses when the joint loses tracking.

For `*_TIP` joints where there is no associated bone, the `-Z` direction is the same as that for the associated `DISTAL` joint, i.e. the direction is along that of the previous bone.


## Obtaining radii

If you wish to obtain a radius ("distance from skin") for a joint, instead of using `getPose()`, you can use `getJointPose()` on the joint space. The `radius` can be accessed on the joint pose.

```js
let radius = frame.getJointPose(joint).radius;
```


## Displaying hand models using this API

Ideally, most of this will be handled by an external library or the framework being used by the user.

A simple skeleton can be displayed as follows:

```js
const orderedJoints = [
   [XRHand.THUMB_METACARPAL, XRHand.THUMB_PHALANX_PROXIMAL, XRHand.THUMB_PHALANX_DISTAL, XRHand.THUMB_PHALANX_TIP],
   [XRHand.INDEX_METACARPAL, XRHand.INDEX_PHALANX_PROXIMAL, XRHand.INDEX_PHALANX_INTERMEDIATE, XRHand.INDEX_PHALANX_DISTAL, XRHand.INDEX_PHALANX_TIP]
   [XRHand.MIDDLE_METACARPAL, XRHand.MIDDLE_PHALANX_PROXIMAL, XRHand.MIDDLE_PHALANX_INTERMEDIATE, XRHand.MIDDLE_PHALANX_DISTAL, XRHand.MIDDLE_PHALANX_TIP]
   [XRHand.RING_METACARPAL, XRHand.RING_PHALANX_PROXIMAL, XRHand.RING_PHALANX_INTERMEDIATE, XRHand.RING_PHALANX_DISTAL, XRHand.RING_PHALANX_TIP]
   [XRHand.LITTLE_METACARPAL, XRHand.LITTLE_PHALANX_PROXIMAL, XRHand.LITTLE_PHALANX_INTERMEDIATE, XRHand.LITTLE_PHALANX_DISTAL, XRHand.LITTLE_PHALANX_TIP]
];

function renderSkeleton(inputSource, frame, renderer) {
   let wrist = inputSource.hand[XRHand.WRIST];
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
   let tip = frame.getPose(inputSource.hand[XRHand.INDEX_PHALANX_TIP], renderer.referenceSpace);
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
   for (finger of [[XRHand.INDEX_PHALANX_TIP, XRHand.INDEX_METACARPAL],
                  [XRHand.MIDDLE_PHALANX_TIP, XRHand.MIDDLE_METACARPAL],
                  [XRHand.RING_PHALANX_TIP, XRHand.RING_METACARPAL],
                  [XRHand.LITTLE_PHALANX_TIP, XRHand.LITTLE_METACARPAL]]) {
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

## Appendix: Proposed IDL

```webidl
partial interface XRInputSource {
   readonly attribute XRHand? hand;
}

partial interface XRFrame {
    XRJointPose? getJointPose(XRJointSpace joint, )
}

interface XRJointPose: XRPose {
    readonly attribute float? radius;
}

interface XRJointSpace: Space {}

interface XRHand {
    // needed for the getter
    readonly attribute unsigned long length;
    getter XRJointSpace(unsigned long jointIndex);

    const unsigned long WRIST = ..;

    // potentially: const unsigned long THUMB_TRAPEZIUM = ..;
    const unsigned long THUMB_METACARPAL = ..;
    const unsigned long THUMB_PHALANX_PROXIMAL = ..;
    const unsigned long THUMB_PHALANX_DISTAL = ..;
    const unsigned long THUMB_PHALANX_TIP = ..;

    const unsigned long INDEX_METACARPAL = ..;
    const unsigned long INDEX_PHALANX_PROXIMAL = ..;
    const unsigned long INDEX_PHALANX_INTERMEDIATE = ..;
    const unsigned long INDEX_PHALANX_DISTAL = ..;
    const unsigned long INDEX_PHALANX_TIP = ..;

    const unsigned long MIDDLE_METACARPAL = ..;
    const unsigned long MIDDLE_PHALANX_PROXIMAL = ..;
    const unsigned long MIDDLE_PHALANX_INTERMEDIATE = ..;
    const unsigned long MIDDLE_PHALANX_DISTAL = ..;
    const unsigned long MIDDLE_PHALANX_TIP = ..;

    const unsigned long RING_METACARPAL = ..;
    const unsigned long RING_PHALANX_PROXIMAL = ..;
    const unsigned long RING_PHALANX_INTERMEDIATE = ..;
    const unsigned long RING_PHALANX_DISTAL = ..;
    const unsigned long RING_PHALANX_TIP = ..;

    const unsigned long LITTLE_METACARPAL = ..;
    const unsigned long LITTLE_PHALANX_PROXIMAL = ..;
    const unsigned long LITTLE_PHALANX_INTERMEDIATE = ..;
    const unsigned long LITTLE_PHALANX_DISTAL = ..;
    const unsigned long LITTLE_PHALANX_TIP = ..;
}

```
