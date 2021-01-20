---
layout: post
title:  "Billboard实现"
date:   2020-10-12 10:33:32 +0800
categories: webgl
---

> 此文面向对渲染管线有一定了解并且有threejs使用经验的读者。

Billboard就是一个一直朝向相机的面板，在3D的模型展示中，经常会用billboard来展示模型的一些细节信息和制作可以点击的热点按钮，这篇随笔展示了基于threejs的billboard的实现。

<br/><br/>

#### 一个最简单的billboard
首先利用threejs的PlaneGeometry来创建一个xy平面的正方形片:

{% highlight javascript %}
var geometry = new THREE.PlaneGeometry( 1, 1);
{% endhighlight %}

想要让这个片一直面朝相机有很多方法，我们这里通过改写vertex shader来实现，先看一下常规的vertex shader的写法:

{% highlight glsl %}
void main() {
  gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
}
{% endhighlight %}

这个vertex shader只有一行代码，作用是把每个顶点的坐标从model space变换到clip space。
下面这张图帮助我们理解这一系列变换的含义：
![My helpful screenshot](/assets/transform.png)
如果对这些变换不熟悉的可以参考[Coordinate-Systems](https://learnopengl.com/Getting-started/Coordinate-Systems)

好了，继续考虑我们要解决的问题：**让面片一直朝向相机**。如果对上述的空间变换都理解了，很自然的会想到，我们需要在view space做一些手脚，参考改造后的vertex shader:
{% highlight glsl %}
void main()
{
  vec3 changePos = position;
  // model space下的坐标原点
  vec4 objCenter = vec4(0.0, 0.0, 0.0, 1.0);
  // 原点变换到view space下
  vec4 ori = modelViewMatrix * objCenter;
  // 这一步相当于在view space下，保持位置不变，使面片面朝相机
  changePos += ori.xyz;
  // 继续把修改过的坐标转换到clip space
  gl_Position = projectionMatrix * vec4(changePos, 1.0);
}
{% endhighlight %}

到此为止，最简单的billboard就实现了，完整代码和运行结果如下：
<p class="codepen" data-height="300" data-theme-id="dark" data-default-tab="result" data-user="soapxxx" data-slug-hash="mdEerOK" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="simple billboard">
  <span>See the Pen <a href="https://codepen.io/soapxxx/pen/mdEerOK">
  simple billboard</a> by soap (<a href="https://codepen.io/soapxxx">@soapxxx</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

> * 由于PlaneGeometry创建的片是xy平面的，正好是我们想要的效果。如果这个片或者整个场景是3D美术制作的，这个片有可能是xz或者yz平面的，就需要先把这个片变换到xy平面(changePos.y = changePos.z; 或者 changePos.x = changePos.z;)。
* 由于我们使用了透明贴图，所以material的depthWrite需要设置成false, mesh也需要通过设置renderOrder最后绘制（详见实例中的代码），不然就会出现透明贴图常见的绘制问题。

<br/><br/>

#### 添加呼吸效果
呼吸效果可以是fade in/out，也可以是scale， 都可以通过改造shader实现。

这里先来实现fade in/out的效果。

首先创建一个Clock，用于计时，并把time作为uniform传递给shader：
{% highlight javascript %}
  // Clock
  const clock = new THREE.Clock();

  // Set up uniform.
  const tuniform = {
    time: {
      type: 'f',
      value: 0.1
    },
    _texture: {value: null}
  };

  const animate = function animate() {
    requestAnimationFrame(animate);
    // 每一帧更新time
    tuniform.time.value += clock.getDelta();
    renderer.render(scene, camera);
  };
{% endhighlight %}

然后对fragment shader进行改造：
{% highlight glsl %}
precision mediump float;
varying vec2 vUv;
uniform sampler2D _texture;
uniform float time;

void main(void){
  gl_FragColor = texture2D(_texture, vUv);
  // 通过sin函数来实现淡入淡出的呼吸动画
  gl_FragColor.a *= (abs(sin(time * 1.5)) * 0.8 + 0.2);
}
{% endhighlight %}

我们再来实现scale的呼吸特效。

由于前面已经给shader添加了time，这里只需要对vertex shader进行改造：
{% highlight glsl %}
  varying vec2 vUv;
  uniform float time;

  void main()
  {
    // uv的中心坐标
    vec2 dotCenter = vec2(0.5, 0.5);
    // uv - dotCenter就是从中心点指向uv的向量
    // 通过sin函数， 对这个向量进行缩放，然后在加到中心点坐标上，得出变换后的uv
    vUv = dotCenter + (uv - dotCenter) * (abs(sin(time*2.0)) * 0.5 + 1.0);

    vec3 changePos = position;
    vec4 objCenter = vec4(0.0, 0.0, 0.0, 1.0);
    vec4 ori = modelViewMatrix * objCenter;
    changePos += ori.xyz;
    gl_Position = projectionMatrix * vec4(changePos, 1.0);
  }
{% endhighlight %}
> 这样的做法要求贴图的边缘留有足够的空白，不然会有边缘颜色被拉伸的现象。

scale动画当然也可以通过对mesh的缩放来实现。[tween.js](https://github.com/tweenjs/tween.js/)可以很方便的制作出tween动画，下面给出实现代码：
{% highlight javascript %}
const tweenParams = { scale: 1.0 };
new TWEEN.Tween(tweenParams)
  .to({ scale: 0.5 }, 1000)
  .easing(TWEEN.Easing.Sinusoidal.InOut)
  .onUpdate(() => {
    bb.scale.set(tweenParams.scale, tweenParams.scale, tweenParams.scale);
  })
  .repeat(Infinity)
  .yoyo(true)
  .start();
{% endhighlight %}

通过添加上面的代码，tween动画应该可以正常播放了，但是结果并没有，什么原因呢？我们反过头来看下已经实现的vertex shader，changePos并没有经过modelMatrix的变换，但是tween动画
是通过改变modelMatrix的scale属性实现的，所以tween动画肯定不能播放了。解决这个问题也很简单，把modelMatrix的scale乘到changePos上就可以了，我们假设mesh上的xyz方向的scale
是相同的，修改后的vertex shader如下：
{% highlight glsl %}
varying vec2 vUv;
void main()
{
  vUv = uv;
  // 假设xyz方向的scale相等，这里计算x向量的模长
  float scalingFactor =
    sqrt(modelMatrix[0][0] * modelMatrix[0][0] +
          modelMatrix[0][1] * modelMatrix[0][1] +
          modelMatrix[0][2] * modelMatrix[0][2]);
  // 把scale应用到position上
  vec3 changePos = scalingFactor * position;
  vec4 objCenter = vec4(0.0, 0.0, 0.0, 1.0);
  vec4 ori = modelViewMatrix * objCenter;
  changePos += ori.xyz;
  gl_Position = projectionMatrix * vec4(changePos, 1.0);
}
{% endhighlight %}
> 矩阵中，每个axis的模长代表着这个分量上的scale。这里我们假设三个方向的分量都是相等的，随便计算其中一个的模长既是scale.

到此为止，我们通过三种不同的方式实现了billboard的呼吸动画，下面的实例展示了这三种动画的运行效果:
<p class="codepen" data-height="300" data-theme-id="dark" data-default-tab="result" data-user="soapxxx" data-slug-hash="pobjGyE" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="tween billboard">
  <span>See the Pen <a href="https://codepen.io/soapxxx/pen/pobjGyE">
  tween billboard</a> by soap (<a href="https://codepen.io/soapxxx">@soapxxx</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

> * front-通过fragment shader实现的fade in/out动画
* left-通过vertex shader实现的scale动画
* bottom-通过tween.js实现的对mesh的scale动画

<br/><br/>

#### 固定大小的billboard

这种近大远小的billboard或许并不是设计师想要的效果，下面我们来实现一个不跟随相机远近变化的，固定size的billboard.
近大远小是因为从clip space到ndc space做了透视除法，即每个分量分别除w，所以只要想办法抵消掉这个透视除法对size的影响就可以实现固定size了。
为了固定size，但是不能影响billboard显示的位置，只有在local space下做变换是最合适的.
修改后的vertex shader如下：
{% highlight glsl %}
varying vec2 vUv;
  void main()
  {
    vUv = uv;
    // 假设xyz方向的scale相等，这里计算x向量的模长
    float scalingFactor =
      sqrt(modelMatrix[0][0] * modelMatrix[0][0] +
           modelMatrix[0][1] * modelMatrix[0][1] +
           modelMatrix[0][2] * modelMatrix[0][2]);
    // 把scale应用到position上
    vec3 changePos = scalingFactor * position;
    vec4 objCenter = vec4(0.0, 0.0, 0.0, 1.0);
    vec4 ori = modelViewMatrix * objCenter;
    // 这里取中心点在clip space下的w，因为所有顶点同中心点都在同一xy平面，w是相等的
    // 0.2是一个系数，决定了最终显示的大小，这个系数根据实际情况调整为合适的值即可
    float w = 0.2 * (projectionMatrix * modelViewMatrix * objCenter).w;
    // 在加ori之前乘w，这样只会影响几何的size，不会影响它在view space下的位置
    changePos *= w;
    changePos += ori.xyz;
    gl_Position = projectionMatrix * vec4(changePos, 1.0);
  }
{% endhighlight %}

下面实例展示了固定size的效果：
<p class="codepen" data-height="300" data-theme-id="dark" data-default-tab="result" data-user="soapxxx" data-slug-hash="YzGgOox" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="tween billboard">
  <span>See the Pen <a href="https://codepen.io/soapxxx/pen/YzGgOox">
  tween billboard</a> by soap (<a href="https://codepen.io/soapxxx">@soapxxx</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

以上，展示了billboard的几种常用实现方式。另外，如果想要实现点击billboard，触发点击事件，上面这种做法用来跟ray求交的几何是一个平面，想象一下，如果不是从平面的正前方点击，就会很难击中。所以这里需要创建SphereGeometry把billboard包裹起来，用来触发点击事件，把sphere的material的visiable设置成false可以实现隐藏mesh，但是可以参与求交，这里就不赘述了，如果有需要可以自行实现。
