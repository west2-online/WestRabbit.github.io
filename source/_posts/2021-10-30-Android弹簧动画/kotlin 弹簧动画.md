---
title: Android 弹簧动画(kotlin)
date: 2021.10.26
tags: 
    - 19级
    - Android
author: Hintt
---

## 1.简介

在Android中，动画（animations）可以添加视觉提示，向用户通知应用中的动态。当界面状态发生改变时（例如有新内容加载或有新操作可用时），动画尤其有用。动画还为应用增加了优美的外观，使其拥有更高品质的外观和风格。

而Android 根据用户需要，提供不同的动画 API：

- Animate bitmaps：一般运用于为静态图片添加动画
- Animate UI visibility and motion：一般用于为界面可见性和动作添加动画
- Physics-based motion：基于物理特性为View添加动画（如弹簧动画、投掷动画）
- Animate layout changes ：为布局更改添加动画

本文将介绍如何用弹簧动画来让页面更加美观。

**示例**

<img src="1635224166076.gif" alt="1635224166076" style="zoom:80%;" />

## 2.弹簧动画

在自然界中，弹簧的弹力是一种引导相互作用和运动的力。而在Android界面中，Android 支持库包含基于物理特性的动画 API，它们依靠物理定律来控制动画的发生方式,让动画应尽可能运用现实世界的物理定律，以使其看起来更自然。例如，它们应在目标发生变化时保持动量，并在任何变化期间进行平稳过渡。

下图演示了弹簧释放效果。圆圈中间的加号 (+) 表示通过触摸手势施加的力。

![spring-release](spring-release.gif)



## 3.代码详解

### 3.1弹簧动画的生命周期

在弹簧动画中，我们利用`SpringForce` 类来自定义弹簧的刚度、阻尼比以及最终位置。动画一开始，弹簧弹力便会更新每一帧的动画值和速度。动画会一直持续，直到弹簧弹力达到平衡状态。

### 3.2弹簧制作的步骤

#### 3.2.1 添加支持库

在`bulid.grade`中添加依赖

```kotlin
implementation "androidx.dynamicanimation:dynamicanimation:1.1.0-alpha03"
```

可查看最新版本，我的是 `1.1.0-aloha03`

https://developer.android.google.cn/jetpack/androidx/versions

#### 3.2.2 创建弹簧动画

弹簧力定义了弹簧的刚度、阻尼比以及静止位置。一旦启动了`SpringAnimation`，在每一帧上，弹簧力将更新动画的值和速度。动画将继续运行，直到弹簧力达到平衡。如果在动画中使用的弹簧是无阻尼的，动画将永远不会达到平衡。相反，它将永远振荡。

```kotlin
val imageView =findViewById<View>(R.id.imageView)
val springAnim = imageView.let { img ->
        SpringAnimation(img, DynamicAnimation.TRANSLATION_Y, 100f)
}
```

`SpringAnimation`类是由`SpringForce`驱动的动画。它定义了弹簧动画的初始值。

```kotlin
//源码
public <K> SpringAnimation(K object, FloatPropertyCompat<K> property,float finalPosition) {
        super(object, property);
        mSpring = new SpringForce(finalPosition);
    }
```

`object`: 想要实现弹簧动画的对象

`FloatPropertyCompat<K> property`:将对象的属性动画化

- `TRANSLATION_X`、`TRANSLATION_Y` 和 `TRANSLATION_Z`：这些属性用于控制视图所在的位置，值为视图的布局容器所设置的左侧坐标、顶部坐标和高度的增量。
  - `TRANSLATION_X` 表示左侧坐标。
  - `TRANSLATION_Y` 表示顶部坐标。
  - `TRANSLATION_Z` 表示视图相对于其高度的深度。

`float finalPosition`:对象最终停留的位置

这样，一个弹簧动画的初始值就被定义好了。

#### 3.2.3 设置起始速度

当然，如果只定义了初始化，那么你将会看到一个不动的动画，这是因为没有设置起始速度，弹簧的速度为0，那么动画自然是不动的。现在我们来设置一下初始固定值。

```kotlin
springAnim.setStartVelocity(3.0F)
```

**将“dp/秒”转换为“像素/秒”**

弹簧的速度必须以“像素/秒”为单位。如果需要提供固定值作为起始速度值，那么请以“dp/秒”为单位提供，然后将其转换为以“像素/秒”为单位。要进行转换，请使用 `TypedValue` 类中的 `applyDimension()` 方法。

```kotlin
val pixelPerSecond: Float = TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, dpPerSecond, resources.displayMetrics)
```

#### 3.2.4 设置弹簧属性

##### 阻尼比

**阻尼**就是使自由振动衰减的各种摩擦和其他阻碍作用，而**阻尼比**用于描述弹簧振动逐渐衰减的状况。调用 `setDampingRatio()` 方法并传递要增加到弹簧上的阻尼比。该方法会返回设置了阻尼比的弹簧弹力对象。

