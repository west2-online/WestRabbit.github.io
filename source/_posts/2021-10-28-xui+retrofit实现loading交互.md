---
title: XUI + Retrofit 完成请求的页面加载
date: 2021-10-28
tags: 
    - Android
    - XUI
    - Retrofit
author: bngel
---


>  前言:
>
> 首先推荐一下本次使用的`Android`的框架: `XUI`. 继承了绝大多数开发`UI`时所需要使用的控件, 大大加快了开发`APP`的速度.
>
> `Github`地址: [XUI]([xuexiangjys/XUI: 💍A simple and elegant Android native UI framework, free your hands! (一个简洁而优雅的Android原生UI框架，解放你的双手！) (github.com)](https://github.com/xuexiangjys/XUI))

关于`retrofit`的使用在本篇文章中就不再赘述了.

直接进入正题:

1. 将retrofit的返回类进行泛型封装(前提是后端有对应的`CommonData`)

   ```kotlin
   data class DefaultData <T> (
       val `data`: T?,
       val message: String,
       val code: Int
   )
   ```

2. 此时只需要在对应的`retrofit`接口中传入特化的数据类即可.

   ```kotlin
   @FormUrlEncoded
   @POST("game")
   fun postGame(
       @Field("private") private: Boolean?,
       @Header("Authorization") token: String
   ): Call<DefaultData<PostGame>>
   ```

   例如本处传入的是`PostGame`数据类

3. 封装对应的`retrofit`异步执行事件

   ```kotlin
   interface DaoEvent {
       fun <T> success(data: DefaultData<T>)
   
       fun <T> failure(data: DefaultData<T>?)
   }
   ```

4. 在对应的`service`中进行传入处理

   同样以上文中`PostGame`为例

   ```kotlin
   fun postGame(private: Boolean, token: String, event: DaoEvent) {
       try {
           val postGame = gameDao.postGame(private, token)
           DaoRepository.enqueue(postGame, event)
       } catch (e:Exception) {
           e.printStackTrace()
       }
   }
   ```

5. 此时根据`kotlin`语言的性质. 就可以使用扩展方法直接扩展对应的`DaoEvent`.

6. 为了页面美观, 需要在发起请求时弹出加载页面的`Dialog`

   此处使用的就是`XUI`中自带的`LoadingDialog`

   为了方便起见, 我也同样进行了工具类的封装

   ```kotlin
   fun createSimpleLoadingTipDialog(context: Context, content: String): MaterialDialog 
   	= MaterialDialog.Builder(context)
           .iconRes(R.drawable.dialog_tip)
           .limitIconToDefaultSize()
           .title("提示:")
           .content(content)
           .progress(true, 0)
           .progressIndeterminateStyle(false)
           .build()
   ```

   在请求开始前进行`show`

   ```kotlin
   val materialDialog = UIRepository.createSimpleLoadingTipDialog(this, "加载中...")
   materialDialog.show()
   ```

7. 之后进行请求的发送, 同样以`postGame`为例

   ```kotlin
   sservice.postGame(true, StatusRepository.userToken, object:DaoEvent {
       override fun <T> success(data: DefaultData<T>) {
           // 成功事件
           materialDialog.dismiss()
       }
       override fun <T> failure(data: DefaultData<T>?) {
           // 失败事件
           materialDialog.dismiss()
       }
   })
   ```

   使用`object:DaoEvent`直接进行了一个事件的传.

   无论成功与否, 在请求结束后都将当前的载入框关闭.

   至此实现了一个比较完善的请求交互.

