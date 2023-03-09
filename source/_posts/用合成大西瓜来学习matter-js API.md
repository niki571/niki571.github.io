---
title: 用合成大西瓜来学习 matter.js API
date: 2023-03-08 14:49:40
tags: [canvas, 2D]
categories: 可视化
---

# 前言

新的一年想业余时间更多的做一些关于可视化的东西，构思了一个关于沙漏的新项目，发现想实现重力感应需要用到物理引擎。而 Matter.js 就是一个比较轻量级的物理引擎。今天就用《合成大西瓜》这个游戏来快速了解下 Matter.js 的 API。

Matter.js 是一个 2D 刚体物理引擎。

Matter.js 引擎内置了物体的运动规律和碰撞检测，因此通过它来实现这个游戏，也仅仅就是一个 API 使用的过程。

<!-- more -->

# 核心功能

在实现的过程中，我将功能分成了五部分，分别为：场景初始化、创建小球、给小球添加事件、碰撞检测以及游戏结束的检测。

核心功能仅展示核心代码，完整代码在文末附出。

## 场景初始化

场景初始化这部分，主要是学习 Matter.js 的大框架，按照官网的指引分别配置 `Engine` 、`Render` 和 `World`：

- `Engine` 是 Matter.js 中物理引擎的配置部分，初始化它，也就给物体添加好了引擎；
- `Render` 和其他引擎类似，是画布的渲染，画布的尺寸、背景颜色等绘制相关的内容再次配置；
- `World` 在 Matter.js 中是类似与舞台，所有要展现出来的内容，都需要添加到 `World` 当中。

```javascript
// Engine 初始化
this.engine = Matter.Engine.create({
  enableSleeping: true, // 在游戏结束检测的时候，用到该 sleep 功能，enableSleeping 为 true 可以检测到小球停止运动的状态，从而方便进行游戏结束的检测。
});

// World 初始化
this.world = this.engine.world;
this.world.bounds = {
  min: { x: 0, y: 0 },
  max: { x: window.innerWidth, y: window.innerHeight },
};

// Render 初始化
this.render = Matter.Render.create({
  canvas: document.getElementById("canvas"),
  engine: this.engine,
  options: {
    width: window.innerWidth,
    height: window.innerHeight,
    wireframes: false, // 设为 false 后，才可以展现出添加到小球的纹理
    background: "#ffe89d",
    showSleeping: false, // 隐藏 sleep 半透明的状态
  },
});

// 这里使用内置的长方形物体创建了游戏的墙壁和地面
// 创建地面
const ground = Matter.Bodies.rectangle(
  window.innerWidth / 2,
  window.innerHeight - 120 / 2,
  window.innerWidth,
  120,
  {
    isStatic: true, // true 可将物体作为墙壁或者地面，将不会有重力等物理属性
    render: {
      fillStyle: "#7b5438", // 地面背景颜色
    },
  }
);

// 左墙
const leftWall = Matter.Bodies.rectangle(
  -10 / 2,
  canvasHeight / 2,
  10,
  canvasHeight,
  { isStatic: true }
);

// 右墙
const rightWall = Matter.Bodies.rectangle(
  10 / 2 + psdWidth,
  canvasHeight / 2,
  10,
  canvasHeight,
  { isStatic: true }
);

// 将创建的物体添加到 World
Matter.World.add(this.world, [ground, leftWall, rightWall]);

// 运行引擎与渲染器
Matter.Engine.run(this.engine);
Matter.Render.run(this.render);
```

## 创建小球

在游戏当中，小球默认悬浮在页面最上面，点击或者左右滑动到指定位置，小球脱落，脱落后延时一段时间再次创建一个小球。

在这里，可以将创建一个方法：创建一个球形物体，指定他出现的位置，并赋予贴图。

```javascript
// 球体半径的合集
const radius = [52/2, 80/2, 108/2, 118/2, 152/2, 184/2, 194/2, 258/2, 308/2, 310/2, 408/2];

// 小球纹理数组
const assets = ['./assets/1.png',.....];

// 小球出现的次数，可以根据次数的累计增加游戏难度，或者用于计算分数。
let circleAmount = 0;

// 添加球体
addCircle(){
// 随机一个半径
  const radiusTemp = radius.slice(0, 6);
  const index = circleAmount === 0 ? 0 : (Math.random() \* radiusTemp.length | 0);
  const circleRadius = radiusTemp[index];

// 创建一个球体
  this.circle = Matter.Bodies.circle(
    window.innerWidth /2, // 小球的 x 坐标，这里是根据小球的圆心来定位的
    circleRadius + 30, // 小球的 y 坐标，将初始化的小球安置在水平居中，距离顶部 30 像素的位置
    circleRadius, {
      isStatic: true, // 首先设置为 true ，在触发事件之后再改为 false ，给小球添加下落动作
      restitution: 0.2, // 设置小球弹性
      render: {
        sprite: {
          texture: assets[index], // 给小球设置纹理
        }
      }
    }
  );

  // 将小球添加到 World
  Matter.World.add(this.world, this.circle);

  // 游戏状态检测，后续会说明
  this.gameProgressChecking(this.circle);
  circleAmount++;
}
```

