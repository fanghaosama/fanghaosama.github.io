# Three.js的Animation框架

为什么是Three.js?

Three.js是优秀出色的网页端WebGl框架，Javascript代码实现简单，框架层级简单，易学易懂。它的Animation框架和Unity/Unreal Engine4是相近的。支持fbx, glb, gltf, obj模型加载，所以完全能支持Adobe的Mixamo导出的fbx模型在网页端动画效果展示。

> Within the three.js animation system you can animate various properties of your models: the bones of a skinned and rigged model, morph targets, different material properties (colors, opacity, booleans), visibility and transforms. The animated properties can be faded in, faded out, crossfaded and warped. The weight and time scales of different simultaneous animations on the same object as well as on different objects can be changed independently. Various animations on the same and on different objects can be synchronized.
>
> To achieve all this in one homogeneous system, the three.js animation system [has completely changed in 2015](https://github.com/mrdoob/three.js/issues/6881) (beware of outdated information!), and it has now an architecture similar to Unity/Unreal Engine 4. 

[Animation system – three.js docs (threejs.org)](https://threejs.org/docs/index.html#manual/en/introduction/Animation-system)

实例

总览

![animate](.\animate.png)

