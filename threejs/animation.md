# Three.js的Animation框架

#### 为什么是Three.js?

Three.js是优秀出色的网页端WebGL框架，Javascript代码实现简单，框架层级简单，易学易懂。它的Animation框架和Unity/Unreal Engine4是相近的。支持fbx, glb, gltf, obj模型加载，所以完全能支持Adobe的Mixamo导出的fbx模型在网页端动画效果展示。

> Within the three.js animation system you can animate various properties of your models: the bones of a skinned and rigged model, morph targets, different material properties (colors, opacity, booleans), visibility and transforms. The animated properties can be faded in, faded out, crossfaded and warped. The weight and time scales of different simultaneous animations on the same object as well as on different objects can be changed independently. Various animations on the same and on different objects can be synchronized.
>
> To achieve all this in one homogeneous system, the three.js animation system [has completely changed in 2015](https://github.com/mrdoob/three.js/issues/6881) (beware of outdated information!), and it has now an architecture similar to Unity/Unreal Engine 4. 

[Animation system – three.js docs (threejs.org)](https://threejs.org/docs/index.html#manual/en/introduction/Animation-system)

#### 总览

![animation uml](https://raw.githubusercontent.com/fanghaosama/fanghaosama.github.io/gh-pages/threejs/animate.png)

动画的实现在于模型中顶点的变化。在每一个关键帧（Key Frame）中，都记录着在什么时间（time），顶点做什么变换，变成什么数值（value），这个过程使用什么插值运算（Interpolation）,通过插值运算，来补齐关键帧至关键帧之间的顶点变化。这一系列的关键帧，作为一个AnimationClip。

AnimationMixer顾名思义就是总入口将模型Group的根节点和Clip中的KeyFrameTrack进行一一对应，并返回action对象进行总控制。

AnimationAction对象为实际执行控制对象，提供控制，播放，暂停，更新等方法，并保存所有的时间变化，weight，关键帧对应的Interpolation，进行调用。

PropertyBinding通过KeyFrameTrack中的name，找到相对应的AnimationObjectGroup中的子节点（可以是Skeleton中的一个Bone），将节点与关键帧名称进行绑定。

PropertyMixer根据KeyFrameTrack的从插值类型，添加相对应的插值计算方法。

#### Usage

```javascript
const loader = new FBXLoader();
loader.load( 'models/fbx/Samba Dancing.fbx', function ( object ) {
    // new a mixer with the model, the mixer will create an action object.
    mixer = new THREE.AnimationMixer( object );
    // add the clip into Action
    const action = mixer.clipAction( object.animations[ 0 ] );
    // action provides methods to control start or stop of the animation.
    action.play();

    scene.add( object );
} );
```

```javascript
function animate() {
	// use browser api to run animate method.
    requestAnimationFrame( animate );
	// get the unit of time.
    const delta = clock.getDelta();
	// update the Animation object status in delta time.
    if ( mixer ) mixer.update( delta );
	// render the scene.
    renderer.render( scene, camera );
    stats.update();
}
```

![bot](https://raw.githubusercontent.com/fanghaosama/fanghaosama.github.io/gh-pages/threejs/bot.gif)

#### Debug

##### 模型解析后的数据结构

![data1](https://raw.githubusercontent.com/fanghaosama/fanghaosama.github.io/gh-pages/threejs/data1.png)

![data2](https://raw.githubusercontent.com/fanghaosama/fanghaosama.github.io/gh-pages/threejs/data2.png)

FBXLoader加载后的object对象类型为Group，这一个Demo的主要动画，就是animations字段数组中的AnimationClip，另外skeleton信息以树状结构存储在children之中，可以看到其中一个Bone名称为‘mixamorigHips’

![data1-1](https://raw.githubusercontent.com/fanghaosama/fanghaosama.github.io/gh-pages/threejs/data1-1.png)

AnimationClip的tracks字段存储着不同类型的KeyFrameTrack，上图例子中，这个是四元数的KeyFrameTrack，对应的Bone是‘mixamorigHips’，数据记录的是630个时间点，’Hip‘旋转的四元数，它们之间通过‘InterpolantFactoryMethodLiner'也就是线性变换进行插值。

##### 源码分析

1.初始化

```javascript
// clipAction
// omit a lot assignment here......

// allocate all resources required to run it
const newAction = new AnimationAction( this, clipObject, optionalRoot, blendMode );

this._bindAction( newAction, prototypeAction );
// then return newAction.
```

```javascript
// _bindAction
// omit a lot assignment here......

for ( let i = 0; i !== nTracks; ++ i ) {
    //KeyFrameTrack
    const track = tracks[ i ],
        trackName = track.name;

    let binding = bindingsByName[ trackName ];
    if ( binding !== undefined ) {
        // bindings is the PropertyMixer Array held in AnimationAction
        bindings[ i ] = binding;
    } else {
        binding = bindings[ i ];
        if ( binding !== undefined ) {
            // existing binding, make sure the cache knows
            if ( binding._cacheIndex === null ) {
                ++ binding.referenceCount;
                this._addInactiveBinding( binding, rootUuid, trackName );
            }
            continue;
        }
		// convert the name in string like 'mixamorigHips.quaternion' into an object which contains the information with 'mixamorigHips' and 'quaternion' and other flags.
        const path = prototypeAction && prototypeAction.
            _propertyBindings[ i ].binding.parsedPath;
		// create PropertyBinding and PropertyMixer
        binding = new PropertyMixer(
            PropertyBinding.create( root, trackName, path ),
            track.ValueTypeName, track.getValueSize() );

        ++ binding.referenceCount;
        this._addInactiveBinding( binding, rootUuid, trackName );
		// give the new PropertyMixer to the Array held in AnimationAction
        bindings[ i ] = binding;
    }
	// interpolants is the Array held in AnimationAction
    interpolants[ i ].resultBuffer = binding.buffer;
}
```

```javascript
// PropertyBinding.create
// just new a PropertyBinding
this.path = path;
this.parsedPath = parsedPath || PropertyBinding.parseTrackName( path );
// find the Bone by the track name
this.node = PropertyBinding.findNode( rootNode, this.parsedPath.nodeName ) || rootNode;
this.rootNode = rootNode;
// initial state of these methods that calls 'bind'
this.getValue = this._getValue_unbound;
this.setValue = this._setValue_unbound;
```

```javascript
// new PropertyMixer
// based on different track types, assign different MixFunctions to the attributes
switch ( typeName ) {
    case 'quaternion':
        mixFunction = this._slerp;
        mixFunctionAdditive = this._slerpAdditive;
        setIdentity = this._setAdditiveIdentityQuaternion;
        this.buffer = new Float64Array( valueSize * 6 );
        this._workIndex = 5;
        break;
    case 'string':
    case 'bool':
        mixFunction = this._select;
        // Use the regular mix function and for additive on these types,
        // additive is not relevant for non-numeric types
        mixFunctionAdditive = this._select;
        setIdentity = this._setAdditiveIdentityOther;
        this.buffer = new Array( valueSize * 5 );
        break;
    default:
        mixFunction = this._lerp;
        mixFunctionAdditive = this._lerpAdditive;
        setIdentity = this._setAdditiveIdentityNumeric;
        this.buffer = new Float64Array( valueSize * 5 );
}
this._mixBufferRegion = mixFunction;
this._mixBufferRegionAdditive = mixFunctionAdditive;
this._setIdentity = setIdentity;
```

2.play

```javascript
// play
play() {
    // still set some states, and get ready.
    this._mixer._activateAction( this );
    return this;
}
```

3.update

```javascript
// AnimationAction _update function 
_update( time, deltaTime, timeDirection, accuIndex ) {

    // called by the mixer
    if ( ! this.enabled ) {
        // call ._updateWeight() to update ._effectiveWeight
        this._updateWeight( time );
        return;
    }

    const startTime = this._startTime;
    if ( startTime !== null ) {
        // check for scheduled start of action
        const timeRunning = ( time - startTime ) * timeDirection;
        if ( timeRunning < 0 || timeDirection === 0 ) {
            return; // yet to come / don't decide when delta = 0
        }
        // start
        this._startTime = null; // unschedule
        deltaTime = timeDirection * timeRunning;
    }

    // apply time scale and advance time
    deltaTime *= this._updateTimeScale( time );
    const clipTime = this._updateTime( deltaTime );

    // note: _updateTime may disable the action resulting in
    // an effective weight of 0
    const weight = this._updateWeight( time );
    if ( weight > 0 ) {
        const interpolants = this._interpolants;
        const propertyMixers = this._propertyBindings;
        switch ( this.blendMode ) {
            case AdditiveAnimationBlendMode:
                for ( let j = 0, m = interpolants.length; j !== m; ++ j ) {

                    interpolants[ j ].evaluate( clipTime );
                    propertyMixers[ j ].accumulateAdditive( weight );

                }
                break;
			// Important!!, AnimationClip BlendMode with a value 2500 is in this case.
            case NormalAnimationBlendMode:
            default:
                for ( let j = 0, m = interpolants.length; j !== m; ++ j ) {
                    // based on the clipTime, calculate the values in different interpolants, store the data in resultBuffer which is the same object with the buffer in PropertyMixer
                    interpolants[ j ].evaluate( clipTime );
                    // the Mixer will accumulate the values in Buffer by weight.
                    propertyMixers[ j ].accumulate( accuIndex, weight );
                }
        }
    }
}
```

#### 结束语

这就是整个Animation框架的大致流程，three.js的代码还有改进之处，特别是AnimationAction和AnimationMixer之间的相互引用的关系和方法写法，尽管如此，它的实现还是非常值得学习的，以后有机会再来更深入讨论插值算法。