## 给小球添加事件

初始化后的小球，有两种情况下落，一种是点击任意一处，根据 x 坐标下落，二是手指触摸，滑动小球到指定位置，手指抬起下落。

```javascript
// 使用 Matter.js 内置的 MouseConstraint 和 Events 就可以实现 touch 事件
const mouseconstraint = Matter.MouseConstraint.create(this.engine);

// touchmove 事件
Matter.Events.on(mouseconstraint, "mousemove", (e)=>{
  if(!this.circle || !this.canPlay) return; // this.canPlay 判断游戏是否结束
  this.updateCirclePosition(e); // 在 touchmove 中更新小球的 x 坐标
})

// touchend 事件
Matter.Events.on(mouseconstraint, "mouseup", (e)=>{
  if(!this.circle || !this.canPlay) return;
  this.updateCirclePosition(e);
  Matter.Sleeping.set(this.circle, false); // 接触小球的 sleep 模式，以便添加物理属性
  Matter.Body.setStatic(this.circle, false ); // 给小球激活物理属性，小球会因为重力自动落下
  this.circle = null;
  setTimeout(()=>{ // 延迟 1s 后再次创建小球
    this.addCircle();
  }, 1000);
});

// 更新小球的 x 坐标
updateCirclePosition(e){
  const xTemp = e.mouse.absolute.x;
  const radius = this.circle.circleRadius;
  Matter.Body.setPosition(this.circle, {x: xTemp < radius ?     radius : xTemp + radius > psdWidth ? psdWidth - radius : xTemp, y: radius + 30});
}
```

## 碰撞检测

游戏中最吸引人的部分，就是两个相同的水果接触会变成一个更大的水果。在此功能部分，我们以来 Matter.js 内置的碰撞检测，只需判断碰撞的两个小球半径是否一致即可，如果半径一致，就变成一个更大半径的小球。

```javascript
Matter.Events.on(this.engine, "collisionStart", e => this.collisionEvent(e)); // 下落的小球刚碰撞在一起的事件
Matter.Events.on(this.engine, "collisionActive", e => this.collisionEvent(e)); // 其他被动的小球相互碰撞的事件

collisionEvent(e){
  if(!this.canPlay) return;
  const { pairs } = e; // pairs 为所有小球碰撞的集合，通过遍历该集合中参与碰撞的小球半径，就完成了逻辑判断
  Matter.Sleeping.afterCollisions(pairs); // 将参与碰撞的小球从休眠中激活

  for (let i = 0; i < pairs.length; i++ ) {
    const {bodyA, bodyB} = pairs[i]; // 拿到参与碰撞的小球
    if (bodyA.circleRadius && bodyA.circleRadius == bodyB.circleRadius) { // 小球半径一致，变成更大的小球
      const { position: { x: bx, y: by }, circleRadius, } = bodyA; // 获取两个相同半径的小球，取中间位置合成大球
      const { position: { x: ax, y: ay } } = bodyB;

      const x = (ax + bx) / 2;
      const y = (ay + by) / 2;

      const index = radius.indexOf(circleRadius)+1;

      const circleNew = Matter.Bodies.circle(x, y, radius [index],{ // 创建大的球
        restitution: 0.2,
        render: {
          sprite: {
            texture: this.assets[index],
          }
        }
      });

      Matter.World.remove(this.world, bodyA); // 移除两个碰撞的小球
      Matter.World.remove(this.world, bodyB);
      Matter.World.add(this.world, circleNew); // 将生成的大球加入到 World
      this.gameProgressChecking(circleNew); // 判断游戏的状态
    }
  }
}
```

## 游戏结束的检测

Matter.js 中提供了小球是否运动停止，也就是`Sleep`状态，我们只需判断最近添加到`World`的小球位置，是否溢出了游戏区域即可，如果 y 坐标溢出了游戏区域，则游戏就结束了。

```javascript
// gameProgressChecking 在上文小球开始掉落的时候开始触发
gameProgressChecking(body){
  Matter.Events.on(body, 'sleepStart', (event)=> {
    if (!event.source.isStatic && event.source.position.y <= 300) { // 如果小球静止时，y 坐标移除游戏区域，游戏结束
      this.gameOver();
    }
  })
}
```

# 总结

以上，就完成了《合成大西瓜》的核心功能，借助 Matter.js ，让我们节省了大量的时间去研究小球间的物理关系，让我们站在巨人的肩膀上快速的完成了游戏的开发。

