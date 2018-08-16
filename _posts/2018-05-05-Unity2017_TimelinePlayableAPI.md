---
layout: post
title:  "探索TimelinePlayableAPI，让Timeline为所欲为"
date:   2018-05-05 18:00:00 +0800
categories: UnityEngine 
tags: Unity2017 NewFeature Timeline PlayableAPI
---

## 简介
之前快速地学习了一下Timeline，需要详细回顾的兄弟可以回头看我的上一篇，[快速了解Timeline](https://aabao.github.io/Unity2017_Timeline_Quickview/)。
这里我们做快速的回顾，Timeline主要分为`TimelineInstance`和`TimelineAsset`，可以用Timeline PlayableDirector控制播放哪一个TimelineAsset
今天我们再来看看Timeline中的Track，并且动手实现一个老年迪斯科场景。

![预览](http://oxujermt3.bkt.clouddn.com/image/blog/201805051800/Disco效果.gif)

## 一图流

![TimelinePlayableAPI-Xmind](http://oxujermt3.bkt.clouddn.com/image/blog/201805051800/TimelinePlayableAPI_Xmind2.png)

## Track
Timeline可以添加很多Track来对不同的对象做控制。
* Animation Track

    在Clip上控制动画，可以在有Animator组件但是没有AnimatorController的情况下直接播放动画。

* Audio Track

    在Clip上控制音频播放。

* Activtion Track

    在Clip上控制物体是否在场景中Active，GameObject.SetActive()的作用

* Control Track

    在Clip上可以控制和时间相关的元素，可以控制粒子系统，克隆物体，控制另一个timeline等。厉害的是我们放在Control Track上的例子特效，可以随意地拖动Timeline来预览效果，对特效兄弟来说是应该很强大很爽的地方。

* Playable Track

    控制继承自BasicPlayableBehaviour的clip
    （在2017.4.0f1版本里原先的BasicPlayableBehaviour因为性能原因已经被弃用，建议采用PlayableAsset和PlayableBehaviour）


## 如何有了这篇文章
虽然上面提供的几个track已经有很强大的功能了，但是实际操作中我们会发现这些远远不够，官方的Mannul上也没有更多的介绍，有效的资料是少之又少。那么官方演示的demo里的对话系统是怎么通过Timeline实现的呢？
于是我开始寻找，在Youtube上找到了Unite Europe2017上的演讲视频，[Extending Timeline with your own playables](https://www.youtube.com/watch?v=uBPRfcox5hE&t=2331s)。
研究了一番，里面的例子就是LightControl将自己的理解分享给大家。让大家都能在Timeline里为所欲为。

## 深度理解Timeline

![视频截图](http://oxujermt3.bkt.clouddn.com/image/blog/201805051800/timelineisGraph.png)

Timeline是建立在[Playables API](https://docs.unity3d.com/Manual/Playables.html)上的PlayableGraph系统，在这套系统中Timeline相当于是一个函数的作用，我们从Assset传入InputData，Timeline处理后输出OutputData到对应的Component。
这套系统中有4个关键部分，`Data`，`Clip`，`Mixer`，`Track`
我希望我的这张图可以帮助大家快速对应和理解这4个部分。

Timeline也一个Templete，可以方便快速地被复用。

* The Data

    用来存放数据，需要被序列化。继承自PlayableBehaviour

    ```csharp
    [System.Serializable]
    public class LightData : PlayableBehaviour{
	
        public float range;
        public Color color;
        public float intensity;
        [HideInInspector]
        public Transform lookTarget;

        public override void OnPlayableCreate(Playable playable){

            var duration = playable.GetDuration();

            if(Mathf.Approximately((float)duration, 0)){
                throw new UnityException("A Clip Cannot have a duration of zero");
            }

        }
    }
    ```

* The Clip

    也就是asset，继承自PlayableAsset。因为是asset，所以不能直接与场景中的物体关联，需要关联场景中的物体时要用ExposedReference<T>来声明，
    并且通过Resolve方法赋值。

    ```csharp
    [System.Serializable]
    public class LightClip : PlayableAsset {

        public LightData templete = new LightData();
        public ExposedReference<Transform> lookTarget;

        // Factory method that generates a playable based on this asset
        public override Playable CreatePlayable(PlayableGraph graph, GameObject go) {
            var playable = ScriptPlayable<LightData>.Create(graph, templete); 
            LightData clone = playable.GetBehaviour();
            clone.lookTarget = lookTarget.Resolve(graph.GetResolver());
            return playable;
        }

    }
    ```

* The Mixer

    Mixer控制当前Track中所有的clip的行为，根据每个clip不同的`输入权重`，计算需要的结果。 Mixer在Timeline中算是最重要的部分了。

    ```csharp
    public class LightMixer : PlayableBehaviour {

        private float m_defaultRange = 1;
        private float m_defaultIntensity = 1;
        private Color m_defaultColor = Color.white;
        private Quaternion m_defaultRotation = Quaternion.identity;
        private Light m_trackBinding;		

        private bool m_isFirstFrameProcess = true;

        // Called when the owning graph starts playing
        public override void OnGraphStart(Playable playable) {
        }

        // Called when the owning graph stops playing
        public override void OnGraphStop(Playable playable) {

            m_isFirstFrameProcess = false;
            if(m_trackBinding == null)
                return;
            m_trackBinding.range = m_defaultRange;
            m_trackBinding.color = m_defaultColor;
            m_trackBinding.intensity = m_defaultIntensity;
            m_trackBinding.transform.rotation = m_defaultRotation;

        }

        // Called when the state of the playable is set to Play
        public override void OnBehaviourPlay(Playable playable, FrameData info) {
        }

        // Called when the state of the playable is set to Paused
        public override void OnBehaviourPause(Playable playable, FrameData info) {
            
        }

        // Called each frame while the state is set to Play
        public override void PrepareFrame(Playable playable, FrameData info) {
        }

        public override void ProcessFrame(Playable playable, FrameData info, object playerData){

            m_trackBinding = playerData as Light;		
            if(m_trackBinding == null)
                return;

            if(!m_isFirstFrameProcess){
                m_isFirstFrameProcess = true;
                m_defaultRange = m_trackBinding.range;
                m_defaultColor = m_trackBinding.color;
                m_defaultIntensity = m_trackBinding.intensity;
                m_defaultRotation = m_trackBinding.transform.rotation;
            }

            int inputCount = playable.GetInputCount();
            float blendRange = 0;
            Color blendColor = Color.clear;
            float blendIntensity = 0;
            Quaternion blendRotation = Quaternion.identity;

            float totalWeight = 0;
            float greatestWeight = 0;
            int currentInputs = 0;

            for(int i = 0; i < inputCount; i++){
                ScriptPlayable<LightData> playableInput = (ScriptPlayable<LightData>)playable.GetInput(i);
                LightData input = playableInput.GetBehaviour();

                float inputWeight = playable.GetInputWeight(i);
                blendRange += input.range * inputWeight;
                blendColor += input.color * inputWeight;
                blendIntensity += input.intensity * inputWeight;
                if(input.lookTarget != null && inputWeight > 0)
                    blendRotation *= Quaternion.LookRotation((input.lookTarget.position - m_trackBinding.transform.position) * inputWeight); 
                totalWeight += inputWeight;

                if(inputWeight > greatestWeight)
                    greatestWeight = inputWeight;
                
                if(!Mathf.Approximately(inputWeight, 0f))
                    currentInputs++;
            }

            m_trackBinding.range = blendRange + m_defaultRange * (1 - totalWeight);
            m_trackBinding.color = blendColor + m_defaultColor * (1 - totalWeight);
            m_trackBinding.intensity = blendIntensity + m_defaultIntensity * (1 - totalWeight);
            m_trackBinding.transform.rotation =  blendRotation;
        }

    }
    ```

* The Track

    把Mixer计算的结果输出，可以指定输出到绑定的物体上。继承自TrackAsset。有好几个特性可以绑定，
    
    * TrackColor : 定义在编辑器中Track的标识颜色

    * BindingObejctType : 当前Track上的Mixer输出后的数据要传递的对象类型

    * TrackClipType : 当前Track上的Clip的类型

    ```csharp
    [TrackColor(1f, 0f, 0f)]
    [TrackClipType(typeof(LightClip))]
    [TrackBindingType(typeof(Light))]
    public class LightContorlTrack : TrackAsset {

        public override Playable CreateTrackMixer(PlayableGraph graph, GameObject go, int inputCount){

            return ScriptPlayable<LightMixer>.Create(graph, inputCount);

        }

    }
    ```

## 实践
通过上面的探究，我们要做老年disco效果，那就来定制一个自己的track，专门用来控制Light，我们叫LightControl

1. 创建一个新的场景添加两个角色，一红一蓝
2. 新建一个ParticleSytem，烘托一下Disco的气氛 
3. 创建一个新的Timeline，DiscoTimeline
4. 在DiscoTimeline上新建2个AnimationTrack，一个控制蓝角色的动画，一个控制红角色的动画
5. 在DiscoTimeline新建一个ControlTrack，控制刚才创建的ParticleSystem
6. 在场景中新建一个SpotLight和一个PointLight
7. 在DiscoTimeline中新建一个TrackGroup，命名为PlayableLight
8. 在DiscoTimeline中新建我们的LightControl Track，将SpotLight与这个Track绑定，命名为SpotLight
9. 在DiscoTimeline中新建我们的LightControl Track，将PointLight与这个Track绑定，命名为PointLight
10. 将我们自定义的Clip添加2段到SpotLightTrack上，在一个Clip上的LookAtTarget上绑定蓝角色，另一个Clip上的LookAtTarget绑定到红角色
11. 将我们自定义的Clip添加2段到PointLightTrack上，设置变换的灯光颜色和强度。

    最后的Timeline就是这样

    ![Timeline结果](http://oxujermt3.bkt.clouddn.com/image/blog/201805051800/discotimeline.png)

Play and Enjoy It


## 结尾
关于Playable API比较有趣的是，Playable使用C#结构体而非C++对象来保存对象。使用结构体的目的是避免分配GC所需的内存。这样用起来可能稍微有点复杂，但由于该API承载了未来的很多功能，所以必须注重性能问题。

掌握了Timeline的系统设计理念后，我们基本上就能掌握Timeline，实现自己想要的效果了，希望大家能打开脑洞，用Timeline创造出更多好玩的东西来。
另外视频中提到的PreviewMode我还没有非常明白，就先不写出来了，如果有兄弟知道的话，希望你能不吝赐教。

### 这是[Demo的Github地址](https://github.com/aaBaO/DemoRepository)欢迎大家Fork过去参考TimelinePlayableAPI