```kotlin
springAnim.spring.dampingRatio = 0.1F
```

- 当阻尼比大于 1 时，会出现过阻尼现象。它会使对象快速地返回到静止位置。
- 当阻尼比等于 1 时，会出现临界阻尼现象。这会使对象在最短时间内返回到静止位置。
- 当阻尼比小于 1 时，会出现欠阻尼现象。这会使对象多次经过并越过静止位置，然后逐渐到达静止位置。
- 当阻尼比等于零时，便会出现无阻尼现象。这会使对象永远振动下去。

系统中提供以下阻尼比常量：

- `DAMPING_RATIO_HIGH_BOUNCY`：弹性很强的弹簧的阻尼比。值为 0.2（图1）
- `DAMPING_RATIO_MEDIUM_BOUNCY`：低弹性弹簧的阻尼比。值为0.75
- `DAMPING_RATIO_LOW_BOUNCY`：中等弹性弹簧的阻尼比。值为0.5
- `DAMPING_RATIO_NO_BOUNCY`:无弹性弹簧的阻尼比。这个阻尼比将创建一个临界阻尼弹簧，在没有振荡的最短时间内返回平衡。值为1.0

默认阻尼比设置为 `DAMPING_RATIO_MEDIUM_BOUNCY`

<img src="high_bounce.gif" alt="high_bounce" style="zoom:50%;" /><img src="low_bounce.gif" alt="low_bounce" style="zoom:50%;" /><img src="medium_bounce.gif" alt="medium_bounce" style="zoom:50%;" /><img src="no_bounce.gif" alt="no_bounce" style="zoom:50%;" />

##### 刚度

**刚度**定义了用于衡量弹簧强度的弹簧常量。不在静止位置的坚硬弹簧可对所连接的对象施加更大的力。调用 `setStiffness()` 方法并传递要增加到弹簧上的刚度值。该方法会返回设置了刚度的弹簧弹力对象。

```kotlin
springAnim.spring.stiffness = STIFFNESS_HIGH
```

系统中提供了以下刚度常量：

- `STIFFNESS_HIGH`：用于极刚性弹簧的刚度常数。值为10000.0
- `STIFFNESS_MEDIUM`：中等刚度弹簧的刚度常数。这是弹簧力的默认刚度。值为1500.0
- `STIFFNESS_LOW`：低刚度弹簧的刚度常数。值为200.0
- `STIFFNESS_VERY_LOW`：刚度很低的弹簧的刚度常数。值为50.0

Hint: 刚度的值必须为正数！

#### 3.2.5 启动动画

```kotlin
springAnim.start()
```

#### 3.2.6 取消动画

我们可以取消动画或跳至动画结尾处。如果用户突然退出应用或视图变得不可见时，就可以使用取消动画。

有两种方法可以终止动画。`cancel()` 方法可在动画当前所在的值处终止它。`skipToEnd()` 方法可使动画跳至最终值，然后终止它。

在终止动画之前，请务必先查看弹簧的状态。如果状态为无阻尼，则动画永远不会到达静止位置。要查看弹簧的状态，请调用 `canSkipToEnd()` 方法。如果弹簧处于有阻尼状态，则该方法会返回 `true`，否则返回 `false`。

`cancel()` 方法只能对主线程调用。

#### 3.2.7 其他属性

##### 监听器

`DynamicAnimation` 类提供了两个监听器：`OnAnimationUpdateListener` 和 `OnAnimationEndListener`。这些监听器会监听动画中的更新，例如当动画值发生变化和动画结束时。

##### 设置动画起始值和范围

要设置动画的起始值，请调用 `setStartValue()` 方法并传递动画的起始值。如果未设置起始值，则动画将使用对象属性的当前值作为起始值。

如果要将属性值限制在特定范围内，则可以设置最小动画值和最大动画值。如果您在为具有内在范围的属性（如 Alpha 透明度的范围为 0 到 1）添加动画效果，这样做还有助于控制范围。

- 要设置最小值，请调用 `setMinValue()` 方法并传递属性的最小值。
- 要设置最大值，请调用 `setMaxValue()` 方法并传递属性的最大值。

这两种方法都将返回正在为其设置值的动画。

##### 创建自定义弹簧弹力

如果我们不想使用默认弹簧弹力，可以创建自定义弹簧弹力。创建弹簧弹力之后，便可以自定义阻尼比和刚度等属性。

1. 创建一个 `SpringForce`对象。

   ```kotlin
   val force = SpringForce()
   ```

2. 通过调用相应的方法来分配属性。您还可以创建一个方法链。

   ```kotlin
   force.setDampingRatio(DAMPING_RATIO_LOW_BOUNCY).setStiffness(STIFFNESS_LOW)
   ```

3. 调用`setSpring()`方法将弹簧设置为动画效果。
