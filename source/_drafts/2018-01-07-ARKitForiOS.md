# ARKit

## 追踪

## 场景理解

## 渲染


## ARSCNView

## ARSession

### ARFrame 

就是当前时刻的快照，包括 session 的所有状态，所有渲染增强现实场景所需的信息。要访问 ARFrame，只要获取 ARSession 的 currentFrame 属性。或者也可以把自己设置为 delegate，接收新的 ARFrame。

#### ARAnchor

ARAnchor 来表示空间中的物理位置。

### AVCaptureSession 

### CMMotionManager



#### ARCamera

每个 ARFrame 都会包含一个 ARCamera。ARCamera 对象表示虚拟摄像头


## ARSessionConfiguration

### ARWorldTrackingSessionConfiguration

## UI required device capability

可以在 app 里设置，这样 app 就只会出现在受支持设备的 App Store 里。

### SceneKit / SpriteKit

### Metal