```javascript
const Engine = window['Matter'].Engine,
Render = window['Matter'].Render,
World = window['Matter'].World,
Bodies = window['Matter'].Bodies,
Body = window['Matter'].Body,
MouseConstraint = window['Matter'].MouseConstraint,
Sleeping = window['Matter'].Sleeping,
Events = window['Matter'].Events;

// 基本数据
const psdWidth = 750,
canvasHeight = window.innerHeight \* psdWidth / window.innerWidth,
radius = [52/2, 80/2, 108/2, 118/2, 152/2, 184/2, 194/2, 258/2, 308/2, 310/2, 408/2];

export default class MatterClass{
  constructor(prop) {
    this.canvas = prop.canvas;
    this.assets = prop.assets;
    this.gameOverCallback = prop.gameOverCallback;
    this.circle = null;
    this.circleAmount = 0;
    this.canPlay = true;

    this.init();
    this.addCircle();
    this.addEvents();
  }

  // 场景初始化
  init() {
    this.engine = Engine.create({
        enableSleeping: true
    });
    this.world = this.engine.world;
    this.world.bounds = { min: { x: 0, y: 0}, max: { x: psdWidth, y: canvasHeight } };
    this.render = Render.create({
      canvas: this.canvas,
      engine: this.engine,
      options: {
        width: psdWidth,
        height: canvasHeight,
        wireframes: false,
        background :"#ffe89d",
        showSleeping: false,
      },
    });

    const ground = Bodies.rectangle(psdWidth / 2, canvasHeight - 120 / 2, psdWidth, 120, { isStatic: true,
      render: {
        fillStyle: '#7b5438',
      }
    });
    const leftWall = Bodies.rectangle(-10/2, canvasHeight/2, 10, canvasHeight, { isStatic: true });
    const rightWall = Bodies.rectangle(10/2 + psdWidth, canvasHeight/2, 10, canvasHeight, { isStatic: true });
    World.add(this.world, [ground, leftWall, rightWall]);

    Engine.run(this.engine);
    Render.run(this.render);
  }

  // 添加球体
  addCircle(){
    const radiusTemp = radius.slice(0, 6);
    const index = this.circleAmount === 0 ? 0 : (Math.random() * radiusTemp.length | 0);
    const circleRadius = radiusTemp[index];
    this.circle = Bodies.circle(psdWidth /2, circleRadius + 30, circleRadius, {
      isStatic: true,
      restitution: 0.2,
      render: {
        sprite: {
          texture: this.assets[index],
        }
      }
    });
    World.add(this.world, this.circle);
    this.gameProgressChecking(this.circle);
    this.circleAmount++;
  }

  // 添加事件
  addEvents(){
    const mouseconstraint = MouseConstraint.create(this.engine);
    Events.on(mouseconstraint, "mousemove", (e)=>{
      if(!this.circle || !this.canPlay) return;
      this.updateCirclePosition(e);
    })
    Events.on(mouseconstraint, "mouseup", (e)=>{
      if(!this.circle || !this.canPlay) return;
      this.updateCirclePosition(e);
      Sleeping.set(this.circle, false);
      Body.setStatic(this.circle, false );
      this.circle = null;
      setTimeout(()=>{
        this.addCircle();
      }, 1000);
    });

    Events.on(this.engine, "collisionStart", e => this.collisionEvent(e));
    Events.on(this.engine, "collisionActive", e => this.collisionEvent(e));
  }

  // 碰撞检测
  collisionEvent(e){
    if(!this.canPlay) return;
    const { pairs } = e;
    Sleeping.afterCollisions(pairs);
    for (let i = 0; i < pairs.length; i++ ) {
      const {bodyA, bodyB} = pairs[i];
      if(bodyA.circleRadius && bodyA.circleRadius == bodyB.circleRadius) {
        const { position: { x: bx, y: by }, circleRadius, } = bodyA;
        const { position: { x: ax, y: ay } } = bodyB;

        const x = (ax + bx) / 2;
        const y = (ay + by) / 2;

        const index = radius.indexOf(circleRadius)+1;

        const circleNew = Bodies.circle(x, y, radius[index],{
          restitution: 0.2,
          render: {
            sprite: {
              texture: this.assets[index],
            }
          }
        });

        World.remove(this.world, bodyA);
        World.remove(this.world, bodyB);
        World.add(this.world, circleNew);
        this.gameProgressChecking(circleNew);
      }
    }
  }

  // 更新小球位置
  updateCirclePosition(e){
    const xTemp = e.mouse.absolute.x * psdWidth / window.innerWidth;
    const radius = this.circle.circleRadius;
    Body.setPosition(this.circle, {x: xTemp < radius ? radius : xTemp + radius > psdWidth ? psdWidth - radius : xTemp, y: radius + 30});
  }

  // 游戏状态检测
  gameProgressChecking(body){
    Events.on(body, 'sleepStart', (event)=> {
      if (!event.source.isStatic && event.source.position.y <= 300) {
        this.gameOver();
      }
    })
  }

  // 游戏结束
  gameOver(){
    this.canPlay = false;
    this.gameOverCallback();
  }
}

```

使用方法：

```javascript
import MatterClass from './matter.js';
const matterObj = new MatterClass({
  canvas: document.getElementById('canvas'), // canvas 元素
  assets: ['../assets/0.png',...], // 纹理合集
  gameOverCallback: () => { // 失败回调
  }
});
```
