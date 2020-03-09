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

The joints can be accessed via indexing, for example to access the middle knuckle joint one would use:

```js
let joint = inputSource.hand[XRJoint.MIDDLE_PHALANX_PROXIMAL];
```

Not all devices support all joints, this indexing getter will return `null` when accessing a joint that is not supported by the current user agent or device. This will not change for a given input source. If a joint is supported but not currently being tracked, the getter will still produce the `XRJoint`, with its associated `space` returning `null` when run through `getPose` (etc).

Each joint has an `XRSpace` `space`, located at its center, with its `-Y` direction pointing perpendicular to the skin, outwards from the palm, and `-Z` direction pointing along their associated bone, away from the wrist. This space will return null poses when the joint loses tracking.

Each joint has a `radius`, which is the distance from the joint to the surrounding skin, and can be used to render a hand skeleton.

For `*_TIP` joints where there is no associated bone, the `-Z` direction is the same as that for the associated `DISTAL` joint, i.e. the direction is along that of the previous bone.

## Displaying hand models using this API

Ideally, most of this will be handled by an external library or the framework being used by the user.

A simple skeleton can be displayed as follows:

```js
const orderedJoints = [
   [XRJoint.THUMB_METACARPAL, XRJoint.THUMB_PHALANX_PROXIMAL, XRJoint.THUMB_PHALANX_DISTAL, XRJoint.THUMB_PHALANX_TIP],
   [XRJoint.INDEX_METACARPAL, XRJoint.INDEX_PHALANX_PROXIMAL, XRJoint.INDEX_PHALANX_INTERMEDIATE, XRJoint.INDEX_PHALANX_DISTAL, XRJoint.INDEX_PHALANX_TIP]
   [XRJoint.MIDDLE_METACARPAL, XRJoint.MIDDLE_PHALANX_PROXIMAL, XRJoint.MIDDLE_PHALANX_INTERMEDIATE, XRJoint.MIDDLE_PHALANX_DISTAL, XRJoint.MIDDLE_PHALANX_TIP]
   [XRJoint.RING_METACARPAL, XRJoint.RING_PHALANX_PROXIMAL, XRJoint.RING_PHALANX_INTERMEDIATE, XRJoint.RING_PHALANX_DISTAL, XRJoint.RING_PHALANX_TIP]
   [XRJoint.LITTLE_METACARPAL, XRJoint.LITTLE_PHALANX_PROXIMAL, XRJoint.LITTLE_PHALANX_INTERMEDIATE, XRJoint.LITTLE_PHALANX_DISTAL, XRJoint.LITTLE_PHALANX_TIP]
];

function renderSkeleton(inputSource, frame, renderer) {
   let wrist = inputSource.hand[XRJoint.WRIST];
   if (!wrist) {
      // this code is written to assume that the wrist joint is exposed
      return;
   }
   renderer.drawSphere(frame, wrist.space, wrist.radius);
   for (finger of orderedJoints) {
      let previous = wrist;
      for (joint of finger) {
         let joint = inputSource.hand[joint];
         if (joint) {
            drawSphere(frame, joint.space, joint.radius);
            drawCylinder(frame,
                         /* from */ previous.space,
                         /* to */ joint.space,
                         /* specify a thinner radius */ joint.radius / 3);
            previous = joint;
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
   let tip = frame.getPose(inputSource.hand[XRJoint.INDEX_PHALANX_TIP], renderer.referenceSpace);
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
   for (finger of [[XRJoint.INDEX_PHALANX_TIP, XRJoint.INDEX_METACARPAL],
                  [XRJoint.MIDDLE_PHALANX_TIP, XRJoint.MIDDLE_METACARPAL],
                  [XRJoint.RING_PHALANX_TIP, XRJoint.RING_METACARPAL],
                  [XRJoint.LITTLE_PHALANX_TIP, XRJoint.LITTLE_METACARPAL]]) {
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
   XRHand? hand;
}

interface XRHand {
    getter XRJoint(unsigned short jointIndex);
}

interface XRJoint {
   XRSpace space;
   float? radius;
   
   const unsigned short WRIST = ..;

   // potentially: const unsigned short THUMB_TRAPEZIUM = ..;
   const unsigned short THUMB_METACARPAL = ..;
   const unsigned short THUMB_PHALANX_PROXIMAL = ..;
   const unsigned short THUMB_PHALANX_DISTAL = ..;
   const unsigned short THUMB_PHALANX_TIP = ..;

   const unsigned short INDEX_METACARPAL = ..;
   const unsigned short INDEX_PHALANX_PROXIMAL = ..;
   const unsigned short INDEX_PHALANX_INTERMEDIATE = ..;
   const unsigned short INDEX_PHALANX_DISTAL = ..;
   const unsigned short INDEX_PHALANX_TIP = ..;

   const unsigned short MIDDLE_METACARPAL = ..;
   const unsigned short MIDDLE_PHALANX_PROXIMAL = ..;
   const unsigned short MIDDLE_PHALANX_INTERMEDIATE = ..;
   const unsigned short MIDDLE_PHALANX_DISTAL = ..;
   const unsigned short MIDDLE_PHALANX_TIP = ..;

   const unsigned short RING_METACARPAL = ..;
   const unsigned short RING_PHALANX_PROXIMAL = ..;
   const unsigned short RING_PHALANX_INTERMEDIATE = ..;
   const unsigned short RING_PHALANX_DISTAL = ..;
   const unsigned short RING_PHALANX_TIP = ..;

   const unsigned short LITTLE_METACARPAL = ..;
   const unsigned short LITTLE_PHALANX_PROXIMAL = ..;
   const unsigned short LITTLE_PHALANX_INTERMEDIATE = ..;
   const unsigned short LITTLE_PHALANX_DISTAL = ..;
   const unsigned short LITTLE_PHALANX_TIP = ..;
}
```