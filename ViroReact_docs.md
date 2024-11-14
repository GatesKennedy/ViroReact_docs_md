
***NOTE: This document is incomplete. The comprehensive docs can be found*** [here](https://viro-community.readme.io/docs/overview)

---

# Overview

⚠️ **Note: AR is limited to ARKit and ARCore supported devices**
- ARKit supported devices can be found [here](https://developer.apple.com/library/content/documentation/DeviceInformation/Reference/iOSDeviceCompatibility/DeviceCompatibilityMatrix/DeviceCompatibilityMatrix.html) (Look for `arkit`)
- ARCore supported devices can be found [here](https://developers.google.com/ar/discover/supported-devices)

The Viro platform supports Augmented Reality (AR) development through various AR-specific components and features. This guide provides an overview of AR and the components and features that enable developers to build AR experiences.

## Building AR Experiences

The Viro platform provides a large suite of components that developers can leverage to build their AR experiences. Unlike traditional 3D rendering and VR, AR experiences are meant to be responsive to the user's real world. The platform provides several AR-specific components and features detailed below:

### AR Components

These components are built specifically for and used only in AR on the Viro platform:

| Component | Description |
|-----------|-------------|
| ViroARSceneNavigator | Top-Level React Native component that presents a view onto which ViroReact renders `ViroARScene`s. |
| ViroARScene | Logical container that contains all components necessary for a singular AR experience. Maintains a scene graph rendered in AR atop the real-world. Multiple instances may be used for multiple "experiences". |
| ViroARPlane | Component that manages positioning of ViroReact components relative to planes discovered by the AR system. Can be used for table-top games or mounting virtual pictures on walls. |
| ViroARPlaneSelector | Convenience component wrapping `ViroARPlane` to let users select which plane to use for displaying content. |
| ViroARImageMarker | Enables placing objects relative to images detected by the AR system. Can create virtual UIs over real-world images or animate movie posters. |
| ViroARTrackingTargets | Allows creating targets for use with ViroARImageMarker. API similar to materials. |

Note: `ViroARSceneNavigator` and `ViroARScene` replace `ViroSceneNavigator` and `ViroScene` for VR while adding AR-specific methods and properties.

### AR Features

The AR system provides additional information through these features:

| Feature | Location | Description |
|---------|-----------|-------------|
| 6 Degrees of Camera Movement | `getCameraOrientationAsync()` in ViroARScene | Camera moves with user movement, keeping virtual objects positioned correctly |
| Video Pass Through | Automatic | Back camera serves as background for AR rendering |
| Ambient Light Estimation | `onAmbientLightUpdate` in ViroARScene | Provides light intensity and color temperature estimates |
| FixedToWorld/FixedToPlane Dragging | `dragType` property on draggable components | Enables dragging objects with fixed positions in real world |
| AR-based Hit Tests | Methods in ViroARScene | Detects real-world points/objects from user interaction |
| Portals | ViroPortalScene and ViroPortal components | Creates virtual portals between real and virtual worlds |
| Post Processing Effects | `postProcessEffects` prop in ViroARScene | Applies pre-built effects to scenes |
| Video/Still Capture | Recording APIs in ViroARSceneNavigator | Records virtual objects overlaid on camera view |

With these features, developers can create realistic AR experiences with proper lighting, environmental interaction, otherworldly portals, and more.

---

# Tracking and Anchors

When using ViroARScene, the first thing you'll notice is the camera feed rendered as the background. This camera background represents the real world. Viro enables you to fuse virtual objects and UI with that real world through tracking and anchors.

## Camera Tracking

The Viro platform supports development of 6 degrees-of-freedom (6DOF) AR experiences, meaning the platform responds to the user's rotation and position as they move about the world by moving the camera. The platform maintains the same right-handed coordinate system as it does in VR, where:

- Initial camera position: `[0,0,0]`
- Forward vector: `[0, 0, -1]`  
- Up vector: `[0,1,0]`

When the user first enters the AR experience, the camera remains at [0,0,0] while the underlying AR system gets its bearings. After a few moments:
1. Camera tracking completes
2. Developer receives the `onTrackingInitialized` callback in ViroARScene
3. Camera begins tracking user movements around their world

## Anchor Detection

The primary way to add virtual content to the real-world is to listen for anchors detected by the AR system. Anchors include:
- Vertical or horizontal planes
- Images (like posters or markers) found in the real world

For information on attaching content to images, see Image Recognition. This guide describes two methods of adding content to detected planes through ViroARPlane:

## Automatic Anchoring

To enable automatic anchoring:

1. Provide `minHeight` and `minWidth` properties to the ViroARPlane component
2. Embed desired content as children to display when the plane is detected

When the AR system finds a matching plane:
- ViroARPlane anchors to the real world plane
- Child components become visible in the plane's coordinate system
- Real-world plane updates are provided through callbacks:
  - `onAnchorFound`
  - `onAnchorUpdated`
  - `onAnchorRemoved`

Example - displaying a box when detecting a 0.5m × 0.5m plane:

```javascript
<ViroARPlane minHeight={.5} minWidth={.5} alignment={"Horizontal"}>
    <ViroBox position={[0, .25, 0]} scale={[.5, .5, .5]} />
</ViroARPlane>
```

## Manual Anchoring

Manual anchoring allows developers to specifically choose which plane to use by listening for all incoming anchors, rather than having the platform automatically select the first available plane that matches requirements.

To implement manual anchoring:

1. Add anchor listeners to the ViroARScene component:
   - `onAnchorFound`
   - `onAnchorUpdated`
   - `onAnchorRemoved`
2. When a suitable anchor is found, set the `anchorId` property of the desired plane in your render function

Example:

```javascript
<ViroARPlane anchorId={foundAnchor.anchorId}>
  <ViroBox position={[0, .25, 0]} scale={[.5, .5, .5]} />
</ViroARPlane>
```

---

# Interaction

Viro supports numerous mechanisms through which users can interact with both the real world and the virtual UI. These are detailed below.

## AR Hit Testing

`<ViroARScene>` provides various methods for "hit-testing" against the real-world. These hit tests determine what real-world features exist at a given point on the 2D screen. Important notes:

- A single 2D point corresponds to a 3D ray in the scene
- Multiple results may be returned (each at different depths)
- Results can be:
  - Anchors (like planes)
  - Feature points not yet fully identified

Example of AR hit testing using a ray:

```javascript
this.refs["arscene"].performARHitTestWithRay(orientation.forward).then((results)=> {
  for (let i = 0; i < results.length; i++) {
    let result = results[i];
    if (result.type == "ExistingPlaneUsingExtent") {
      // We hit a plane, do something!    
    }  
  }
})
```

For more details on AR hit tests, refer to the ViroARScene and ARHitTestResult documentation.

## Fixed to World Dragging

### Default Dragging Behavior
When dragging a `<ViroNode>` (by setting its `onDrag` property), the Node maintains a fixed distance from the user by default. This creates the effect of dragging across the inner surface of a sphere and is called `FixedDistance` dragging.

### FixedToWorld Dragging
Viro also supports `FixedToWorld` dragging in AR, which offers different behavior:
- Instead of maintaining fixed distance from user
- Node's distance is determined by intersection with nearest real-world object
- Useful for dragging virtual objects across real-world surfaces

To enable FixedToWorld dragging:
```javascript
// Set the node's dragType property to "FixedToWorld"
<ViroNode dragType="FixedToWorld" />
```

## Fixed to Plane Dragging

Fixed to Plane dragging constrains a `<ViroNode>` to move along a specified plane. Key features:

- Node cannot leave the defined plane
- Can set maximum camera distance to prevent infinite dragging
- Requires setting both `dragType` and `dragPlane` properties

### dragPlane Configuration Type
```typescript
type dragPlane = {  
  planePoint: number[],    // Any point on the plane
  planeNormal: number[],   // Normal vector of the plane
  maxDistance: number      // Maximum allowed distance from camera
}
```

### Example: Horizontal Plane Dragging
This example constrains a Box to:
- Move only on a horizontal plane 1 meter below the user
- Stay within 5 meters of the camera

```javascript
<ViroNode
  position={[0, -1, -1]}
  dragType="FixedToPlane"
  dragPlane={{
    planePoint: [0, -1, 0],
    planeNormal: [0, 1, 0],
    maxDistance: 5
  }} />
```

Each dragging method serves different interaction needs:
- `FixedDistance`: Default spherical movement
- `FixedToWorld`: Real-world surface interaction
- `FixedToPlane`: Constrained plane movement

---

# Image Recognition

Image recognition is a key component of AR: it enables you to *interpret* the real world and *respond* to it accordingly. This guide provides an overview of Viro's image recognition capabilities.

## Image Targets

Image targets are reference images that Viro will recognize and track. For example, with a Tesla logo as an image target, your application can:
- Detect when the logo appears
- Trigger callbacks on detection
- Create virtual UI elements around the logo

### Basic Implementation

Image targets in Viro are represented by ViroARTrackingTargets and can be constructed from:
- JPG images
- PNG images
- Other image formats

To implement image tracking:

1. Add a ViroARImageMarker component to either:
   - `<ViroARScene>`
   - Any `<ViroNode>`
2. When Viro detects the target image:
   - Content within ViroARImageMarker renders
   - Marker continuously tracks and syncs with the detected image

### Basic Example
```javascript
// In your render function, add an image marker that references the target
<ViroARScene>
  <ViroARImageMarker target={"targetOne"}>
    <ViroBox position={[0, .25, 0]} scale={[.5, .5, .5]} />
  </ViroARImageMarker>
</ViroARScene>

// Outside of the render function, register the target
ViroARTrackingTargets.createTargets({
  "targetOne": {
    source: require('res/targetOne.html'),
    orientation: "Up",
    physicalWidth: 0.1 // real world width in meters  
  },
});
```

### Advanced Example: Black Panther Poster

This example demonstrates creating an interactive AR experience with a movie poster:
1. Detects a Black Panther movie poster
2. Loads a 3D Black Panther model
3. Animates the character "jumping" out of the poster

```javascript
// In your render function:
<ViroARScene>
  <ViroAmbientLight color="#ffffff" intensity={200} />
  <ViroARImageMarker
    target={"poster"}
    onAnchorFound={this._onAnchorFound}
    pauseUpdates={this.state.pauseUpdates}>
      <ViroNode
        position={[0, -.1, 0]}
        scale={[0, 0, 0]}
        rotation={[-90, 0, 0]}
        dragType="FixedToWorld" 
        onDrag={() => { }}
        animation={{
          name: "scaleModel",
          run: this.state.playAnim
        }}>
      <Viro3DObject
        onLoadEnd={this._onModelLoad}
        source={require('./res/blackpanther/object_bpanther_anim.vrx')}
        position={[0, -1.45, 0]}
        scale={[.9, .9, .9]}
        animation={{
          name: "01",
          run: true,
          loop: false,
          onFinish: this._onFinish
        }}
        type="VRX" />
    </ViroNode>
  </ViroARImageMarker>
</ViroARScene>

//Outside the render function:
ViroARTrackingTargets.createTargets({
  "poster": {
    source: require('res/blackpanther.html'),
    orientation: "Up",
    physicalWidth: 0.6096 // real world width in meters
  }
});
```

## Continuous Image Tracking (iOS 12 Only)

ARKit 2.0 introduced continuous Image Tracking for smooth marker tracking as users move. Features:

- Enables fluid tracking beyond initial detection
- Controlled via ViroARSceneNavigator's `numberOfTrackedImages` property
- Tracks concurrent images based on specified limit
- Example: With limit of 3
  - First 3 visible images are tracked
  - When tracked image leaves view, next untracked image starts tracking
  - Performance decreases with higher tracking numbers

## Image Target Quality

For optimal image target recognition:

1. Use Google's Image Target quality checker tool:
   - Available for OSX and Windows
   - URL: https://developers.google.com/ar/develop/c/augmented-images/arcoreimg
   - Works for both iOS and Android platforms

2. Best Practices:
   - Keep tracked image count low for better performance
   - Use high-contrast images
   - Ensure proper lighting conditions
   - Consider physical size of target images

---

==============
API REFERENCE
==============


# ViroScene

ViroScene is the top-most component for each scene in a Viro application. For more information, see our Scene Guide.

## Basic Usage

```javascript
<ViroScene onClick={this._onClick}>
    <ViroSphere position={[0, 0, -1]} />
</ViroScene>
```

## Props

### dragPlane
**Type**: PropTypes.shape
```typescript
{
  planePoint: PropTypes.arrayOf(PropTypes.number),
  planeNormal: PropTypes.arrayOf(PropTypes.number),
  maxDistance: PropTypes.number
}
```
Used with `"FixedToPlane"` drag type to limit dragging to a user-defined plane. Configure using:
- Point on the plane
- Normal vector of the plane
- Maximum allowed camera distance

### dragType
**Type**: PropTypes.oneOf(["FixedDistance", "FixedToWorld"])

Controls dragging behavior when `onDrag` is specified:

| Value | Description |
|-------|-------------|
| FixedDistance | Dragging limited to fixed radius around user, from grab point |
| FixedDistanceOrigin | Dragging limited to fixed radius around user, from node's world position |
| FixedToWorld | Dragging based on real world object intersection (AR only) |
| FixedToPlane | Dragging limited to fixed plane defined by `dragPlane` prop |

Default: "FixedDistance"

### Event Handlers

#### onClick
**Type**: React.PropTypes.func

Triggered when user clicks scene and no other object captures the click. Useful for 360° photos/videos.

#### onClickState
**Type**: React.PropTypes.func

Tracks click state progression:

| State | Description |
|-------|-------------|
| 1 | Click Down: Initial click/touch |
| 2 | Click Up: Release click/touch |
| 3 | Clicked: Complete click sequence |

```javascript
_onClickState(stateValue, position, source) {
  if(stateValue == 1) {
    // Click Down
  } else if(stateValue == 2) {
    // Click Up
  } else if(stateValue == 3) {
    // Clicked
  }
}
```

#### onDrag
**Type**: React.PropTypes.func

Provides current 3D position while dragging:

```javascript
_onDrag(dragToPos, source) {
  // dragtoPos[0]: x position
  // dragtoPos[1]: y position  
  // dragtoPos[2]: z position
}
```

### Physics & Sound

#### physicsWorld
**Type**: PropTypes.shape
```typescript
{
  gravity: PropTypes.arrayOf(PropTypes.number).isRequired,
  drawBounds: PropTypes.bool
}
```

Manages physics bodies and properties:
- `gravity`: Vector acceleration applied to physics objects [x,y,z]
- `drawBounds`: If true, renders physics body mesh boundaries

#### soundRoom
**Type**: PropTypes.shape

Configures acoustic room properties:

```javascript
soundRoom={{
  size: [2,2,2],
  wallMaterial: "acoustic_ceiling_tiles",
  ceilingMaterial: "glass_thin",
  floorMaterial: "concrete_block_coarse"
}}
```

Supported materials include:
- acoustic_ceiling_tiles
- brick_bare/painted
- concrete_block_coarse/painted
- glass_thin/thick
- marble
- wood_panel/ceiling
(and many more)

### Camera Methods

#### getCameraOrientationAsync()
Returns current camera orientation including:
- position [x,y,z]
- rotation [x,y,z] 
- forward vector
- up vector

#### findCollisionsWithRayAsync()
Detects collisions between physics bodies and a ray:
```typescript
async findCollisionsWithRayAsync(
  from: number[], 
  to: number[],
  closest: boolean,
  viroTag: string
): Promise<boolean>
```

### Additional Props

- **ignoreEventHandling**: (bool) When true, allows events to pass through to objects behind
- **visible**: (bool) Controls component visibility
- **transformBehaviors**: (string[]) Transformation behavior modifiers
- **rotation**: (number[]) [x,y,z] rotation values
- **style**: (stylePropType) Style properties
- **width**: (number) Component width

## Methods

### setNativeProps(nativeProps: object)
Directly updates native component properties without re-rendering. See React Native's [Direct Manipulation](https://facebook.github.io/react-native/docs/direct-manipulation.html) docs.

```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```

---

# ViroNode

A generic, empty 3D node in the scene graph. Transforms (position, rotation, scale) set on this node apply to all children. 

## Basic Example
```javascript
<ViroNode
    position={[2.0, 5.0, -2.0]}
    rotation={[0, 45, 45]}
    scale={[2.0, 2.0, 2.0]}
/>
```

## Props

### Transform Props

#### position
**Type**: `[number, number, number]`
Cartesian position in 3D space [x, y, z]

#### rotation  
**Type**: `[number, number, number]`
Euler angles [x, y, z] in degrees for local axis rotation

#### scale
**Type**: `[number, number, number]`  
Scale factors [x, y, z]. Values:
- 1: Current size
- <1: Smaller 
- >1: Larger

#### Transform Behavior Props

##### transformBehaviors
**Type**: `string[]`
Transform constraints affecting the node:
- "billboard": Face user on x,y,z
- "billboardX": Face user on x axis
- "billboardY": Face user on y axis 
- "billboardZ": Face user on z axis

##### rotationPivot
**Type**: `[number, number, number]`
Point [x,y,z] around which rotation is applied

##### scalePivot
**Type**: `[number, number, number]`
Point [x,y,z] from which scaling is applied

### Interaction Props

#### dragType
**Type**: `"FixedDistance" | "FixedToWorld" | "FixedDistanceOrigin" | "FixedToPlane"`
Controls dragging behavior:
- **FixedDistance**: Fixed radius from user's grab point (default)
- **FixedDistanceOrigin**: Fixed radius from node's world position
- **FixedToWorld**: Intersection with real objects (AR only)
- **FixedToPlane**: Movement constrained to defined plane

#### dragPlane
**Type**: ViroDragPlane
Plane constraints for FixedToPlane drag type:
- Point on plane
- Normal vector
- Maximum camera distance

#### highAccuracyEvents
**Type**: `boolean`
- `true`: Use geometry for event detection
- `false`: Use bounding box (default)

### Event Handlers

#### onClick
```typescript
(position: number[], source: string) => void
```
Called when node is clicked

#### onClickState
```typescript
(stateValue: number, position: number[], source: string) => void
```
Click state progression:
1. Click Down 
2. Click Up
3. Clicked

#### onDrag
```typescript
(dragToPos: number[], source: string) => void
```
Called during dragging with current 3D position
```javascript
_onDrag = (dragToPos, source) => {    
  // dragtoPos[0]: x position    
  // dragtoPos[1]: y position    
  // dragtoPos[2]: z position
}
```

#### Touch/Gesture Events
- **onPinch**: Pinch gesture (AR only)
- **onRotate**: Rotation gesture (AR only) 
- **onTouch**: Touch interactions
- **onSwipe**: Swipe gestures
- **onScroll**: Scroll actions
- **onHover**: Hover enter/exit

### Visual Props

#### opacity
**Type**: `number`  
0 (transparent) to 1 (opaque)

#### visible
**Type**: `boolean`
Show/hide the node

#### renderingOrder
**Type**: `number`
Rendering priority relative to other nodes. Higher numbers render later.

### Physics Props

#### physicsBody
**Type**: Physics Body
Physics simulation configuration

#### viroTag
**Type**: `string`  
Tag for physics collision identification

### Methods

#### Transforms
```typescript
async getBoundingBoxAsync(): Promise<BoundingBox>
async getTransformAsync(): Promise<Transform>
```

#### Physics
```typescript
applyImpulse(force: number[], position: number[]): void
applyTorqueImpulse(torque: number[], position: number[]): void
setVelocity(velocity: number[]): void
```

#### Direct Manipulation
```typescript
setNativeProps(props: object): void
```
Update native props without re-rendering:
```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```

### Types

```typescript
interface BoundingBox {
  minX: number;
  maxX: number;
  minY: number;
  maxY: number;
  minZ: number;
  maxZ: number;
}

interface Transform {
  position: [number, number, number];
  rotation: [number, number, number];
  scale: [number, number, number];
}
```

---

# ViroARScene

The `ViroARScene` component allows developers to logically group AR experiences and components, which can be managed through the ViroARSceneNavigator. This component provides controls and interactions with the AR subsystem.

## Basic Usage

```javascript
<ViroARScene onTrackingUpdated={this._trackingUpdated}>
  <ViroARPlane>
    <ViroBox position={[0, .5, 0]} />
  </ViroARPlane>
</ViroARScene>
```

## Props

### Core Properties

#### anchorDetectionTypes
**Type**: string | string[]
- Determines what types of anchors to detect
- Supported values:
  - "None"
  - "PlanesHorizontal"
  - "PlanesVertical"

#### displayPointCloud
**Type**: boolean | PointCloudOptions
```typescript
interface PointCloudOptions {
  imageSource: ImageSourcePropType;    // Image for each point
  imageScale: [number, number, number]; // Scale of point image [.01,.01,.01]
  maxPoints: number;                    // Max points per frame
}
```

Example:
```javascript
<ViroARScene 
  displayPointCloud={{
    imageSource: require("./res/pointCloudPoint.png"),
    imageScale: [.02,.02,.02],
    maxPoints: 100 
  }} />
```

### Tracking & Camera Props

#### onTrackingUpdated
**Type**: Function
Called when device tracking state changes. States include:
- TRACKING_UNAVAILABLE (1): Position unknown
- TRACKING_LIMITED (2): Position may be inaccurate
- TRACKING_NORMAL (3): Optimal tracking

iOS provides additional diagnostic info via "reason" parameter:
- TRACKING_REASON_NONE (1): Not limited
- TRACKING_REASON_EXCESSIVE_MOTION (2): Moving too fast
- TRACKING_REASON_INSUFFICIENT_FEATURES (3): Not enough visual features

```javascript
const handleTrackingUpdated = (state, reason) => {
  if (state == ViroConstants.TRACKING_NORMAL) {
    // Show AR experience
  } else if (state == ViroConstants.TRACKING_NONE) {
    // Prompt movement
  }  
}
```

#### onCameraTransformUpdate
**Type**: (updateObj) => void
Callback when camera changes (max once per frame). Returns:
```typescript
{
  cameraTransform: {    
    position: [posX, posY, posZ],
    rotation: [rotX, rotY, rotZ],
    forward: [forwardX, forwardY, forwardZ],    
    up: [upX, upY, upZ]  
  }
}
```

### Anchor Related Props

#### onAnchorFound
**Type**: (anchor) => void
- Called when AR system finds an anchor
- See Anchor type definition below

#### onAnchorUpdated  
**Type**: (anchor) => void
- Called when anchor properties change
- See Anchor type definition below

#### onAnchorRemoved
**Type**: () => void
- Called when anchor no longer exists
- See Anchor type definition below

### Physics Properties

#### physicsWorld
**Type**: PhysicsWorld
```typescript
interface PhysicsWorld {
  gravity: number[];  // Default: [0, -9.81, 0]
  drawBounds: boolean;
}
```

### Interaction Props

#### dragType
**Type**: "FixedDistance" | "FixedToWorld" | "FixedDistanceOrigin" | "FixedToPlane"
Controls dragging behavior:
- FixedDistance: Fixed radius from user, from grab point
- FixedDistanceOrigin: Fixed radius from user, from node position
- FixedToWorld: Based on real world intersections (AR only)
- FixedToPlane: Limited to specified plane

#### dragPlane 
**Type**: ViroDragPlane
Configures plane-based dragging constraints

## Methods

### Hit Testing
```typescript
async performARHitTestWithRay(ray: number[]): Promise<ARHitTestResult[]>
async performARHitTestWithPosition(position: number[]): Promise<ARHitTestResult[]>
async performARHitTestWithPoint(x: number, y: number): Promise<ARHitTestResult[]>
```

### Camera & Collisions
```typescript
async getCameraOrientationAsync(): Promise<CameraOrientation>
async findCollisionsWithRayAsync(
  from: number[], 
  to: number[], 
  closest: boolean,
  viroTag: string
): Promise<boolean>
```

## Types

### ARHitTestResult
```typescript
interface ARHitTestResult {
  type: "ExistingPlaneUsingExtent" | "ExistingPlane" | 
        "EstimatedHorizontalPlane" | "FeaturePoint";
  transform: {
    position: number[];
    rotation: number[];
    scale: number[];
  }
}
```

### Anchor
Properties passed to anchor callbacks:

| Property | Type | Description |
|----------|------|-------------|
| anchorId | string | Unique anchor identifier |
| type | string | Type of anchor |
| position | number[] | World coordinates |
| rotation | number[] | Rotation in degrees |
| center | number[] | Plane center (ViroARPlane only) |
| width | number | Plane width (ViroARPlane only) |
| height | number | Plane height (ViroARPlane only) |
| vertices | number[][] | Boundary points (ViroARPlane only) |

### Post-Process Effects

Available effects:
- grayscale: Black and white
- sepia: Reddish-brown tint
- sincity: B&W with saturated reds
- baralleldistortion: Fish-eye center
- pincushiondistortion: Center pinch
- thermalvision: Heat-map style
- crosshatch: Line pattern
- pixelated: Pixelization

---

# ViroARSceneNavigator
ViroARSceneNavigator is the entry point for AR applications with Viro.

## Properties

### initialScene (Required)
- **Type**: { scene: JSX.Element }
- **Description**: The initial AR scene to display for your application on application start
- **Note**: ViroARSceneNavigator only accepts ViroARScene

### autofocus
- **Type**: boolean
- **Description**: Whether or not to enable autofocus for this AR session
- **Default**: 
  - iOS: true
  - Android: false (optimized for ARCore 1.4.0)

### bloomEnabled
- **Type**: boolean
- **Description**: Enable/disable bloom effects. Adds soft glow to bright areas, simulating human eye perception
- **Usage**: Requires setting object's material `bloomThreshold` property

### hdrEnabled
- **Type**: boolean
- **Description**: Enables HDR rendering with:
  - Deeper color space
  - Floating point texture rendering
  - Tone-mapping algorithm
  - Required for Bloom and PBR features
  - Note: Not supported on all devices

### numberOfTrackedImages
- **Type**: number
- **Platform**: iOS 12+/ARKit 2.0+ only
- **Description**: Maximum number of images to track concurrently
- **Notes**:
  - First N visible images will be tracked
  - Lower numbers = better performance
  - Untracked markers become tracked when tracked markers leave view

### pbrEnabled
- **Type**: boolean
- **Description**: Enable/disable physically-based rendering (PBR)
- **Features**:
  - More realistic lighting
  - Intuitive artist workflow
  - Requires materials using `physicallyBased` lighting model

### shadowsEnabled
- **Type**: boolean
- **Description**: Enable/disable dynamic shadow rendering

### videoQuality
- **Type**: "High" | "Low"
- **Platform**: iOS 11.3+/ARKit 1.5+ only
- **Default**: "High"
- **Description**: Sets video quality when device supports multiple quality levels

### viroAppProps
- **Type**: object
- **Description**: Properties object for the Viro app
- **Access**: Via `this.props.sceneNavigator.viroAppProps` in ViroARScene objects

### worldAlignment
- **Type**: "Gravity" | "GravityAndHeading" | "Camera"
- **Platform**: iOS only
- **Description**: Determines initial world coordinate system alignment:
  - **Gravity**: Origin at device start, X-Z plane perpendicular to gravity
  - **GravityAndHeading**: Origin at device start, X/Z axes aligned to latitude/longitude
  - **Camera**: Coordinate system locked to camera orientation

## Methods

### Navigation Methods

#### push({ scene: ViroARScene, passProps: Object })
Pushes new scene onto stack and displays it
- **Parameters**:
  - scene: Scene to display
  - passProps: Props to pass to scene

#### pop()
Removes top scene from stack, returning to previous scene

#### pop(n: number) 
Removes N scenes from stack at once

#### jump({ scene: ViroARScene, passProps: Object })
Moves existing scene to top of stack or pushes new scene if not found
- **Best for**: Frequently switching between scenes
- **Parameters**:
  - scene: Scene to display
  - passProps: Props to pass to scene

#### replace({ scene: ViroARScene, passProps: Object })
Replaces current scene while preserving stack
- **Parameters**:
  - scene: Replacement scene
  - passProps: Props to pass to scene

### AR Session Methods

#### resetARSession(resetTracking: boolean, removeAnchors: boolean)
**Platform**: iOS Only
- **Parameters**:
  - resetTracking: Realigns scene with camera
  - removeAnchors: Removes all anchors (`onAnchorRemoved` called)
- **Best Practice**: Set both parameters true together

#### setWorldOrigin(offset: Object)
**Platform**: iOS 11.3+/ARKit 1.5+ Only
- **Parameters**:
  - offset: Object containing position and/or rotation
- **Behavior**: Transforms accumulate with multiple calls
- **Example**:
```typescript
this.arSceneNavigator.setWorldOrigin({
  position: [x, y, z],
  rotation: [x, y, z]
})
```

### Recording Methods

#### startVideoRecording(fileName: string, saveToCameraRoll: bool, onError: func)
Captures Viro components and audio
- **Requirements**: Camera/microphone permissions
- **Parameters**:
  - fileName: Save location (auto-adds extension)
  - saveToCameraRoll: Save to Photos (requires permission)
  - onError: Error handling callback

#### async stopVideoRecording()
- **Returns**: Object with:
  - success: boolean
  - url: File path
  - errorCode: Error number if failed

#### async takeScreenshot(fileName: string, saveToCameraRoll: bool)
Captures Viro components only
- **Parameters**:
  - fileName: Save location (auto-adds extension)
  - saveToCameraRoll: Save to Photos (requires permission)
- **Returns**: Object with:
  - success: boolean
  - url: File path
  - errorCode: Error number if failed

### Projection Methods

#### async project(point: array)
Converts 3D position to screen coordinates
- **Parameters**:
  - point: [x, y, z] world position
- **Returns**: Object with:
  - screenPosition: [x, y] screen coordinates
- **Example**:
```typescript
this.props.arSceneNavigator.project(projectPoint).then((returnPos) => {
  this.setState({
    screenX: returnPos["screenPosition"][0],
    screenY: returnPos["screenPosition"][1]
  });
});
```

#### async unproject(point: array)
Converts screen position to 3D coordinates
- **Parameters**:
  - point: [screenX, screenY, depth] where depth is 0-1
    - 0: Near clipping plane
    - 1: Far clipping plane
- **Returns**: Object with:
  - position: [x, y, z] world coordinates
- **Example**:
```typescript
this.props.arSceneNavigator.unproject(unprojectPoint).then((returnDict) => {
  this.setState({ unprojectedPoint: returnDict["position"] });
});
```

---

# ViroARPlane

A ViroARPlane is a component that allows developers to place components relative to a plane discovered by the AR system. The process of attaching a ViroARPlane to a plane discovered by the AR system is discussed in the next section.

## Anchoring

Anchoring is our term for attaching virtual content to detected real-world points/features (anchors). For ViroARPlanes, we support two types: manual and automatic anchoring.

### Automatic Anchoring

To enable automatic anchoring, the developer provides a `minHeight` and `minWidth` to the `ViroARPlane` and adds the content they desire within the `ViroARPlane` component. When the AR system finds a plane that matches the given dimensions, the `ViroARPlane` will be anchored to the real world plane and the child components will be made visible. Any updates to the real-world plane will be given to the developer through the `onAnchorFound`, `onAnchorUpdated`, and `onAnchorRemoved` callback functions.

###### Example use:

```javascript
<ViroARScene
  onAnchorFound={() => console.log('onAnchorFound')}
  onAnchorUpdated={() => console.log('onAnchorUpdated')}
  onAnchorRemoved={() => console.log('onAnchorRemoved')}>
  <ViroARPlane minHeight={0.1} minWidth={0.1} alignment={'Horizontal'}>
    <ViroBox position={[0, 0, 0]} scale={[0.1, 0.1, 0.1]} />
  </ViroARPlane>
</ViroARScene>
```

### Manual Anchoring

In manual anchoring, rather than having the platform determine which real-world feature/anchor the `ViroARPlane` is attached to, the developer is required to listen for all the anchors and "choose" the anchor they want the `ViroARPlane` to attach to via the `anchorId` property. To listen for all the anchors, the user should add `onAnchorFound`, `onAnchorUpdated` and `onAnchorRemoved` listeners to their ViroARScene component which will receive all anchors that the AR platform discovers.

###### Example use:

```javascript
<ViroARScene
  onAnchorFound={(foundAnchor) => console.log('onAnchorFound', foundAnchor)}
  onAnchorUpdated={() => console.log('onAnchorUpdated')}
  onAnchorRemoved={() => console.log('onAnchorRemoved')}>
  <ViroARPlane anchorId={foundAnchor.anchorId}>
    <ViroBox position={[0, 0, 0]} scale={[0.1, 0.1, 0.1]} />
  </ViroARPlane>
</ViroARScene>
```

## Props

### alignment

| Type | Description |
|------|-------------|
| "Horizontal" \| "HorizontalUpward" \| "HorizontalDownward" \| "Vertical" | Specifies the desired alignment of a plane that this component will "anchor" to.The default value is "Horizontal". Don't forget to set the `anchorDetectionTypes` prop of ViroARScene to tell the AR Session what type of planes to discover. **Note: "HorizontalUpward" and "HorizontalDownward" are only supported in Android.** **For Automatic Anchoring**, see [Anchoring](#anchoring) |

### anchorId

| Type | Description |
|------|-------------|
| string | **For Manual Anchoring**, see [Anchoring](#anchoring). The ID of the anchor that the platform should anchor this `ViroARPlane` to. If no Anchor has the specified anchorId, then plane will not be visible until an `Anchor` appears with the same ID. |

### dragPlane

| Type | Description |
|------|-------------|
| ViroDragPlane | When a drag type of "FixedToPlane" is given, dragging is limited to a user defined plane. The dragging behavior is then configured by this property (specified by a point on the plane and its normal vector). You can also limit the maximum distance the dragged object is allowed to travel away from the camera/controller (useful for situations where the user can drag an object towards infinity). |

### dragType

| Type | Description |
|------|-------------|
| "FixedDistance" \| "FixedToWorld" \| "FixedDistanceOrigin" \| "FixedToPlane" | Determines the behavior of drag if **onDrag** is specified. **"FixedDistance":** Dragging is limited to a fixed radius around the user, dragged from the point at which the user has grabbed the geometry containing this draggable node. **"FixedDistanceOrigin":** Dragging is limited to a fixed radius around the user, dragged from the point of this node's position in world space. **"FixedToWorld":** Dragging is based on intersection with real world objects. **Available only in AR**. **"FixedToPlane":** Dragging is limited to a fixed plane around the user. The configuration of this plane is defined by the **dragPlane** property. The default value is "FixedDistance". |

### ignoreEventHandling

| Type | Description |
|------|-------------|
| boolean | When set to true, this control will ignore events and not prevent controls behind it from receiving event callbacks.The default value is false. |

### minHeight

| Type | Description |
|------|-------------|
| number | **For Automatic Anchoring**, see [Anchoring](#anchoring) Specifies the minimum height, in meters, of a plane that this component will "anchor" to. The default value is 0. |

### minWidth

| Type | Description |
|------|-------------|
| number | **For Automatic Anchoring**, see [Anchoring](#anchoring) Specifies the minimum width, in meters, of a plane that this component will "anchor" to. The default value is 0. |

### onAnchorFound

| Type | Description |
|------|-------------|
| (anchor) => void | Called when this component is anchored to a plane that is at least `minHeight` by `minWidth` large. This is when the component is made visible. anchor: see Anchor |

### onAnchorRemoved

| Type | Description |
|------|-------------|
| Function | Called when this component is detached from a plane and is no longer visible. |

### onAnchorUpdated

| Type | Description |
|------|-------------|
| (anchor) => void | Called when the plane to which this component is anchored is updated. anchor: see Anchor |

### onClick

See [ViroNode onClick](#onclick).

### onClickState

See [ViroNode onClickState](#onclickstate).

### onCollision

See [ViroNode onCollision](#oncollision).

### onDrag

See [ViroNode onDrag](#ondrag).

### onFuse

See [ViroNode onFuse](#onfuse).

### onHover

See [ViroNode onHover](#onhover).

### onPinch

See [ViroNode onPinch](#onpinch).

### onRotate

See [ViroNode onRotate](#onrotate).

### onScroll

See [ViroNode onScroll](#onscroll).

### onSwipe

See [ViroNode onSwipe](#onswipe).

### onTouch

See [ViroNode onTouch](#ontouch).

### onTransformUpdate

See [ViroNode onTransformUpdate](#ontransformupdate).

### opacity

| Type | Description |
|------|-------------|
| number | A number from 0 to 1 that specifies the opacity of the container. A value of 1 translates into a fully opaque node while 0 represents full transparency. |

### pauseUpdates

| Type | Description |
|------|-------------|
| boolean | True/False to stop the automatic positioning/rotation of children components of a `ViroARPlane`. This does not stop `onAnchorUpdated` from being called. |

### viroTag

| Type | Description |
|------|-------------|
| string | A tag given to other components when their physics body collides with this component's physics body. Refer to [physics](#physics) for more information. |

### visible

| Type | Description |
|------|-------------|
| boolean | False if the container should be hidden. By default the container is visible and this value is true. |

### renderingOrder

| Type | Description |
|------|-------------|
| number | This determines the order in which this Node is rendered relative to other Nodes. Nodes with greater rendering orders are rendered last. The default rendering order is zero. For example, setting a Node's rendering order to -1 will cause the Node to be rendered before all Nodes with rendering orders greater than or equal to 0. |

### rotation

| Type | Description |
|------|-------------|
| [number, number, number] | The rotation of the component around it's local axis specified as Euler angles [x, y, z]. Units for each angle are specified in degrees. |

### scale

| Type | Description |
|------|-------------|
| | PropTypes.arrayOf(PropTypes.number)Put the PropType Description here. |

### transformBehaviors

| Type | Description |
|------|-------------|
| string[] | An array of transform constraints that affect the transform of the object. For example, putting the value "billboard" will ensure the box is facing the user as the user rotates their head on any axis. This is useful for icons or text where you'd like the box to always face the user at a particular rotation. Allowed values(values are case sensitive): **"billboard":** Billboard object on x,y,z axis **"billboardX":** Billboard object on the x axis **"billboardY":** Billboard object on the y axis **"billboardZ":** Billboard object on the z axis |

## Methods

### setNativeProps(nativeProps)

A wrapper function around the native component's setNativeProps which allow users to set values on the native component without changing state/setting props and re-rendering. Refer to the React Native documentation on [Direct Manipulation](https://facebook.github.io/react-native/docs/direct-manipulation) for more information.

| Parameter | Type | Description |
|-----------|------|-------------|
| nativeProps | object | an object where the keys are the properties to set and the values are the values to set |

```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```

---

# ViroARPlaneSelector

The `ViroARPlaneSelector` is a composite component written entirely in React Native leveraging ViroARPlane that provides developers with an easy way to have the end user select a plane they want to use. This component surfaces all the same properties as ViroARPlane but rather than attaching the developer's components to the first surface found, it presents end users with white, transparent surfaces which they can select to indicate where they want to have their AR experience.

###### Example use:

```javascript
<ViroARPlaneSelector
    minHeight={.5}
    minWidth={.5}
    onPlaneSelected={...}
>
    <ViroBox
        position={[0, .25, 0]}
        scale={[.5, .5, .5]}
    />
    ...
</ViroARPlaneSelector>
```

## Props

### alignment

| Type | Description |
|------|-------------|
| "Horizontal" \|"HorizontalUpward" \| "HorizontalDownward" \| "Vertical" | Don't forget to set the `anchorDetectionTypes` prop of ViroARScene to tell the AR Session what type of planes to discover.<br><br>**Note: "HorizontalUpward" and "HorizontalDownward" are only supported in Android.**<br><br>Specifies the desired alignment of a plane that this component will "anchor" to.<br><br>The default value is "Horizontal". |

### dragPlane

| Type | Description |
|------|-------------|
| ViroDragPlane | When a drag type of "FixedToPlane" is given, dragging is limited to a user defined plane. The dragging behavior is then configured by this property (specified by a point on the plane and its normal vector). You can also limit the maximum distance the dragged object is allowed to travel away from the camera/controller (useful for situations where the user can drag an object towards infinity). |

### dragType

| Type | Description |
|------|-------------|
| "FixedDistance" \| "FixedToWorld" \| "FixedDistanceOrigin" \| "FixedToPlane" | Determines the behavior of drag if **onDrag** is specified. The default value is "FixedDistance".<br><br>**FixedDistance:** Dragging is limited to a fixed radius around the user, dragged from the point at which the user has grabbed the geometry containing this draggable node<br><br>**FixedDistanceOrigin:** Dragging is limited to a fixed radius around the user, dragged from the point of this node's position in world space.<br><br>**FixedToWorld:** Dragging is based on intersection with real world objects. **Available only in AR**<br><br>**FixedToPlane:** Dragging is limited to a fixed plane around the user. The configuration of this plane is defined by the **dragPlane** property. |

### ignoreEventHandling

| Type | Description |
|------|-------------|
| boolean | When set to true, this control will ignore events and not prevent controls behind it from receiving event callbacks.<br><br>The default value is false. |

### maxPlanes

| Type | Description |
|------|-------------|
| number | The number of planes to present to the end user for them to select. If the AR system discovers more planes than this number, then it will only display the first `maxPlanes` number of planes to the end user. The default value is 15. |

### minHeight

| Type | Description |
|------|-------------|
| number | Specifies the minimum height, in meters, of a plane that this component will "anchor" to.<br><br>The default value is 0. |

### minWidth

| Type | Description |
|------|-------------|
| number | Specifies the minimum width, in meters, of a plane that this component will "anchor" to.<br><br>The default value is 0. |

### onAnchorFound

| Type | Description |
|------|-------------|
| (anchor) => void | **Developer should instead listen to `onPlaneSelected`**<br><br>Called when this component is anchored to a plane that is at least `minHeight` by `minWidth` large. This is when the component is made visible.<br><br>See [Anchor](#anchor) for more. |

### onAnchorRemoved

| Type | Description |
|------|-------------|
| () => void | Called when this component is detached from a plane and is no longer visible. |

### onAnchorUpdated

| Type | Description |
|------|-------------|
| (anchor) => void | Called when the plane to which this component is anchored is updated.<br><br>See [Anchor](#anchor) for more. |

### onClick
See [ViroNode onClick](docs/vironode#onclick).

### onClickState
See [ViroNode onClickState](docs/vironode#onclickstate).

### onCollision
See [ViroNode onCollision](docs/vironode#oncollision).

### onDrag
See [ViroNode onDrag](docs/vironode#ondrag).

### onFuse
See [ViroNode onFuse](docs/vironode#onfuse).

### onHover
See [ViroNode onHover](docs/vironode#onhover).

### onPinch
See [ViroNode onPinch](docs/vironode#onpinch).

### onPlaneSelected

| Type | Description |
|------|-------------|
| () => void | This function is called when the end user has selected a plane to use. |

### onRotate
See [ViroNode onRotate](docs/vironode#onrotate).

### onScroll
See [ViroNode onScroll](docs/vironode#onscroll).

### onSwipe
See [ViroNode onSwipe](docs/vironode#onswipe).

### onTouch
See [ViroNode onTouch](docs/vironode#ontouch).

### opacity

| Type | Description |
|------|-------------|
| number | A number from 0 to 1 that specifies the opacity of the container. A value of 1 translates into a fully opaque node while 0 represents full transparency. |

### viroTag

| Type | Description |
|------|-------------|
| string | A tag given to other components when their physics body collides with this component's physics body. Refer to [physics](docs/physics) for more information. |

### visible

| Type | Description |
|------|-------------|
| boolean | False if the container should be hidden. By default the container is visible and this value is true. |

### renderingOrder

| Type | Description |
|------|-------------|
| number | This determines the order in which this Node is rendered relative to other Nodes. Nodes with greater rendering orders are rendered last. The default rendering order is zero. For example, setting a Node's rendering order to -1 will cause the Node to be rendered before all Nodes with rendering orders greater than or equal to 0. |

### rotation

| Type | Description |
|------|-------------|
| [number, number, number] | The rotation of the component around it's local axis specified as Euler angles [x, y, z]. Units for each angle are specified in degrees. |

### scale

| Type | Description |
|------|-------------|
| [number, number, number] | The scale of the component in 3D space, specified as [x,y,z]. A scale of 1 represents the current size of the component. A scale value of < 1 will make the component proportionally smaller while a value >1 will make the component proportionally bigger along the specified axis. |

### transformBehaviors

| Type | Description |
|------|-------------|
| string[] | An array of transform constraints that affect the transform of the object. For example, putting the value "billboard" will ensure the box is facing the user as the user rotates their head on any axis. This is useful for icons or text where you'd like the box to always face the user at a particular rotation. Allowed values(values are case sensitive):<br><br>**"billboard":** Billboard object on x,y,z axis **"billboardX":** Billboard object on the x axis<br>**"billboardY":** Billboard object on the y axis<br>**"billboardZ":** Billboard object on the z axis |

## Anchor

This is the object given to the developer through the `onAnchorFound` and `onAnchorUpdated` callback functions.

| Key | Type | Value |
|-----|------|-------|
| position | [number, number, number] | Position of the attached plane in world coordinates. |
| rotation | [number, number, number] | Rotation of the attached plane in world coordinates. |
| center | [number, number, number] | Center of the underlying plane relative to the plane's position. |
| width | number | Current width of the attached plane |
| height | number | Current height of the attached plane |

## Methods

### reset()

This function resets the `ARPlaneSelector` back to the "selection" state which presents the end user with all planes that have been found by the AR system (up to `maxPlanes` number of planes).

---

# ViroImage

A component used to display 2D images.

## Example use:

```javascript
<ViroImage
    height={2}
    width={2}
    placeholderSource={require("./res/local_spinner.jpg")}
    source={{ uri: "https://my_s3_image.jpg" }}
/>
```

## Props

## Required Props

**source** - **PropTypes.oneOfType( [PropTypes.shape( {uri:  PropTypes.string} ), PropTypes.number] )**
The image source, a remote URL or a local file resource. PNG and JPG images accepted.
To invoke with remote url: `{uri:"http://example.org/myimage.png"}`
To invoke with local source: `require('image.html');`

### animation

**PropTypes.shape({      name: PropTypes.string,      delay: PropTypes.number,      loop: PropTypes.bool,      onStart: PropTypes.func,      onFinish: PropTypes.func,      run: PropTypes.bool,    })**

A collection of parameters that determine if this component should animate. For more information on animated components please see our Animation Guide.

### dragPlane

**PropTypes.shape({  planePoint:PropTypes.arrayOf(PropTypes.number),  planeNormal:PropTypes.arrayOf(PropTypes.number),   maxDistance:PropTypes.number})**

When a drag type of "FixedToPlane" is given, dragging is limited to a user defined plane. The dragging behavior is then configured by this property (specified by a point on the plane and its normal vector). You can also limit the maximum distance the dragged object is allowed to travel away from the camera/controller.

### dragType

**PropTypes.oneOf(["FixedDistance", "FixedToWorld"])**

Determines the behavior of drag if **onDrag** is specified.

| Value | Description |
|-------|-------------|
| FixedDistance | Dragging is limited to a fixed radius around the user, dragged from the point at which the user has grabbed the geometry containing this draggable node |
| FixedDistanceOrigin | Dragging is limited to a fixed radius around the user, dragged from the point of this node's position in world space. |
| FixedToWorld | Dragging is based on intersection with real world objects. **Available only in AR** |
| FixedToPlane | Dragging is limited to a fixed plane around the user. The configuration of this plane is defined by the **dragPlane** property. |

The default value is "FixedDistance".

### format

**PropTypes.oneOf(['RGBA8', 'RGBA4', 'RGB565'])**

Image texture formats for storage on the GPU.

| Format | Description |
|--------|-------------|
| RGBA8 | Each pixel is described with 32-bits, using eight bits per channel |
| RGBA4 | Each pixel is described with 16 bits, using four bits per channel |
| RGB565 | Formats the picture into 16 bit color values without alpha |

### height

**PropTypes.number**

The height of the image in 3D space. Default value is 1.

### highAccuracyEvents

**PropTypes.bool**

True if events should use the geometry of the object to determine if the user is interacting with this object. If false, the object's axis-aligned bounding box will be used instead. Enabling this is more accurate but takes more processing power, so it is set to false by default.

### ignoreEventHandling

**PropTypes.bool**

When set to true, this control will ignore events and not prevent controls behind it from receiving event callbacks.
The default value is false.

### imageClipMode

**PropTypes.oneOf(['None', 'ClipToBounds'])**

Defaults to clipToBounds. If `ClipToBounds` is set, image is cropped if it overshoots with `ScaleToFill`. Setting to `None` does not crop image if it overshoots bounds.

### lightReceivingBitMask

**PropTypes.number**

A bit mask that is bitwise and-ed (&) with each light's influenceBitMask. If the result is > 0, then the light will illuminate this object. For more information please see the Lighting and Materials Guide.

### materials

**PropTypes.oneOfType([PropTypes.arrayOf(PropTypes.string),PropTypes.string])**

An array of strings that each represent a material that was created via ViroMaterials.createMaterials(). ViroImage takes 1 material. The diffuseTexture of the material will be the image shown unless a source prop is specified. The other material properties will preserve as the source prop changes.

### mipmap

**PropTypes.bool**

If true, we dynamically generate mipmaps for the source of this given image.

### onClick

**React.PropTypes.func**

Called when an object has been clicked.
Example code:
```javascript
_onClick(position, source) {
    // user has clicked the object
}
```
The position parameter represents the position in world coordinates on the object where the click occurred. For the mapping of sources to controller inputs, see the Events section.

### onClickState

**React.PropTypes.func**

Called for each click state an object goes through as it is clicked. Supported click states and their values are the following:

| State Value | Description |
|-------------|-------------|
| 1 | Click Down: Triggered when the user has performed a click down action while hovering on this control. |
| 2 | Click Up: Triggered when the user has performed a click up action while hovering on this control. |
| 3 | Clicked: Triggered when the user has performed both a click down and click up action on this control sequentially, thereby having "Clicked" the object. |

Example code:
```javascript
_onClickState(stateValue, position, source) {
    if(stateValue == 1) {
        // Click Down
    } else if(stateValue == 2) {
        // Click Up
    } else if(stateValue == 3) {
        // Clicked
    }
}
```
For the mapping of sources to controller inputs, see the Events section.

### onCollision

**React.PropTypes.func**

Called when this component's physics body collides with another component's physics body. Also invoked by ViroScene/ViroARScene's `findCollisions...` functions.

| Return Value | Description |
|--------------|-------------|
| viroTag | the given viroTag (string) of the collided component |
| collidedPoint | an array of numbers representing the position, in world coordinates, of the point of collision |
| collidedNormal | an array representing the normal of the collision in world coordinates. |

### onDrag

**React.PropTypes.func**

Called when the view is currently being dragged. The dragToPos parameter provides the current 3D location of the dragged object. Example code:
```javascript
_onDrag(dragToPos, source) {
    // dragtoPos[0]: x position
    // dragtoPos[1]: y position
    // dragtoPos[2]: z position
}
```
For the mapping of sources to controller inputs, see the Events section. Unsupported VR Platforms: Cardboard iOS

### onFuse

**PropTypes.oneOfType**
```javascript
PropTypes.oneOfType([
    React.PropTypes.shape({
        callback: React.PropTypes.func.isRequired,
        timeToFuse: PropTypes.number
    }),
    React.PropTypes.func,
])
```
As shown above, onFuse takes one of the types - either a callback, or a dictionary with a callback and duration. It is called after the user hovers onto and remains hovered on the control for a certain duration of time, as indicated in timeToFuse that represents the duration of time in milliseconds. While hovering, the reticle will display a count down animation while fusing towards timeToFuse.
Note that timeToFuse defaults to 2000ms.
For example:
```javascript
_onFuse(source) {
    // User has hovered over object for timeToFuse milliseconds
}
```
For the mapping of sources to controller inputs, see the Events section.

### onHover

**React.PropTypes.func**

Called when the user hovers on or off the control.
For example:
```javascript
_onHover(isHovering, position, source) {
    if(isHovering) {
        // user is hovering over the box
    } else {
        // user is no longer hovering over the box
    }
}
```
The position parameter represents the position in world coordinates on the object where the click occurred. For the mapping of sources to controller inputs, see the Events section.

### onError

**React.PropTypes.func**

Callback invoked when the Image fails to load. The error message is contained in event.nativeEvent.error

### onLoadStart

**React.PropTypes.func**

Callback triggered when we are processing the image to be displayed as specified by the source prop.

### onLoadEnd

**React.PropTypes.func**

Callback triggered when we have finished loading the image to be displayed. Whether or not the image was loaded and displayed properly will be indicated by the parameter "success". For example:
```javascript
_onLoadEnd(event:Event) {
    // Indication of asset loading success
    if(event.nativeEvent.success) {
        //our image successfully loaded!
    }
}
```

### onPinch

**React.PropTypes.func**

Called when the user performs a pinch gesture on the control. When the pinch starts, the scale factor is set to 1 is relative to the points of the two touch points. For example:
```javascript
_onPinch(pinchState, scaleFactor, source) {
    if(pinchState == 3) {
        // update scale of obj by multiplying by scaleFactor when pinch ends.
        return;
    }
    //set scale using native props to reflect pinch.
}
```
pinchState can be the following values:

| State Value | Description |
|-------------|-------------|
| 1 | Pinch Start: Triggered when the user has started a pinch gesture. |
| 2 | Pinch Move: Triggered when the user has adjusted the pinch, moving both fingers. |
| 3 | Pinch End: When the user has finishes the pinch gesture and released both touch points. |

**This event is only available in AR**.

### onRotate

**React.PropTypes.func**

Called when the user performs a rotation touch gesture on the control. Rotation factor is returned in degrees.
When setting rotation, the rotation should be relative to it's current rotation, *not* set to the absolute value of the given rotationFactor.
For example:
```javascript
_onRotate(rotateState, rotationFactor, source) {
    if (rotateState == 3) {
        //set to current rotation - rotationFactor.
        return;
    }
    //update rotation using setNativeProps
}
```
rotateState can be the following values:

| State Value | Description |
|-------------|-------------|
| 1 | Rotation Start: Triggered when the user has started a rotation gesture. |
| 2 | Rotation Move: Triggered when the user has adjusted the rotation, moving both fingers. |
| 3 | Rotation End: When the user has finishes the rotation gesture and released both touch points. |

**This event is only available in AR**.

### onScroll

**React.PropTypes.func**

Called when the user performs a scroll action, while hovering on the control.
For example:
```javascript
_onScroll(scrollPos, source) {
    // scrollPos[0]: x scroll position from 0.0 to 1.0.
    // scrollPos[1]: y scroll position from 0.0 to 1.0.
}
```
For the mapping of sources to controller inputs, see the Events section.
Unsupported VR Platforms: Cardboard(Android and iOS)

### onSwipe

**React.PropTypes.func**

Called when the user performs a swipe gesture on the physical controller, while hovering on this control. For example:
```javascript
_onSwipe(state, source) {
    if(state == 1) {
        // Swiped up
    } else if(state == 2) {
        // Swiped down
    } else if(state == 3) {
        // Swiped left
    } else if(state == 4) {
        // Swiped right
    }
}
```
For the mapping of sources to controller inputs, see the Events section.
Unsupported VR Platforms: Cardboard(Android and iOS)

### onTouch

**React.PropTypes.func**

Called when the user performs a touch action, while hovering on the control. Provides the touch state type, and the x/y coordinate at which this touch event has occurred.

| State Value | Description |
|-------------|-------------|
| 1 | Touch Down: Triggered when the user makes physical contact with the touch pad on the controller. |
| 2 | Touch Down Move: Called when the user moves around the touch pad immediately after having performed a Touch Down action. |
| 3 | Touch Up: Triggered after the user is no longer in physical contact with the touch pad after a Touch Down action. |

For example:
```javascript
_onTouch(state, touchPos, source) {
    var touchX = touchPos[0];
    var touchY = touchPos[1];
    if(state == 1) {
        // Touch Down
    } else if(state == 2) {
        // Touch Down Move
    } else if(state == 3) {
        // Touch Up
    }
}
```
For the mapping of sources to controller inputs, see the Events section.
Unsupported VR Platforms: Cardboard(Android and iOS).

### onTransformUpdate

**PropTypes.func**

A function that is invoked when the component moves and provides an array of numbers representing the component's position in world coordinates.

### opacity

**PropTypes.number**

A number from 0 to 1 that specifies the opacity of the object. A value of 1 translates into a fully opaque object while 0 represents full transparency.

### placeholderSource

**PropTypes.oneOfType([ PropTypes.shape({ uri: PropTypes.string }), PropTypes.number ])**

A static placeholder image that is shown until the source image is loaded it. It not specified, nothing will show until the source image is finished loading. PNG and JPG images accepted.
Example:
To invoke with local source: `require('image_placeholder.html');`

### position

**PropTypes.arrayOf(PropTypes.number)**

Cartesian position in 3D space, stored as [x, y, z].

### physicsBody

**PropTypes.shape({..physics.api..})**

Creates and binds a physics body that is configured with the provided collection of physics properties associated with this control.
For more information on physics components, please see the physics.api.

### resizeMode

**PropTypes.oneOf(['ScaleToFill','ScaleToFit','StretchToFill'])**

If not specified, the default value is stretchToFill.

| Value | Description |
|-------|-------------|
| scaleToFill | Scale the image up to fit the component width or height. Aspect ratio is preserved. |
| scaleToFit | Scale the image down to fit the component width or height. Aspect ratio is preserved. |
| stretchToFill | Stretch the image to fit on the entire surface of the ViroImage component. Aspect ratio is **not** preserved. |

### rotation

**PropTypes.arrayOf(PropTypes.number)**

The rotation of the transform in world space stored as Euler angles [x, y, z]. Units for each angle are specified in degrees.

### rotationPivot

**PropTypes.arrayOf(PropTypes.number)**

Cartesian position in [x,y,z] about which rotation is applied relative to the component's position.

### scale

**PropTypes.arrayOf(PropTypes.number)**

The scale of the image in 3D space, specified as [x,y,z]. A scale of 1 represents the current size of the image. A scale value of < 1 will make the image proportionally smaller while a value >1 will make the image proportionally bigger along the specified axis.

### scalePivot

**PropTypes.arrayOf(PropTypes.number)**

Cartesian position in [x,y,z] from which scale is applied relative to the component's position.

### shadowCastingBitMask

**PropTypes.number**

A bit mask that is bitwise and-ed (&) with each light's influenceBitMask. If the result is > 0, then this object will cast shadows from the light. For more information please see the Lighting and Materials Guide.

### stereoMode

**PropTypes.oneOf(['leftRight', 'rightLeft', 'topBottom', 'bottomTop', 'none'])**

Specifies the alignment mode of the provided stereo image in source. The image will be rendered in the given order, the first being the left eye, the next the right eye.
For example, leftRight will render the left half of the image to the left eye, and the right half of the image to the right eye. Similarly, topBottom will render the top half of the image to the left eye, and the bottom half of the image to the right eye. Defaults to none.

Note: There's a known issue with stereoscopic images of the format RGB565 (the fix is on the roadmap).

### style

**stylePropType**

Style properties determine the position and scale of the component within a ViroFlexView. Please see the UI Controls & Flexbox guide and Styles reference for more information.

### transformBehaviors

**PropTypes.oneOfType([PropTypes.arrayOf(PropTypes.string),PropTypes.string])**

An array of transform constraints that affect the transform of the image. For example, putting the value "billboard" will ensure the image is facing the user as the user rotates their head on any axis. This is useful for having the image always face the user on a particular axis, which especially useful for tappable icons.

Allowed values(values are case sensitive):

| Value | Description |
|-------|-------------|
| billboard | Billboard object on x,y,z axis |
| billboardX | Billboard object on the x axis |
| billboardY | Billboard object on the y axis |

### viroTag

**PropTypes.string**

A tag given to other components when their physics body collides with this component's physics body. Refer to physics for more information.

### visible

**PropTypes.bool**

False if the image should be hidden. By default the button is visible and this value is true.

### width

**PropTypes.number**

The width of the image in 3D space. Default value is 1.

### renderingOrder

**PropTypes.number**

This determines the order in which this Node is rendered relative to other Nodes. Nodes with greater rendering orders are rendered last. The default rendering order is zero. For example, setting a Node's rendering order to -1 will cause the Node to be rendered before all Nodes with rendering orders greater than or equal to 0.

## Methods

### async getBoundingBoxAsync()

Async function that returns the component's bounding box in world coordinates. Returns a Promise that will be completed with the following object:
```javascript
{
    boundingBox: {
        minX: number,
        maxX: number,
        minY: number,
        maxY: number,
        minZ: number,
        maxZ: number
    }
}
```

### async getTransformAsync()

Async function that returns the component's transform (position, scale and rotation).

| Return value | Description |
|--------------|-------------|
| transform | an object that contains "position", "scale" and "rotation" keys which point to number arrays |

### applyImpulse(force: arrayOf(number), position: arrayOf(number))

A function used with physics to apply an impulse (instantaneous) force to an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| force | an array of magnitudes to be applied as force (N) to the object in the positive x, y and z directions |

### applyTorqueImpulse(torque: arrayOf(number), position: arrayOf(number))

A function used with physics to apply an impulse (instantaneous) torque to an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| torque | an array of magnitudes to be applied as a torque (N * m) to the object in the positive x, y and z directions at the given position |
| position | a position relative to the object from which to apply the given torque |

### setVelocity(velocity: arrayOf(number))

A function used with physics to set the velocity of an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| velocity | an array of numbers corresponding to x, y, and z velocity |

### setNativeProps(nativeProps)

A wrapper function around the native component's setNativeProps which allow users to set values on the native component without changing state/setting props and re-rendering. Refer to the React Native documentation on Direct Manipulation for more information.

| Parameter | Type | Description |
|-----------|------|-------------|
| nativeProps | object | an object where the keys are the properties to set and the values are the values to set |

Example:
```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```

## Static Methods

### evictFromCache(source)

**source** - `PropTypes.oneOfType([PropTypes.shape({uri: PropTypes.string}), PropTypes.number])`

The given source will be purged from the memory and local storage cache of the device. On Android, Viro, like React-Native, uses the Fresco image library to load and cache images and this is required if the given source reference (uri, etc) is the same, but the image data changes. **Android Only**

---

# ViroARImageMarker

ViroARImageMarker is a component that attempts to find the given target in the user's view allowing the developer to place objects with respect to the image or to determine their location relative to the image target.

###### Example use:

```javascript
// Register the target
ViroARTrackingTargets.createTargets({
    "targetOne": {
        source: require('res/targetOne.html'),
        orientation: "Up",
        physicalWidth: 0.1 // real world width in meters  
    },
});

<ViroARImageMarker target={"targetOne"}>
  <ViroBox position={[0, .25, 0]} scale={[.5, .5, .5]} />
</ViroARImageMarker>
```

## Props

### target (* required)

| Type | Description |
|------|-------------|
| string | Name of the image target created with ViroARTrackingTargets. |

### dragPlane

| Type | Description |
|------|-------------|
| ViroDragPlane | When a drag type of "FixedToPlane" is given, dragging is limited to a user defined plane. The dragging behavior is then configured by this property (specified by a point on the plane and its normal vector). You can also limit the maximum distance the dragged object is allowed to travel away from the camera/controller (useful for situations where the user can drag an object towards infinity). |

### dragType

| Type | Description |
|------|-------------|
| "FixedDistance" \| "FixedToWorld" \| "FixedDistanceOrigin" \| "FixedToPlane" | Determines the behavior of drag if **onDrag** is specified. The default value is "FixedDistance".<br><br>**FixedDistance:** Dragging is limited to a fixed radius around the user, dragged from the point at which the user has grabbed the geometry containing this draggable node<br><br>**FixedDistanceOrigin:** Dragging is limited to a fixed radius around the user, dragged from the point of this node's position in world space.<br><br>**FixedToWorld:** Dragging is based on intersection with real world objects. **Available only in AR**<br><br>**FixedToPlane:** Dragging is limited to a fixed plane around the user. The configuration of this plane is defined by the **dragPlane** property. |

### ignoreEventHandling

| Type | Description |
|------|-------------|
| boolean | When set to true, this control will ignore events and not prevent controls behind it from receiving event callbacks.<br><br>The default value is false. |

### onAnchorFound

| Type | Description |
|------|-------------|
| (anchor) => void | Called when this component is anchored to a plane that is at least `minHeight` by `minWidth` large. This is when the component is made visible.<br><br>See Anchor for more. |

### onAnchorRemoved

| Type | Description |
|------|-------------|
| () => void | Called when this component is detached from a plane and is no longer visible. |

### onAnchorUpdated

| Type | Description |
|------|-------------|
| (anchor) => void | Called when the anchor is update. For image markers, there's additional information provided in the returned anchor.<br><br>See Anchor for more. |

### onClick
See ViroNode onClick.

### onClickState
See ViroNode onClickState.

### onCollision
See ViroNode onCollision.

### onDrag
See ViroNode onDrag.

### onFuse
See ViroNode onFuse.

### onHover
See ViroNode onHover.

### onPinch
See ViroNode onPinch.

### onRotate
See ViroNode onRotate.

### onScroll
See ViroNode onScroll.

### onSwipe
See ViroNode onSwipe.

### onTouch
See ViroNode onTouch.

### opacity

| Type | Description |
|------|-------------|
| number | A number from 0 to 1 that specifies the opacity of the object. A value of 1 translates into a fully opaque object while 0 represents full transparency. |

### pauseUpdates

| Type | Description |
|------|-------------|
| boolean | True/False to stop the automatic positioning/rotation of children components of a `ViroARPlane`. This does not stop `onAnchorUpdated` from being called. |

### viroTag

| Type | Description |
|------|-------------|
| string | A tag given to other components when their physics body collides with this component's physics body. Refer to physics for more information. |

### visible

| Type | Description |
|------|-------------|
| boolean | False if the container should be hidden. By default the container is visible and this value is true. |

### renderingOrder

| Type | Description |
|------|-------------|
| number | This determines the order in which this Node is rendered relative to other Nodes. Nodes with greater rendering orders are rendered last. The default rendering order is zero. For example, setting a Node's rendering order to -1 will cause the Node to be rendered before all Nodes with rendering orders greater than or equal to 0. |

### rotation

| Type | Description |
|------|-------------|
| [number, number, number] | The rotation of the component around it's local axis specified as Euler angles [x, y, z]. Units for each angle are specified in degrees. |

### scale

| Type | Description |
|------|-------------|
| [number, number, number] | The scale of the box in 3D space, specified as [x,y,z]. A scale of 1 represents the current size of the box. A scale value of < 1 will make the box proportionally smaller while a value >1 will make the box proportionally bigger along the specified axis. |

### transformBehaviors

| Type | Description |
|------|-------------|
| string[] | An array of transform constraints that affect the transform of the object. For example, putting the value "billboard" will ensure the box is facing the user as the user rotates their head on any axis. This is useful for icons or text where you'd like the box to always face the user at a particular rotation. Allowed values(values are case sensitive):<br><br>**"billboard":** Billboard object on x,y,z axis **"billboardX":** Billboard object on the x axis<br>**"billboardY":** Billboard object on the y axis<br>**"billboardZ":** Billboard object on the z axis |

## Methods

### setNativeProps(nativeProps)

A wrapper function around the native component's setNativeProps which allow users to set values on the native component without changing state/setting props and re-rendering. Refer to the React Native documentation on Direct Manipulation for more information.

| Parameter | Type | Description |
|-----------|------|-------------|
| nativeProps | object | an object where the keys are the properties to set and the values are the values to set |

```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```

---

# ViroARObjectMarker

> 📘 **Unsupported platform: Android**  
> This component is currently available only for iOS, specifically for iOS 12+/ARKit 2.0+

ViroARObjectMarker is a component that attempts to find the given target in the user's view allowing the developer to place objects with respect to the object or to determine their location relative to the object target.

This component has the exact same signature as ViroARImageMarker but its target uses the new "Object" target type.

You can create an .arobject file through the instructions here (download the sample app, build it, scan your object, and send it to your computer):  
[https://developer.apple.com/documentation/arkit/scanning_and_detecting_3d_objects?language=objc](https://developer.apple.com/documentation/arkit/scanning_and_detecting_3d_objects?language=objc)

If you are running into file path issues, make sure your metro.config.js file has been updated with the correct extensions -> [Adding Asset Types](/docs/assets#adding-asset-types-to-react-native).

###### Example use:

```javascript
// Register the target
ViroARTrackingTargets.createTargets({
  "targetOne": {
    source: require('./res/targetOne.arobject'),
    type: 'Object'
  },
});


<ViroARObjectMarker target={"targetOne"}>
  <ViroBox position={[0, .25, 0]} scale={[.5, .5, .5]} />
</ViroARObjectMarker>
```

## Props

### target (* required)

| Type | Description |
|------|-------------|
| string | Name of the object target created with ViroARTrackingTargets. |

### dragPlane

| Type | Description |
|------|-------------|
| [ViroDragPlane](/docs/virodragplane) | When a drag type of "FixedToPlane" is given, dragging is limited to a user defined plane. The dragging behavior is then configured by this property (specified by a point on the plane and its normal vector). You can also limit the maximum distance the dragged object is allowed to travel away from the camera/controller (useful for situations where the user can drag an object towards infinity). |

### dragType

| Type | Description |
|------|-------------|
| "FixedDistance" \| "FixedToWorld" \| "FixedDistanceOrigin" \| "FixedToPlane" | Determines the behavior of drag if **onDrag** is specified. The default value is "FixedDistance".<br><br>**FixedDistance:** Dragging is limited to a fixed radius around the user, dragged from the point at which the user has grabbed the geometry containing this draggable node<br><br>**FixedDistanceOrigin:** Dragging is limited to a fixed radius around the user, dragged from the point of this node's position in world space.<br><br>**FixedToWorld:** Dragging is based on intersection with real world objects. **Available only in AR**<br><br>**FixedToPlane:** Dragging is limited to a fixed plane around the user. The configuration of this plane is defined by the **dragPlane** property. |

### ignoreEventHandling

| Type | Description |
|------|-------------|
| boolean | When set to true, this control will ignore events and not prevent controls behind it from receiving event callbacks.<br><br>The default value is false. |

### onAnchorFound

| Type | Description |
|------|-------------|
| (anchor) => void | Called when this component is anchored to a plane that is at least `minHeight` by `minWidth` large. This is when the component is made visible. See [Anchor](/docs/viroarscene#anchor) |

### onAnchorRemoved

| Type | Description |
|------|-------------|
| () => void | Called when this component is detached from a plane and is no longer visible. |

### onAnchorUpdated

| Type | Description |
|------|-------------|
| (anchor) => void | Called when the plane to which this component is anchored is updated.<br><br>See [Anchor](/docs/viroarscene#anchor) |

### onClick
See [ViroNode onClick](/docs/vironode#onclick).

### onClickState
See [ViroNode onClickState](/docs/vironode#onclickstate).

### onCollision
See [ViroNode onCollision](/docs/vironode#oncollision).

### onDrag
See [ViroNode onDrag](/docs/vironode#ondrag).

### onFuse
See [ViroNode onFuse](/docs/vironode#onfuse).

### onHover
See [ViroNode onHover](/docs/vironode#onhover).

### onPinch
See [ViroNode onPinch](/docs/vironode#onpinch).

### onRotate
See [ViroNode onRotate](/docs/vironode#onrotate).

### onScroll
See [ViroNode onScroll](/docs/vironode#onscroll).

### onSwipe
See [ViroNode onSwipe](/docs/vironode#onswipe).

### onTouch
See [ViroNode onTouch](/docs/vironode#ontouch).

### onTransformUpdate
See [ViroNode onTransformUpdate](/docs/vironode#ontransformupdate).

### opacity

| Type | Description |
|------|-------------|
| number | A number from 0 to 1 that specifies the opacity of the container. A value of 1 translates into a fully opaque node while 0 represents full transparency. |

### pauseUpdates

| Type | Description |
|------|-------------|
| boolean | True/False to stop the automatic positioning/rotation of children components of a `ViroARPlane`. This does not stop `onAnchorUpdated` from being called. |

### viroTag

| Type | Description |
|------|-------------|
| string | A tag given to other components when their physics body collides with this component's physics body. Refer to [physics](/docs/physics) for more information. |

### visible

| Type | Description |
|------|-------------|
| boolean | False if the container should be hidden. By default the container is visible and this value is true. |

### renderingOrder

| Type | Description |
|------|-------------|
| number | This determines the order in which this Node is rendered relative to other Nodes. Nodes with greater rendering orders are rendered last. The default rendering order is zero. For example, setting a Node's rendering order to -1 will cause the Node to be rendered before all Nodes with rendering orders greater than or equal to 0. |

### rotation

| Type | Description |
|------|-------------|
| [number, number, number] | The rotation of the component around it's local axis specified as Euler angles [x, y, z]. Units for each angle are specified in degrees. |

### scale

| Type | Description |
|------|-------------|
| [number, number, number] | The scale of the box in 3D space, specified as [x,y,z]. A scale of 1 represents the current size of the box. A scale value of < 1 will make the box proportionally smaller while a value >1 will make the box proportionally bigger along the specified axis. |

### transformBehaviors

| Type | Description |
|------|-------------|
| string[] | An array of transform constraints that affect the transform of the object. For example, putting the value "billboard" will ensure the box is facing the user as the user rotates their head on any axis. This is useful for icons or text where you'd like the box to always face the user at a particular rotation. Allowed values(values are case sensitive):<br><br>**"billboard":** Billboard object on x,y,z axis **"billboardX":** Billboard object on the x axis<br>**"billboardY":** Billboard object on the y axis<br>**"billboardZ":** Billboard object on the z axis |

## Methods

### setNativeProps(nativeProps)

A wrapper function around the native component's setNativeProps which allow users to set values on the native component without changing state/setting props and re-rendering. Refer to the React Native documentation on [Direct Manipulation](https://facebook.github.io/react-native/docs/direct-manipulation) for more information.

| Parameter | Type | Description |
|-----------|------|-------------|
| nativeProps | object | an object where the keys are the properties to set and the values are the values to set |

```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```

---

# ViroARTrackingTargets

ViroARTrackingTargets contain the information required for AR tracking components such as ViroARImageMarker to work properly.

Before using an AR tracking component, a `ViroARTrackingTarget` should be created and referenced by name in the component itself.

Components that use `ViroARTrackingTargets`: ViroARImageMarker and ViroARObjectMarker

###### Example use:

```javascript
ViroARTrackingTargets.createTargets({
  "ben": {
    source: require('res/ben.png'),
    orientation: "Up",
    physicalWidth: 0.157, // real world width in meters  
    type: 'Image'
  },
  "targetOne": {
    source: require('res/targetOne.png'),
    orientation: "Up",
    physicalWidth: 0.25, // real world width in meters
    type: 'Image'
  }
});
```

## Methods

### static createTargets(targets:{[key:string]: any})

| Description |
|------------|
| This function creates the targets specified by the given targets object with the properties specified below under Image Target Properties. |

### static deleteTarget(targetName)

| Description |
|------------|
| This function takes the name of one registered target and deletes it. |

## Types of Targets

## Image Targets

Image targets should be used with ViroARImageMarkers and they specify the properties of a given image.

## Object Targets

Object targets should be used with ViroARObjectMarker and they specify the properties of a given object.

## ViroTrackingTarget

### source

| Type | Description |
|------|-------------|
| [ImageSourcePropType](https://reactnative.dev/docs/image#source) | The source of the image to find. An asset can be loaded by using `require()` or `{ uri: 'https://example.com/your-image.png' }` |

### orientation

| Type | Description |
|------|-------------|
| 'Up' \| 'Down' \| 'Left' \| 'Right' | Determines the orientation of the source image. |

### physicalWidth

| Type | Description |
|------|-------------|
| number | The width of the image in the real world in meters. |

### type

| Type | Description |
|------|-------------|
| 'Image' \| 'Object' | Determines the type of tracking target. |

---

# Viro3DObject

A component that displays a 3D Object that is positioned in world space. 

###### Example use:

```javascript
{/* Objects need lights to be visible! */}
<ViroAmbientLight
    color="#ffffff"
/>
<Viro3DObject
    source={require("./res/spaceship.obj")}
    resources={[
        require('./res/spaceship.mtl'),
        require('res/texture1.html'),
        require('res/texture2.html'),
        require('res/texture3.html')
    ]}
    highAccuracyEvents={true}
    position={[1, 3, -5]}
    scale={[2, 2, 2]}
    rotation={[45, 0, 0]}
    type="OBJ"
    transformBehaviors={["billboard"]}
/>
```

> 🚧 **Model Not Appearing?**
> 1. Try adding a light: `<ViroAmbientLight color="#FFFFFF" />`
> 2. Are your materials/textures in the right place? Most OBJ/FBX models expect their materials/textures in the same directory.
> 3. Is your model scaled/positioned properly? Viro displays the object in a 1 to 1 mapping of vertex coordinates to world space, so if your object coordinates specify a 100x100x100 model, then it'll appear 100x100x100 in Viro.
> 4. If you are running into file path issues, make sure your rn-cli.config file has been updated with the correct extensions -> [Adding Asset Types](https://docs.viromedia.com/docs/importing-assets#adding-asset-types)

## Props

### source (* required)

| Type | Description |
|------|-------------|
| [ImageSourcePropType](https://reactnative.dev/docs/image#source) | The object source, a remote URL or a local file resource. OBJ files accepted.To invoke with remote OBJ file:<br><br>`{ uri: "http://example.org/myobject.obj" }`<br><br>To invoke with local source:<br><br>`require('./myobject.obj')` |

### type (* required)

| Type | Description |
|------|-------------|
| 'OBJ' \| 'VRX' | Specify the 3d model file type being loaded, which can be OBJ or VRX. VRX is a custom model format for Viro. Currently, FBX files can be converted to VRX. For more information please see our [Assets](docs/assets) Guide. |

### animation

| Type | Description |
|------|-------------|
| [ViroAnimationProps](/docs/viroanimations#viroanimationprops) | A collection of parameters that determine if this component should animate. For more information on animated components please see our [Animation](/docs/animation) Guide.<br><br>Viro3DObject's animation also contains a `duration` property, which allows you to override the default duration of the named skeletal or keyframe animation. This enables you to speed up or slow down baked animations. The duration is specified in milliseconds. |

### dragType

| Type | Description |
|------|-------------|
| "FixedDistance" \| "FixedToWorld" \| "FixedToPlane" \| "FixedDistanceOrigin" | Determines the behavior of drag if **onDrag** is specified.<br><br>**"FixedDistance":** Dragging is limited to a fixed radius around the user, dragged from the point at which the user has grabbed the geometry containing this draggable node<br><br>**"FixedDistanceOrigin":** Dragging is limited to a fixed radius around the user, dragged from the point of this node's position in world space.<br><br>**"FixedToWorld":** Dragging is based on intersection with real world objects. **Available only in AR**<br><br>**"FixedToPlane":** Dragging is limited to a fixed plane around the user. The configuration of this plane is defined by the **dragPlane** property.<br><br>The default value is "FixedDistance". |

### dragPlane

| Type | Description |
|------|-------------|
| [ViroDragPlane](/docs/virodragplane) | When a drag type of "FixedToPlane" is given, dragging is limited to a user defined plane. The dragging behavior is then configured by this property (specified by a point on the plane and its normal vector). You can also limit the maximum distance the dragged object is allowed to travel away from the camera/controller (useful for situations where the user can drag an object towards infinity). |

### highAccuracyEvents

| Type | Description |
|------|-------------|
| boolean | True if events should use the geometry of the object to determine if the user is interacting with this object. If false, the object's axis-aligned bounding box will be used instead. Enabling this is more accurate but takes more processing power, so it is set to false by default. |

### ignoreEventHandling

| Type | Description |
|------|-------------|
| boolean | When set to true, this control will ignore events and not prevent controls behind it from receiving event callbacks.The default value is false. |

### lightReceivingBitMask

| Type | Description |
|------|-------------|
| number | A bit mask that is bitwise and-ed (&) with each light's influenceBitMask. If the result is > 0, then the light will illuminate this object. For more information please see the [Lighting and Materials](https://viro-community.readme.io/docs/lighting-and-materials) Guide. |

### materials

| Type | Description |
|------|-------------|
| string[] \| string | An array of strings that each represent a material that was created via ViroMaterials.createMaterials(). If materials are set on a Viro3DObject, then any materials in the OBJ file (e.g. materials loaded from the MTL) will be discarded and replaced with these materials.<br><br>**Swapping textures is not supported for VRX files** |

### morphTargets

| Type | Description |
|------|-------------|
| `{ target: string, weight: number }[]` | An array of dictionaries representing morph targets and its corresponding weight. Example usage:<br><br>`morphTargets={[{ target: "thin", weight: 1.0 }]}` |

### onClick

| Type | Description |
|------|-------------|
| Function | Called when an object has been clicked.<br><br>Example code:<br>`_onClick(position, source) { // user has clicked the object }`<br><br>The position parameter represents the position in world coordinates on the object where the click occurred. For the mapping of sources to controller inputs, see the [Events](/docs/events) section. |

### onClickState

| Type | Description |
|------|-------------|
| Function | Called for each click state an object goes through as it is clicked. Supported click states and their values are the following:<br><br>**Click Down (1):** Triggered when the user has performed a click down action while hovering on this control.<br><br>**Click Up (2):** Triggered when the user has performed a click up action while hovering on this control.<br><br>**Clicked (3):** Triggered when the user has performed both a click down and click up action on this control sequentially, thereby having "Clicked" the object.<br><br>For the mapping of sources to controller inputs, see the [Events](/docs/events) section. |

### onCollision

| Type | Description |
|------|-------------|
| Function | Called when this component's physics body collides with another component's physics body. Also invoked by [ViroScene](/docs/viroscene)/[ViroARScene](/docs/viroarscene)'s `findCollisions...` functions.<br><br>**viroTag:** the given viroTag (string) of the collided component<br><br>**collidedPoint:** an array of numbers representing the position, in world coordinates, of the point of collision<br><br>**collidedNormal:** an array representing the normal of the collision in world coordinates. |

### onDrag

| Type | Description |
|------|-------------|
| Function | Called when the view is currently being dragged. The dragToPos parameter provides the current 3D location of the dragged object.<br><br>Example code:<br>`_onDrag(dragToPos, source) { // dragtoPos[0]: x position // dragtoPos[1]: y position // dragtoPos[2]: z position }`<br><br>For the mapping of sources to controller inputs, see the [Events](/docs/events) section. **Unsupported VR Platforms: Cardboard iOS** |

### onError

| Type | Description |
|------|-------------|
| | Callback invoked when the OBJ model fails to load. The error message is contained in event.nativeEvent.error |

### onFuse

| Type | Description |
|------|-------------|
| `{ callback: Function, timeToFuse: number } \| Function` | As shown above, onFuse takes one of the types - either a callback, or a dictionary with a callback and duration. It is called after the user hovers onto and remains hovered on the control for a certain duration of time, as indicated in timeToFuse that represents the duration of time in milliseconds. While hovering, the reticle will display a count down animation while fusing towards timeToFuse.Note that timeToFuse defaults to 2000ms.For example:`_onFuse(source){ // User has hovered over object for timeToFuse milliseconds}`For the mapping of sources to controller inputs, see the [Events](/docs/events) section. |

### onHover

| Type | Description |
|------|-------------|
| Function | Called when the user hovers on or off the control.For example:<br><br>`_onHover(isHovering, position, source) { if(isHovering) { // user is hovering over the box } else { // user is no longer hovering over the box } }`<br><br>For the mapping of sources to controller inputs, see the [Events](/docs/events) section. |

### onLoadEnd

| Type | Description |
|------|-------------|
| Function | Callback invoked when the OBJ finishes loading, whether successful or in error. |

### onLoadStart

| Type | Description |
|------|-------------|
| Function | Callback invoked when the OBJ file starts loading. |

### onPinch

| Type | Description |
|------|-------------|
| Function | Called when the user performs a pinch gesture on the control. When the pinch starts, the scale factor is set to 1 is relative to the points of the two touch points.<br><br>For example:<br>`_onPinch(pinchState, scaleFactor, source) { if(pinchState == 3) { // update scale of obj by multiplying by scaleFactor when pinch ends. return; } //set scale using native props to reflect pinch. }`<br><br>pinchState can be the following values:<br><br>**Pinch Start (1):** Triggered when the user has started a pinch gesture.<br><br>**Pinch Move (2):** Triggered when the user has adjusted the pinch, moving both fingers.<br><br>**Pinch End (3):** When the user has finishes the pinch gesture and released both touch points.<br><br>**This event is only available in AR**. |

### onRotate

| Type | Description |
|------|-------------|
| Function | Called when the user performs a rotation touch gesture on the control. Rotation factor is returned in degrees.<br><br>When setting rotation, the rotation should be relative to it's current rotation, *not* set to the absolute value of the given rotationFactor.<br><br>For example:<br>`_onRotate(rotateState, rotationFactor, source) { if (rotateState == 3) { //set to current rotation - rotationFactor. return; } //update rotation using setNativeProps },`<br><br>rotationState can be the following values:<br><br>**Rotation Start (1):** Triggered when the user has started a rotation gesture.<br><br>**Rotation Move (2):** Triggered when the user has adjusted the rotation, moving both fingers.<br><br>**Rotation End (3):** When the user has finishes the rotation gesture and released both touch points.<br><br>**This event is only available in AR**. |

### onScroll

| Type | Description |
|------|-------------|
| Function | Called when the user performs a scroll action, while hovering on the control.<br><br>For example:<br>`_onScroll(scrollPos, source) { // scrollPos[0]: x scroll position from 0.0 to 1.0. // scrollPos[1]: y scroll position from 0.0 to 1.0. }`<br><br>For the mapping of sources to controller inputs, see the [Events](/docs/events) section.<br><br>**Unsupported VR Platforms: Cardboard(Android and iOS)** |

### onSwipe

| Type | Description |
|------|-------------|
| Function | Called when the user performs a swipe gesture on the physical controller, while hovering on this control.<br><br>For example:<br>`_onSwipe(state, source) { if(state == 1) { // Swiped up } else if(state == 2) { // Swiped down } else if(state == 3) { // Swiped left } else if(state == 4) { // Swiped right } }`<br><br>For the mapping of sources to controller inputs, see the [Events](/docs/events) section.<br><br>**Unsupported VR Platforms: Cardboard(Android and iOS)** |

### onTouch

| Type | Description |
|------|-------------|
| Function | Called when the user performs a touch action, while hovering on the control. Provides the touch state type, and the x/y coordinate at which this touch event has occurred.<br><br>**Touch Down (1):** Triggered when the user makes physical contact with the touch pad on the controller.<br><br>**Touch Down Move (2):** Called when the user moves around the touch pad immediately after having performed a Touch Down action.<br><br>**Touch Up (3):** Triggered after the user is no longer in physical contact with the touch pad after a Touch Down action.<br><br>For example:<br>`_onTouch(state, touchPos, source) { var touchX = touchPos[0]; var touchY = touchPos[1]; if(state == 1) { // Touch Down } else if(state == 2) { // Touch Down Move } else if(state == 3) { // Touch Up } }`<br><br>For the mapping of sources to controller inputs, see the [Events](/docs/events) section.<br><br>**Unsupported VR Platforms: Cardboard(Android and iOS).** |

### onTransformUpdate

| Type | Description |
|------|-------------|
| Function | A function that is invoked when the component moves and provides an array of numbers representing the component's position in world coordinates. |

### opacity

| Type | Description |
|------|-------------|
| number | A number from 0 to 1 that specifies the opacity of the object. A value of 1 translates into a fully opaque object while 0 represents full transparency. |

### physicsBody

| Type | Description |
|------|-------------|
| [Physics](/docs/physics#physicsbody-api) | Creates and binds a physics body that is configured with the provided collection of physics properties associated with this control. For more information on physics components, please see the [Physics Body API](/docs/physics#physicsbody-api). |

### position

| Type | Description |
|------|-------------|
| [number, number, number] | Cartesian position of the object in 3d world space, specified as [x, y, z]. |

### resources

| Type | Description |
|------|-------------|
| [ImageSourcePropType](https://reactnative.dev/docs/image#source)[] | Array of resources that are required by the OBJ file. OBJ files may references MTL files for materials, and various images for textures. In order for the packager to find these resources, they must also be listed here as an array, each with the require() function. |

### rotation

| Type | Description |
|------|-------------|
| [number, number, number] | The rotation of the object around it's local axis specified as Euler angles [x, y, z]. Units for each angle are specified in degrees. |

### rotationPivot

| Type | Description |
|------|-------------|
| [number, number, number] | Cartesian position in [x,y,z] about which rotation is applied relative to the component's position. |

### scale

| Type | Description |
|------|-------------|
| [number, number, number] | The scale of the object in 3d space, specified as [x,y,z]. A scale of 1 represents the object's current size. A scale value of < 1 will make the object proportionally smaller while a value >1 will make the object proportionally bigger along the specified axis. |

### scalePivot

| Type | Description |
|------|-------------|
| [number, number, number] | Cartesian position in [x,y,z] from which scale is applied relative to the component's position. |

### shadowCastingBitMask

| Type | Description |
|------|-------------|
| number | A bit mask that is bitwise and-ed (&) with each light's influenceBitMask. If the result is > 0, then this object will cast shadows from the light. For more information please see the [Lighting and Materials](/docs/3d-scene-lighting) Guide. |

### transformBehaviors

| Type | Description |
|------|-------------|
| string \| string[] | An array of transform constraints that affect the transform of the object. For example, putting the value "billboard" will ensure the object is facing the user as the user rotates their head on any axis. This is useful for icons or text where you'd like the object to always face the user.<br><br>Allowed values (values are case sensitive):<br><br>**'billboard':** Billboard object on x,y,z axis<br>**'billboardX':** Billboard object on the x axis<br>**'billboardY':** Billboard object on the y axis<br>**'billboardZ':** Billboard object on the z axis |

### viroTag

| Type | Description |
|------|-------------|
| string | A tag given to other components when their physics body collides with this component's physics body. Refer to [physics](/docs/physics) for more information. |

### visible

| Type | Description |
|------|-------------|
| boolean | False if the object should be hidden. By default the object is visible and this value is true. |

### renderingOrder

| Type | Description |
|------|-------------|
| number | This determines the order in which this Node is rendered relative to other Nodes. Nodes with greater rendering orders are rendered last. The default rendering order is zero. For example, setting a Node's rendering order to -1 will cause the Node to be rendered before all Nodes with rendering orders greater than or equal to 0. |

## Methods

### async getBoundingBoxAsync()

[Async](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) function that returns the component's bounding box in world coordinates.

| Arguments | Returns |
|-----------|----------|
| None | Returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) that will be completed with the following object:<br><br>`{ "boundingBox": { "minX": number, "maxX": number, "minY": number, "maxY": number, "minZ": number, "maxZ": number } }` |

### async getTransformAsync()

[Async](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) function that returns the component's transform (position, scale and rotation).

| Description | Returns |
|-------------|----------|
| None | **transform:** an object that contains "position", "scale" and "rotation" keys which point to number arrays |

### applyImpulse(force: arrayOf(number), position: arrayOf(number))

A function used with [physics](/docs/physics) to apply an impulse (instantaneous) force to an object with a physics body.

| Arguments | Returns |
|-----------|----------|
| None | **force:** an array of magnitudes to be applied as force (N) to the object in the positive x, y and z directions |

### applyTorqueImpulse(torque: number[], position: number[])

A function used with [physics](/docs/physics) to apply an impulse (instantaneous) torque to an object with a physics body.

| Arguments | Returns |
|-----------|----------|
| **torque:** an array of magnitudes to be applied as a torque (N * m) to the object in the positive x, y and z directions at the given position<br><br>**position:** a position relative to the object from which to apply the given torque | None |

### setVelocity(velocity: number[])

A function used with [physics](/docs/physics) to set the velocity of an object with a physics body.

| Arguments | Returns |
|-----------|----------|
| **velocity:** an array of numbers corresponding to x, y, and z velocity | None |

### setNativeProps(nativeProps)

A wrapper function around the native component's setNativeProps which allow users to set values on the native component without changing state/setting props and re-rendering. Refer to the React Native documentation on [Direct Manipulation](https://facebook.github.io/react-native/docs/direct-manipulation.html) for more information.

| Parameter | Type | Description |
|-----------|------|-------------|
| nativeProps | object | an object where the keys are the properties to set and the values are the values to set |

```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```

---

# ViroAmbientLight

A light object that emits ambient light that affects all objects equally. See our [Lighting and Material Guide](https://viro-community.readme.io/docs/lighting-and-materials) for more information on lights in a scene.

###### Example use:

```javascript
<ViroAmbientLight color="#ffffff" />
```

## Props

### color
| Type | Description |
|------|-------------|
| ColorPropType | The color of the light. The default light color is white. Valid color formats are:<br><br>- '#f0f' (#rgb)<br>- '#f0fc' (#rgba)<br>- '#ff00ff' (#rrggbb)<br>- '#ff00ff00' (#rrggbbaa)<br>- 'rgb(255, 255, 255)'<br>- 'rgba(255, 255, 255, 1.0)'<br>- 'hsl(360, 100%, 100%)'<br>- 'hsla(360, 100%, 100%, 1.0)'<br>- 'transparent'<br>- 'red'<br>- 0xff00ff00 (0xrrggbbaa)<br><br>See [Styles](/docs/styles#color) for more. |

### influenceBitMask
| Type | Description |
|------|-------------|
| number | This property is used to make lights apply to specific nodes. Lights and nodes in the scene can be assigned bit-masks to determine how each light influences each node. During rendering, Viro compares each light's influenceBitMask with each node's lightReceivingBitMask and shadowCastingBitMask. The bit-masks are compared using a bitwise AND operation:<br><br>If `(influenceBitMask & lightReceivingBitMask) != 0`, then the light will illuminate the node (and the node will receive shadows cast from objects occluding the light).<br><br>If `(influenceBitMask & shadowCastingBitMask) != 0`, then the node will cast shadows from the light.<br><br>The default mask is 0x1. |

### intensity
| Type | Description |
|------|-------------|
| number | The brightness of the light. Set to 1000 for normal intensity. The intensity is simply divided by 1000 and multiplied by the light's color.<br><br>Lower intensities will decrease the brightness of the light, and higher intensities will increase the brightness of the light.<br><br>The default intensity is 1000. |

### temperature
| Type | Description |
|------|-------------|
| number | The temperature of the light, in Kelvin. Viro will derive a hue from this temperature and multiply it by the light's color. To model a physical light with a known temperature, you can leave the color of this Light set to (1.0, 1.0, 1.0) and set its temperature only.<br><br>The default value for temperature is 6500K, which represents pure white light. |

### rotation
| Type | Description |
|------|-------------|
| number | The rotation of the component around its local axis specified as Euler angles [x, y, z]. Units for each angle are specified in degrees. |

### style
| Type | Description |
|------|-------------|
| [Styles](/docs/styles) | Styles of the component. |

### transformBehaviors
| Type | Description |
|------|-------------|
| string[] | An array of transform constraints that affect the transform of the object. For example, putting the value "billboard" will ensure the box is facing the user as the user rotates their head on any axis. This is useful for icons or text where you'd like the box to always face the user at a particular rotation. Allowed values(values are case sensitive):<br><br>**"billboard":** Billboard object on x,y,z axis<br>**"billboardX":** Billboard object on the x axis<br>**"billboardY":** Billboard object on the y axis<br>**"billboardZ":** Billboard object on the z axis |

### width
| Type | Description |
|------|-------------|
| number | The width of the component in 3D space. Default value is 1. |

### visible
| Type | Description |
|------|-------------|
| boolean | False if the box should be hidden. By default the box is visible and this value is true. |

## Methods

### setNativeProps(nativeProps)
A wrapper function around the native component's setNativeProps which allow users to set values on the native component without changing state/setting props and re-rendering. Refer to the React Native documentation on [Direct Manipulation](https://facebook.github.io/react-native/docs/direct-manipulation) for more information.

| Parameter | Type | Description |
|-----------|------|-------------|
| nativeProps | object | an object where the keys are the properties to set and the values are the values to set |

```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```I'll convert the HTML documentation into comprehensive Markdown format.

# ViroAmbientLight

A light object that emits ambient light that affects all objects equally. See our [Lighting and Material Guide](https://viro-community.readme.io/docs/lighting-and-materials) for more information on lights in a scene.

###### Example use:

```javascript
<ViroAmbientLight color="#ffffff" />
```

## Props

### color
| Type | Description |
|------|-------------|
| ColorPropType | The color of the light. The default light color is white. Valid color formats are:<br><br>- '#f0f' (#rgb)<br>- '#f0fc' (#rgba)<br>- '#ff00ff' (#rrggbb)<br>- '#ff00ff00' (#rrggbbaa)<br>- 'rgb(255, 255, 255)'<br>- 'rgba(255, 255, 255, 1.0)'<br>- 'hsl(360, 100%, 100%)'<br>- 'hsla(360, 100%, 100%, 1.0)'<br>- 'transparent'<br>- 'red'<br>- 0xff00ff00 (0xrrggbbaa)<br><br>See [Styles](/docs/styles#color) for more. |

### influenceBitMask
| Type | Description |
|------|-------------|
| number | This property is used to make lights apply to specific nodes. Lights and nodes in the scene can be assigned bit-masks to determine how each light influences each node. During rendering, Viro compares each light's influenceBitMask with each node's lightReceivingBitMask and shadowCastingBitMask. The bit-masks are compared using a bitwise AND operation:<br><br>If `(influenceBitMask & lightReceivingBitMask) != 0`, then the light will illuminate the node (and the node will receive shadows cast from objects occluding the light).<br><br>If `(influenceBitMask & shadowCastingBitMask) != 0`, then the node will cast shadows from the light.<br><br>The default mask is 0x1. |

### intensity
| Type | Description |
|------|-------------|
| number | The brightness of the light. Set to 1000 for normal intensity. The intensity is simply divided by 1000 and multiplied by the light's color.<br><br>Lower intensities will decrease the brightness of the light, and higher intensities will increase the brightness of the light.<br><br>The default intensity is 1000. |

### temperature
| Type | Description |
|------|-------------|
| number | The temperature of the light, in Kelvin. Viro will derive a hue from this temperature and multiply it by the light's color. To model a physical light with a known temperature, you can leave the color of this Light set to (1.0, 1.0, 1.0) and set its temperature only.<br><br>The default value for temperature is 6500K, which represents pure white light. |

### rotation
| Type | Description |
|------|-------------|
| number | The rotation of the component around its local axis specified as Euler angles [x, y, z]. Units for each angle are specified in degrees. |

### style
| Type | Description |
|------|-------------|
| [Styles](/docs/styles) | Styles of the component. |

### transformBehaviors
| Type | Description |
|------|-------------|
| string[] | An array of transform constraints that affect the transform of the object. For example, putting the value "billboard" will ensure the box is facing the user as the user rotates their head on any axis. This is useful for icons or text where you'd like the box to always face the user at a particular rotation. Allowed values(values are case sensitive):<br><br>**"billboard":** Billboard object on x,y,z axis<br>**"billboardX":** Billboard object on the x axis<br>**"billboardY":** Billboard object on the y axis<br>**"billboardZ":** Billboard object on the z axis |

### width
| Type | Description |
|------|-------------|
| number | The width of the component in 3D space. Default value is 1. |

### visible
| Type | Description |
|------|-------------|
| boolean | False if the box should be hidden. By default the box is visible and this value is true. |

## Methods

### setNativeProps(nativeProps)
A wrapper function around the native component's setNativeProps which allow users to set values on the native component without changing state/setting props and re-rendering. Refer to the React Native documentation on [Direct Manipulation](https://facebook.github.io/react-native/docs/direct-manipulation) for more information.

| Parameter | Type | Description |
|-----------|------|-------------|
| nativeProps | object | an object where the keys are the properties to set and the values are the values to set |

```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```

---

# Drag Plane

A drag plane allows you to specify the plane which the object is dragged. When a drag type of "FixedToPlane" is given, dragging is limited to a user defined plane. The dragging behavior is then configured by this property (specified by a point on the plane and its normal vector). You can also limit the maximum distance the dragged object is allowed to travel away from the camera/controller (useful for situations where the user can drag an object towards infinity). Make sure the object is positioned on the drag plane, or it may not appear to drag correctly.

## Example Usage

```javascript
<ViroBox
  position={[0, 0, -2]}
  dragPlane={{
    planeNormal: [0, 0, 0],
    planePoint: [0, 0, -2],
    maxDistance: 10,
  }}
  onDrag={(event) => console.log("Drag Event:", event)}
/>
```

## Properties

| Property | Type | Description |
|----------|------|-------------|
| planePoint | number[] | a point on the plane |
| planeNormal | number[] | the point on the plane's normal vector |
| maxDistance | number | the maximum distance the dragged object is allowed to travel away from the camera/controller |

---

# Viro Platform Constants

A collection of constants used in the Viro Platform.

## Example use:

## Constants:

### Video Recording/Screenshot Errors

```javascript
import { ViroRecordingErrorConstants } from '@reactvision/react-viro';

if (returnValue == ViroRecordingErrorConstants.RECORD_ERROR_NONE) {
  console.log("Success!");
}
```

| Name | Value | Notes |
|------|-------|-------|
| RECORD_ERROR_NONE | **-1** | Indicates that there is no error. |
| RECORD_ERROR_UNKNOWN | **0** | Indicates that the platform encountered an unknown error. |
| RECORD_ERROR_NO_PERMISSION | **1** | The user has denied permission required for recording/saving videos/screenshots. |
| RECORD_ERROR_INITIALIZATION | **2** | Indicates there was an error during initialization. |
| RECORD_ERROR_WRITE_TO_FILE | **3** | Indicates that there was an error writing to file. |
| RECORD_ERROR_ALREADY_RUNNING | **4** | Indicates that the system is already recording. |
| RECORD_ERROR_ALREADY_STOPPED | **5** | Indicates that the system is not currently recording. |

### AR Tracking States for ARScene

```typescript
import { 
  ViroTrackingStateConstants,
  ViroARTrackingReasonConstants
} from '@reactvision/react-viro';

if (returnValue == ViroTrackingStateConstants.TRACKING_LIMITED) {
  if (reason == ViroARTrackingReasonConstants.TRACKING_REASON_EXCESSIVE_MOTION) {
    console.log("Stop shaking the camera so hard!");
  }
}
```

| Name | Value | Notes |
|------|-------|-------|
| TRACKING_UNAVAILABLE | 1 | AR Camera position is not available. |
| TRACKING_LIMITED | 2 | Tracking is available but quality of results can be may be wildly inaccurate and should generally not be used |
| TRACKING_NORMAL | 3 | Camera position tracking is providing optimal results. |
| TRACKING_REASON_NONE | 1 | The current tracking state is not limited. |
| TRACKING_REASON_EXCESSIVE_MOTION | 2 | The device is moving too fast for accurate image-based position tracking. |
| TRACKING_REASON_INSUFFICIENT_FEATURES | 3 | The scene visible to the camera does not contain enough distinguishable features for image-based position tracking. |

---

# ViroBox

A simple 3D box that is defined by width, height and length.

###### Example use:

```javascript
<ViroBox
  height={2}
  length={2}
  width={2}
/>
```

## Props

### animation
| Type | Description |
|------|-------------|
| [ViroAnimation](/docs/viroanimations) | A collection of parameters that determine if this component should animate. For more information on animated components please see our [Animation](/docs/animation) Guide. |

### dragPlane
| Type | Description |
|------|-------------|
| [ViroDragPlane](/docs/virodragplane) | When a drag type of "FixedToPlane" is given, dragging is limited to a user defined plane. The dragging behavior is then configured by this property (specified by a point on the plane and its normal vector). You can also limit the maximum distance the dragged object is allowed to travel away from the camera/controller (useful for situations where the user can drag an object towards infinity). |

### dragType
| Type | Description |
|------|-------------|
| "FixedDistance" \| "FixedToWorld" \| "FixedDistanceOrigin" \| "FixedToPlane" | Determines the behavior of drag if **onDrag** is specified. The default value is "FixedDistance".<br><br>**FixedDistance:** Dragging is limited to a fixed radius around the user, dragged from the point at which the user has grabbed the geometry containing this draggable node<br><br>**FixedDistanceOrigin:** Dragging is limited to a fixed radius around the user, dragged from the point of this node's position in world space.<br><br>**FixedToWorld:** Dragging is based on intersection with real world objects. **Available only in AR**<br><br>**FixedToPlane:** Dragging is limited to a fixed plane around the user. The configuration of this plane is defined by the **dragPlane** property. |

### height
| Type | Description |
|------|-------------|
| number | The height of the box in 3D space. Default value is 1. |

### highAccuracyEvents
| Type | Description |
|------|-------------|
| boolean | True if events should use the geometry of the object to determine if the user is interacting with this object. If false, the object's axis-aligned bounding box will be used instead. Enabling this is more accurate but takes more processing power, so it is set to false by default. |

### ignoreEventHandling
| Type | Description |
|------|-------------|
| boolean | When set to true, this control will ignore events and not prevent controls behind it from receiving event callbacks.<br><br>The default value is false. |

### lightReceivingBitMask
| Type | Description |
|------|-------------|
| number | A bit mask that is bitwise and-ed (&) with each light's influenceBitMask. If the result is > 0, then the light will illuminate this object. For more information please see the [Lighting and Materials](/docs/lighting-and-materials) Guide. |

### length
| Type | Description |
|------|-------------|
| number | The length of the box in 3D space. Default value is 1.0. |

### materials
| Type | Description |
|------|-------------|
| string[] | An array of strings that each represent a material that was created via ViroMaterials.createMaterials(). See [ViroMaterials](/docs/viromaterials) for more.<br><br>A ViroBox can accept one material, which is used for all sides. |

### onClick
See [ViroNode onClick](/docs/vironode#onclick).

### onClickState
See [ViroNode onClickState](/docs/vironode#onclickstate).

### onCollision
See [ViroNode onCollision](/docs/vironode#oncollision).

### onDrag
See [ViroNode onDrag](/docs/vironode#ondrag).

### onFuse
See [ViroNode onFuse](/docs/vironode#onfuse).

### onHover
See [ViroNode onHover](/docs/vironode#onhover).

### onPinch
See [ViroNode onPinch](/docs/vironode#onpinch).

### onRotate
See [ViroNode onRotate](/docs/vironode#onrotate).

### onScroll
See [ViroNode onScroll](/docs/vironode#onscroll).

### onSwipe
See [ViroNode onSwipe](/docs/vironode#onswipe).

### onTouch
See [ViroNode onTouch](/docs/vironode#ontouch).

### onTransformUpdate
See [ViroNode onTransformUpdate](/docs/vironode#ontransformupdate).

### opacity
| Type | Description |
|------|-------------|
| number | A number from 0 to 1 that specifies the opacity of the object. A value of 1 translates into a fully opaque object while 0 represents full transparency. |

### position
| Type | Description |
|------|-------------|
| [number, number, number] | Cartesian position of the box in 3D world space, specified as [x, y, z]. |

### physicsBody
| Type | Description |
|------|-------------|
| [Physics Body](/docs/physics#physicsbody-api) | Creates and binds a physics body that is configured with the provided collection of physics properties associated with this control.For more information on physics components, please see the [Physics](/docs/physics). |

### rotation
| Type | Description |
|------|-------------|
| [number, number, number] | The rotation of the component around it's local axis specified as Euler angles [x, y, z]. Units for each angle are specified in degrees. |

### rotationPivot
| Type | Description |
|------|-------------|
| [number, number, number] | Cartesian position in [x,y,z] about which rotation is applied relative to the component's position. |

### scale
| Type | Description |
|------|-------------|
| [number, number, number] | The scale of the box in 3D space, specified as [x,y,z]. A scale of 1 represents the current size of the box. A scale value of < 1 will make the box proportionally smaller while a value >1 will make the box proportionally bigger along the specified axis. |

### scalePivot
| Type | Description |
|------|-------------|
| [number, number, number] | Cartesian position in [x,y,z] from which scale is applied relative to the component's position. |

### shadowCastingBitMask
| Type | Description |
|------|-------------|
| number | A bit mask that is bitwise and-ed (&) with each light's influenceBitMask. If the result is > 0, then this object will cast shadows from the light. For more information please see the [Lighting and Materials](/docs/lighting-and-materials) Guide. |

### transformBehaviors
| Type | Description |
|------|-------------|
| string[] | An array of transform constraints that affect the transform of the object. For example, putting the value "billboard" will ensure the box is facing the user as the user rotates their head on any axis. This is useful for icons or text where you'd like the box to always face the user at a particular rotation. Allowed values(values are case sensitive):<br><br>**"billboard":** Billboard object on x,y,z axis **"billboardX":** Billboard object on the x axis<br>**"billboardY":** Billboard object on the y axis<br>**"billboardZ":** Billboard object on the z axis |

### viroTag
| Type | Description |
|------|-------------|
| string | A tag given to other components when their physics body collides with this component's physics body. Refer to [physics](/docs/physics) for more information. |

### visible
| Type | Description |
|------|-------------|
| boolean | False if the box should be hidden. By default the box is visible and this value is true. |

### width
| Type | Description |
|------|-------------|
| number | The width of the component in 3D space. Default value is 1. |

### renderingOrder
| Type | Description |
|------|-------------|
|  | This determines the order in which this Node is rendered relative to other Nodes. Nodes with greater rendering orders are rendered last. The default rendering order is zero. For example, setting a Node's rendering order to -1 will cause the Node to be rendered before all Nodes with rendering orders greater than or equal to 0. |

## Methods

### async getBoundingBoxAsync()

[Async](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) function that returns the component's bounding box in world coordinates. Returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) that will be completed with the following object:

```json
{
  "boundingBox": {
    "minX": number,
    "maxX": number,
    "minY": number,
    "maxY": number,
    "minZ": number, 
    "maxZ": number
  }
}
```

### async getTransformAsync()

[Async](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) function that returns the component's transform (position, scale and rotation).

Returns a *transform* object that contains "position", "scale" and "rotation" keys which point to number arrays

### applyImpulse(force: number[], position: number[])

A function used with [physics](/docs/physics) to apply an impulse (instantaneous) force to an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| force | an array of magnitudes to be applied as force (N) to the object in the positive x, y and z directions |
| position | a position relative to the object from which to apply the given force |

### applyTorqueImpulse(torque: number[], position: number[])

A function used with [physics](/docs/physics) to apply an impulse (instantaneous) torque to an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| torque | an array of magnitudes to be applied as a torque (N * m) to the object in the positive x, y and z directions at the given position |
| position | a position relative to the object from which to apply the given torque |

### setVelocity(velocity: number[])

A function used with [physics](/docs/physics) to set the velocity of an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| velocity | an array of numbers corresponding to x, y, and z velocity |

### setNativeProps(nativeProps)

A wrapper function around the native component's setNativeProps which allow users to set values on the native component without changing state/setting props and re-rendering. Refer to the React Native documentation on [Direct Manipulation](https://facebook.github.io/react-native/docs/direct-manipulation) for more information.

| Parameter | Description |
|-----------|-------------|
| nativeProps | an object where the keys are the properties to set and the values are the values to set |

```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```You're right - I'll provide the complete markdown conversion with every detail from the original document:

# ViroBox

A simple 3D box that is defined by width, height and length.

###### Example use:

```javascript
<ViroBox
  height={2}
  length={2}
  width={2}
/>
```

## Props

### animation
| Type | Description |
|------|-------------|
| [ViroAnimation](/docs/viroanimations) | A collection of parameters that determine if this component should animate. For more information on animated components please see our [Animation](/docs/animation) Guide. |

### dragPlane
| Type | Description |
|------|-------------|
| [ViroDragPlane](/docs/virodragplane) | When a drag type of "FixedToPlane" is given, dragging is limited to a user defined plane. The dragging behavior is then configured by this property (specified by a point on the plane and its normal vector). You can also limit the maximum distance the dragged object is allowed to travel away from the camera/controller (useful for situations where the user can drag an object towards infinity). |

### dragType
| Type | Description |
|------|-------------|
| "FixedDistance" \| "FixedToWorld" \| "FixedDistanceOrigin" \| "FixedToPlane" | Determines the behavior of drag if **onDrag** is specified. The default value is "FixedDistance".<br><br>**FixedDistance:** Dragging is limited to a fixed radius around the user, dragged from the point at which the user has grabbed the geometry containing this draggable node<br><br>**FixedDistanceOrigin:** Dragging is limited to a fixed radius around the user, dragged from the point of this node's position in world space.<br><br>**FixedToWorld:** Dragging is based on intersection with real world objects. **Available only in AR**<br><br>**FixedToPlane:** Dragging is limited to a fixed plane around the user. The configuration of this plane is defined by the **dragPlane** property. |

### height
| Type | Description |
|------|-------------|
| number | The height of the box in 3D space. Default value is 1. |

### highAccuracyEvents
| Type | Description |
|------|-------------|
| boolean | True if events should use the geometry of the object to determine if the user is interacting with this object. If false, the object's axis-aligned bounding box will be used instead. Enabling this is more accurate but takes more processing power, so it is set to false by default. |

### ignoreEventHandling
| Type | Description |
|------|-------------|
| boolean | When set to true, this control will ignore events and not prevent controls behind it from receiving event callbacks.<br><br>The default value is false. |

### lightReceivingBitMask
| Type | Description |
|------|-------------|
| number | A bit mask that is bitwise and-ed (&) with each light's influenceBitMask. If the result is > 0, then the light will illuminate this object. For more information please see the [Lighting and Materials](/docs/lighting-and-materials) Guide. |

### length
| Type | Description |
|------|-------------|
| number | The length of the box in 3D space. Default value is 1.0. |

### materials
| Type | Description |
|------|-------------|
| string[] | An array of strings that each represent a material that was created via ViroMaterials.createMaterials(). See [ViroMaterials](/docs/viromaterials) for more.<br><br>A ViroBox can accept one material, which is used for all sides. |

### onClick
See [ViroNode onClick](/docs/vironode#onclick).

### onClickState
See [ViroNode onClickState](/docs/vironode#onclickstate).

### onCollision
See [ViroNode onCollision](/docs/vironode#oncollision).

### onDrag
See [ViroNode onDrag](/docs/vironode#ondrag).

### onFuse
See [ViroNode onFuse](/docs/vironode#onfuse).

### onHover
See [ViroNode onHover](/docs/vironode#onhover).

### onPinch
See [ViroNode onPinch](/docs/vironode#onpinch).

### onRotate
See [ViroNode onRotate](/docs/vironode#onrotate).

### onScroll
See [ViroNode onScroll](/docs/vironode#onscroll).

### onSwipe
See [ViroNode onSwipe](/docs/vironode#onswipe).

### onTouch
See [ViroNode onTouch](/docs/vironode#ontouch).

### onTransformUpdate
See [ViroNode onTransformUpdate](/docs/vironode#ontransformupdate).

### opacity
| Type | Description |
|------|-------------|
| number | A number from 0 to 1 that specifies the opacity of the object. A value of 1 translates into a fully opaque object while 0 represents full transparency. |

### position
| Type | Description |
|------|-------------|
| [number, number, number] | Cartesian position of the box in 3D world space, specified as [x, y, z]. |

### physicsBody
| Type | Description |
|------|-------------|
| [Physics Body](/docs/physics#physicsbody-api) | Creates and binds a physics body that is configured with the provided collection of physics properties associated with this control.For more information on physics components, please see the [Physics](/docs/physics). |

### rotation
| Type | Description |
|------|-------------|
| [number, number, number] | The rotation of the component around it's local axis specified as Euler angles [x, y, z]. Units for each angle are specified in degrees. |

### rotationPivot
| Type | Description |
|------|-------------|
| [number, number, number] | Cartesian position in [x,y,z] about which rotation is applied relative to the component's position. |

### scale
| Type | Description |
|------|-------------|
| [number, number, number] | The scale of the box in 3D space, specified as [x,y,z]. A scale of 1 represents the current size of the box. A scale value of < 1 will make the box proportionally smaller while a value >1 will make the box proportionally bigger along the specified axis. |

### scalePivot
| Type | Description |
|------|-------------|
| [number, number, number] | Cartesian position in [x,y,z] from which scale is applied relative to the component's position. |

### shadowCastingBitMask
| Type | Description |
|------|-------------|
| number | A bit mask that is bitwise and-ed (&) with each light's influenceBitMask. If the result is > 0, then this object will cast shadows from the light. For more information please see the [Lighting and Materials](/docs/lighting-and-materials) Guide. |

### transformBehaviors
| Type | Description |
|------|-------------|
| string[] | An array of transform constraints that affect the transform of the object. For example, putting the value "billboard" will ensure the box is facing the user as the user rotates their head on any axis. This is useful for icons or text where you'd like the box to always face the user at a particular rotation. Allowed values(values are case sensitive):<br><br>**"billboard":** Billboard object on x,y,z axis **"billboardX":** Billboard object on the x axis<br>**"billboardY":** Billboard object on the y axis<br>**"billboardZ":** Billboard object on the z axis |

### viroTag
| Type | Description |
|------|-------------|
| string | A tag given to other components when their physics body collides with this component's physics body. Refer to [physics](/docs/physics) for more information. |

### visible
| Type | Description |
|------|-------------|
| boolean | False if the box should be hidden. By default the box is visible and this value is true. |

### width
| Type | Description |
|------|-------------|
| number | The width of the component in 3D space. Default value is 1. |

### renderingOrder
| Type | Description |
|------|-------------|
|  | This determines the order in which this Node is rendered relative to other Nodes. Nodes with greater rendering orders are rendered last. The default rendering order is zero. For example, setting a Node's rendering order to -1 will cause the Node to be rendered before all Nodes with rendering orders greater than or equal to 0. |

## Methods

### async getBoundingBoxAsync()

[Async](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) function that returns the component's bounding box in world coordinates. Returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) that will be completed with the following object:

```json
{
  "boundingBox": {
    "minX": number,
    "maxX": number,
    "minY": number,
    "maxY": number,
    "minZ": number, 
    "maxZ": number
  }
}
```

### async getTransformAsync()

[Async](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) function that returns the component's transform (position, scale and rotation).

Returns a *transform* object that contains "position", "scale" and "rotation" keys which point to number arrays

### applyImpulse(force: number[], position: number[])

A function used with [physics](/docs/physics) to apply an impulse (instantaneous) force to an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| force | an array of magnitudes to be applied as force (N) to the object in the positive x, y and z directions |
| position | a position relative to the object from which to apply the given force |

### applyTorqueImpulse(torque: number[], position: number[])

A function used with [physics](/docs/physics) to apply an impulse (instantaneous) torque to an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| torque | an array of magnitudes to be applied as a torque (N * m) to the object in the positive x, y and z directions at the given position |
| position | a position relative to the object from which to apply the given torque |

### setVelocity(velocity: number[])

A function used with [physics](/docs/physics) to set the velocity of an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| velocity | an array of numbers corresponding to x, y, and z velocity |

### setNativeProps(nativeProps)

A wrapper function around the native component's setNativeProps which allow users to set values on the native component without changing state/setting props and re-rendering. Refer to the React Native documentation on [Direct Manipulation](https://facebook.github.io/react-native/docs/direct-manipulation) for more information.

| Parameter | Description |
|-----------|-------------|
| nativeProps | an object where the keys are the properties to set and the values are the values to set |

```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```

---

# ViroAnimatedImage

A component used to display plays animated GIF images from local or remote sources.

###### Example use:

```javascript
<ViroAnimatedImage
    height={2}
    width={2}
    placeholderSource={require("./res/local_spinner.jpg")}
    source={{ uri: "https://myGIFImage.jpg" }}
/>
```

## Props

### source (*required*)
**Type:** [ImageSourcePropType](https://reactnative.dev/docs/image#source)  
**Description:** The image source, a remote URL or a local file resource. PNG and JPG images accepted.

```javascript
// use remote URL
<ViroAnimatedImage
    height={2}
    width={2}
    placeholderSource={require("./res/local_spinner.jpg")}
    source={{ uri: "https://myGIFImage.jpg" }}
/>

// use file in bundle
<ViroAnimatedImage
    height={2}
    width={2}
    placeholderSource={require("./res/local_spinner.jpg")}
    source={require('../res/emoji.vrx')}
/>
```

### animation
**Type:** [ViroAnimationProps](/docs/viroanimations#viroanimationprops)  
**Description:** A collection of parameters that determine if this component should animate. For more information on animated components please see our [Animation](/docs/animation) Guide.

### dragPlane
**Type:** [ViroDragPlane](/docs/virodragplane)  
**Description:** When a drag type of "FixedToPlane" is given, dragging is limited to a user defined plane. The dragging behavior is then configured by this property (specified by a point on the plane and its normal vector). You can also limit the maximum distance the dragged object is allowed to travel away from the camera/controller (useful for situations where the user can drag an object towards infinity).

### dragType
**Type:** "FixedDistance" | "FixedToWorld" | "FixedDistanceOrigin" | "FixedToPlane"  
**Description:** Determines the behavior of drag if **onDrag** is specified. The default value is "FixedDistance".

- **FixedDistance:** Dragging is limited to a fixed radius around the user, dragged from the point at which the user has grabbed the geometry containing this draggable node
- **FixedDistanceOrigin:** Dragging is limited to a fixed radius around the user, dragged from the point of this node's position in world space.
- **FixedToWorld:** Dragging is based on intersection with real world objects. **Available only in AR**
- **FixedToPlane:** Dragging is limited to a fixed plane around the user. The configuration of this plane is defined by the **dragPlane** property.

### format
**Type:** 'RGBA8' | 'RGBA4' | 'RGB565'  
**Description:** Image texture formats for storage on the GPU.

- **RGBA8:** Each pixel is described with 32-bits, using eight bits per channel
- **RGBA4:** Each pixel is described with 16 bits, using four bits per channel
- **RGB565:** Formats the picture into 16 bit color values without alpha

### height
**Type:** number  
**Description:** The height of the image in 3D space. Default value is 1.

### highAccuracyEvents
**Type:** boolean  
**Description:** True if events should use the geometry of the object to determine if the user is interacting with this object. If false, the object's axis-aligned bounding box will be used instead. Enabling this is more accurate but takes more processing power, so it is set to false by default.

### ignoreEventHandling
**Type:** boolean  
**Description:** When set to true, this control will ignore events and not prevent controls behind it from receiving event callbacks.

The default value is false.

### imageClipMode
**Type:** 'None' | 'ClipToBounds'  
**Description:** Defaults to clipToBounds. If `ClipToBounds` is set, image is cropped if it overshoots with `ScaleToFill`. Setting to `None` does not crop image if it overshoots bounds.

### lightReceivingBitMask
**Type:** number  
**Description:** A bit mask that is bitwise and-ed (&) with each light's influenceBitMask. If the result is > 0, then the light will illuminate this object. For more information please see the [Lighting and Materials](/docs/lighting-and-materials) Guide.

### loop
**Type:** boolean  
**Description:** Set to true to loop the animation. This is set to false by default.

### materials
**Type:** string | string[]  
**Description:** An array of strings that each represent a material that was created via ViroMaterials.createMaterials(). ViroImage takes 1 material. The diffuseTexture of the material will be the image shown unless a source prop is specified. The other material properties will preserve as the source prop changes.

### mipmap
**Type:** boolean  
**Description:** If true, we dynamically generate mipmaps for the source of this given image.

### onClick
See [ViroNode onClick](/docs/vironode#onclick).

### onClickState
See [ViroNode onClickState](/docs/vironode#onclickstate).

### onCollision
See [ViroNode onCollision](/docs/vironode#oncollision).

### onDrag
See [ViroNode onDrag](/docs/vironode#ondrag).

### onFuse
See [ViroNode onFuse](/docs/vironode#onfuse).

### onHover
See [ViroNode onHover](/docs/vironode#onhover).

### onError
**Type:** (event) => void  
**Description:** Callback invoked when the Image fails to load. The error message is contained in event.nativeEvent.error

### onLoadStart
**Type:** () => void  
**Description:** Callback triggered when we are processing the image to be displayed as specified by the source prop.

### onLoadEnd
**Description:** Callback triggered when we have finished loading the image to be displayed. Whether or not the image was loaded and displayed properly will be indicated by the parameter "success".

```javascript
<ViroAnimatedImage
  source={require('./path/to/your/image.png')}
  onLoadEnd={(event) => {
    if (event.nativeEvent.success) {
      // image successfully loaded
    }
  } />
```

### onPinch
See [ViroNode onPinch](/docs/vironode#onpinch).

### onRotate
See [ViroNode onRotate](/docs/vironode#onrotate).

### onScroll
See [ViroNode onScroll](/docs/vironode#onscroll).

### onSwipe
See [ViroNode onSwipe](/docs/vironode#onswipe).

### onTouch
See [ViroNode onTouch](/docs/vironode#ontouch).

### onTransformUpdate
See [ViroNode onTransformUpdate](/docs/vironode#ontransformupdate).

### opacity
**Type:** number  
**Description:** A number from 0 to 1 that specifies the opacity of the object. A value of 1 translates into a fully opaque object while 0 represents full transparency.

### paused
**Type:** boolean  
**Description:** Set to true to pause the GIF animation. This is set to false by default.

### placeholderSource
**Type:** [ImageSourcePropType](https://reactnative.dev/docs/image#source)  
**Description:** A static placeholder image that is shown until the source image is loaded it. It not specified, nothing will show until the source image is finished loading. PNG and JPG images accepted.

### position
**Type:** [number, number, number]  
**Description:** Cartesian position in 3D space, stored as [x, y, z].

### physicsBody
**Type:** [Physics Body](/docs/physics#physicsbody-api)  
**Description:** Creates and binds a physics body that is configured with the provided collection of physics properties associated with this control. For more information on physics components, please see the [Physics](/docs/physics).

### resizeMode
**Type:** 'ScaleToFill' | 'ScaleToFit' | 'StretchToFill'  
**Description:** If not specified, the default value is stretchToFill.

- **scaleToFill:** Scale the image up to fit the component width or height. Aspect ratio is preserved.
- **scaleToFit:** Scale the image down to fit the component width or height. Aspect ratio is preserved.
- **stretchToFill:** Stretch the image to fit on the entire surface of the ViroImage component. Aspect ratio is **not** preserved.

### rotation
**Type:** [number, number, number]  
**Description:** The rotation of the box around it's local axis specified as Euler angles [x, y, z]. Units for each angle are specified in degrees.

### rotationPivot
**Type:** [number, number, number]  
**Description:** Cartesian position in [x,y,z] about which rotation is applied relative to the component's position.

### scale
**Type:** [number, number, number]  
**Description:** The scale of the box in 3D space, specified as [x,y,z]. A scale of 1 represents the current size of the box. A scale value of < 1 will make the box proportionally smaller while a value >1 will make the box proportionally bigger along the specified axis.

### scalePivot
**Type:** [number, number, number]  
**Description:** Cartesian position in [x,y,z] from which scale is applied relative to the component's position.

### shadowCastingBitMask
**Type:** number  
**Description:** A bit mask that is bitwise and-ed (&) with each light's influenceBitMask. If the result is > 0, then this object will cast shadows from the light. For more information please see the [Lighting and Materials](/docs/lighting-and-materials) Guide.

### stereoMode
**Type:** 'leftRight' | 'rightLeft' | 'topBottom' | 'bottomTop' | 'none'  
**Description:** Specifies the alignment mode of the provided stereo image in source. The image will be rendered in the given order, the first being the left eye, the next the right eye. For example, leftRight will render the left half of the image to the left eye, and the right half of the image to the right eye. Similarly, topBottom will render the top half of the image to the left eye, and the bottom half of the image to the right eye. Defaults to none.

Note: There's a known issue with stereoscopic images of the format RGB565 (the fix is on the roadmap).

### style
**Type:** [Styles](/docs/styles)  
**Description:** Style properties determine the position and scale of the component within a ViroFlexView. Please see the [UI Controls & Flexbox](/docs/flexbox-ui-layouts) guide and [Styles](/docs/styles) reference for more information.

### transformBehaviors
**Type:** string[]  
**Description:** An array of transform constraints that affect the transform of the object. For example, putting the value "billboard" will ensure the box is facing the user as the user rotates their head on any axis. This is useful for icons or text where you'd like the box to always face the user at a particular rotation. Allowed values(values are case sensitive):

- **"billboard":** Billboard object on x,y,z axis
- **"billboardX":** Billboard object on the x axis
- **"billboardY":** Billboard object on the y axis
- **"billboardZ":** Billboard object on the z axis

### viroTag
**Type:** string  
**Description:** A tag given to other components when their physics body collides with this component's physics body. Refer to [physics](/docs/physics) for more information.

### visible
**Type:** boolean  
**Description:** False if the container should be hidden. By default the container is visible and this value is true.

### width
**Description:** The width of the image in 3D space. Default value is 1.

## Methods

### async getBoundingBoxAsync()
[Async](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) function that returns the component's bounding box in world coordinates. Returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) that will be completed with the following object:

```json
{
  "boundingBox": {
    "minX": number,
    "maxX": number,
    "minY": number,
    "maxY": number,
    "minZ": number, 
    "maxZ": number
  }
}
```

### async getTransformAsync()
[Async](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) function that returns the component's transform (position, scale and rotation).

Returns a *transform* object that contains "position", "scale" and "rotation" keys which point to number arrays

### applyImpulse(force: number[], position: number[])
A function used with [physics](/docs/physics) to apply an impulse (instantaneous) force to an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| force | an array of magnitudes to be applied as force (N) to the object in the positive x, y and z directions |
| position | a position relative to the object from which to apply the given force |

### applyTorqueImpulse(torque: number[], position: number[])
A function used with [physics](/docs/physics) to apply an impulse (instantaneous) torque to an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| torque | an array of magnitudes to be applied as a torque (N * m) to the object in the positive x, y and z directions at the given position |
| position | a position relative to the object from which to apply the given torque |

### setVelocity(velocity: number[])
A function used with [physics](/docs/physics) to set the velocity of an object with a physics body.

| Parameter | Description |
|-----------|-------------|
| velocity | an array of numbers corresponding to x, y, and z velocity |

### setNativeProps(nativeProps)
A wrapper function around the native component's setNativeProps which allow users to set values on the native component without changing state/setting props and re-rendering. Refer to the React Native documentation on [Direct Manipulation](https://facebook.github.io/react-native/docs/direct-manipulation) for more information.

| Parameter | Description |
|-----------|-------------|
| nativeProps | an object where the keys are the properties to set and the values are the values to set |

```javascript
componentRef.setNativeProps({ position: [0, 0, -1] });
```

## Static Methods

### evictFromCache(source: [ImageSourcePropType](https://reactnative.dev/docs/image#source))
The given source will be purged from the memory and local storage cache of the device. On Android, Viro, like React-Native, uses the Fresco image library to load and cache images and this is required if the given source reference (uri, etc) is the same, but the image data changes.

**Android Only**

---

