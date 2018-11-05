---
title: IDEA&Android Studio Plugin开发
date: 2018-11-05 20:47:53
author: CoderQiang
tags:
	- Java
	- 15级
---

如果敲代码时总是会有一些<span data-type="color" style="color:#F5222D">规律、重复</span>和<span data-type="color" style="color:#F5222D">浪费时间</span>的工作，那么你就可以好好考虑一下能否通过 <span data-type="color" style="color:#F5222D">脚本或插件</span> 的方式提高工作效率。idea 本身自带众多实用代码插件，getter,setter,代码补全... 且其第三方插件采用<span data-type="color" style="color:rgb(79, 79, 79)"><span data-type="background" style="background-color:rgb(255, 255, 255)">集中发布模式</span></span>，可以让开发者快速安装和发布插件，这点要优于 eclipse。
开发插件之前，得看看idea平台给我提供了哪些接口给我们使用，使用接口前最好简单了解下背后的实现原理。

这篇文章包含：
1. VFS(Virtual File System)  虚拟文件系统
2. Program Structure Interface  程序结构接口
3. 创建 idea plugin
4. plugin 代码编写及安装
### VFS
IDEA的文件系统 Virtual File System 目标:

> The virtual file system (VFS) is a component of *IntelliJ Platform* that encapsulates most of its activity for working with files. It serves the following main purposes:
> * Providing a universal API for working with files regardless of their actual location (on disk, in archive, on a HTTP server etc.)【提供一个通用的API处理文件，不论其位置(是在硬盘上，压缩文件里，网络服务器上等) 】
> * Tracking file modifications and providing both old and new versions of the file content when a modification is detected.【跟踪文件的修改，并且在检测到修改时，提供文件内容的旧版本和新版本】
> * Providing a possibility to associate additional persistent data with a file in the VFS.【<span data-type="color" style="color:rgb(51, 51, 51)"><span data-type="background" style="background-color:rgb(255, 255, 255)">提供将持久数据与VFS中的文件关联的可能性</span></span>】

第一个目的和Linux的VFS差不多，都是为了提供 统一的上层接口，以此 屏蔽 底层各种不同的文件系统，对开发者透明，以及方便其他文件系统快速接入。
具体实现就是通过 __快照 __的方式，将快照作为中间层，提供文件的“多”读“一”写，并且适配不同的文件系统，给插件开发者 提供统一接口。新增类文件，写入代码等对文件的操作 都是通过<span data-type="color" style="color:rgb(79, 79, 79)"><span data-type="background" style="background-color:rgb(255, 255, 255)">com.intellij.openapi.vfs.VirtualFile，基于二进制流进行读写，可以把它当做原生File类来类比。</span></span>
VituralFile是作用域是应用级别(即使在<span data-type="color" style="color:rgb(79, 79, 79)"><span data-type="background" style="background-color:rgb(255, 255, 255)">多个工程中打开，各工程展示的实例也由同一个虚拟文件实例代理</span></span>)， 对于一份快照，来自任何线程的数据读取都是允许的，但是只有主线程能够写入，官方称为 [通用线程规则](http://www.jetbrains.org/intellij/sdk/docs/basics/architectural_overview/general_threading_rules.html)，使用一个单独的 读写锁 实现。需要注意的是当读取文件时候，应调用VirtualFile的isValid方法是否返回true，因为本地文件已经被移除了，但是对应的VirtualFile没有被回收。
几种获取VirtualFile的方式：
> 1. <span data-type="color" style="color:rgb(64, 72, 83)"><span data-type="background" style="background-color:rgb(255, 255, 255)">从一个 action </span></span>: 通过调用 <span data-type="color" style="color:rgb(64, 72, 83)"><span data-type="background" style="background-color:rgb(239, 239, 239)">e.getData(PlatformDataKeys.VIRTUAL_FILE) 获取当前正在编辑的文件</span></span>
> 1. <span data-type="color" style="color:rgb(64, 72, 83)"><span data-type="background" style="background-color:rgb(255, 255, 255)">从本地系统的路径 </span></span>: <span data-type="color" style="color:rgb(64, 72, 83)"><span data-type="background" style="background-color:rgb(239, 239, 239)">LocalFileSystem.getInstance().findFileByIoFile()</span></span>
> 2. <span data-type="color" style="color:rgb(64, 72, 83)"><span data-type="background" style="background-color:rgb(255, 255, 255)">从一个PSI文件：</span></span><span data-type="color" style="color:rgb(64, 72, 83)"><span data-type="background" style="background-color:rgb(239, 239, 239)">psiFile.getVirtualFile()</span></span>
> 3. <span data-type="color" style="color:rgb(64, 72, 83)"><span data-type="background" style="background-color:rgb(255, 255, 255)">从文档</span></span> : <span data-type="color" style="color:rgb(64, 72, 83)"><span data-type="background" style="background-color:rgb(239, 239, 239)">FileDocumentManager.getInstance().getFile()</span></span>

### PSI
程序结构接口 <span data-type="color" style="color:rgb(64, 72, 83)"><span data-type="background" style="background-color:rgb(255, 255, 255)">Program Structure Interface，</span></span><span data-type="color" style="color:rgb(51, 51, 51)"><span data-type="background" style="background-color:rgb(255, 255, 255)">负责解析文件，创建语法和语义代码模型。</span></span>
__PsiFile__ 是将<span data-type="color" style="color:rgb(79, 79, 79)"><span data-type="background" style="background-color:rgb(255, 255, 255)">文件内容按照特定编程语言的元素层次结构相对应进行代理的根结构。PsiFile是所有Psi文件的基类，一种语言会有特定的实现，例如 Java语言对应PsiJavaFile类，XML对应 </span></span>XmlFile<span data-type="color" style="color:rgb(47, 47, 47)"><span data-type="background" style="background-color:rgb(255, 255, 255)"> 类</span></span><span data-type="color" style="color:rgb(79, 79, 79)"><span data-type="background" style="background-color:rgb(255, 255, 255)">。</span></span>
PsiFile作用域是 应用级别 ，如果一个文件所属的不同项目同时打开，这个文件会被不同的PsiFile实例代理。
获取__PsiFile__的方式：
> 1. 从action中：e.getData(LangDataKeys.PSI\_FILE)
>     
> 2. 从虚拟文件中：PsiManager.getInstance(project).findFile()
>     
> 3. 从文档中：PsiDocumentManager.getInstance(project).getPsiFile()
> 4. 从文件中的一个元素：psiElement.getContainingFile()
>     
> 5. 用FilenameIndex.getFilesByName(project, name, scope)方法在项目中找到一个指定名称的文件

<span data-type="color" style="color:rgb(51, 51, 51)"><span data-type="background" style="background-color:rgb(255, 255, 255)">PSI（程序结构接口）文件表示PSI元素的层次结构（所谓的PSI树），</span></span>__PsiElement__ 是所有PSI元素的基类，Psi元素可以是PsiClass Psi类，PsiMethod Psi方法，PsiImportList Psi导入列表，PsiParameter Psi参数，总之只要是Java类文件里有的结构，都用对应的Psi元素。元素之间可以相互嵌套，成树状结构。
获取__PsiElements__
> 1. 从 action中: e.getData(LangDataKeys.PSI\_ELEMENT) 如果光标位于一个引用上，返回的是解析引用之后的元素。
> 2. 从文件偏移: PsiFile.findElementAt() 返回的是最细级别的元素，如偏移量位于方法头中，返回的是PsiMethod,而不是PsiClass，虽然方法确实属于这个类.
> 3. 通过遍历一个 PSI file : 使用 PsiRecursiveElementWalkingVisitor
> 4. 通过解析一个引用 : PsiReference.resolve()


---

### 创建一个简单的Plugin
介绍完相关背景和知识后，我们实战开发2个plugin，现在开始先来实现第一个最简单的弹窗提示的Plugin，并将其 构建 打包 安装 到自己的idea中。
##### 1. 准备环境。安装 idea Ultimate 2018版 ( 其他版本也可以 )，JDK8。
##### 2.新建 Plugin工程 ， 顶部菜单栏 File > New > Project... ，然后出现如下界面，选择 IntelliJ Platform Plugin，SDK检查一下有没有，没有的话，下载到电脑，手动选择目录。然后 点击 Next> 为工程命名 > Next

![image.png | left | 457x448](https://cdn.nlark.com/yuque/0/2018/png/152699/1541414927417-a437dc20-104e-4bdf-8eb4-548243fa29cd.png "")

##### 3.创建好后，工程结构 如下图，在src目录下新建一个自己的包，我这用的是 com.chenyi


![image.png | left | 301x222](https://cdn.nlark.com/yuque/0/2018/png/152699/1541415135649-3107803c-1435-4ad3-9b5e-d1549254b311.png "")

4. __在刚刚的包下 新建一个 HelloAction 类，继承自 AnAction (全类名com.intellij.openapi.actionSystem.AnAction) ，并重写 actionPerformed 方法，当用户点击插件提供的 菜单项 时会触发该方法。与此同时还可以重写 update 方法，该方法是要显示 插件菜单项 时会触发，主要用于控制菜单项的显示与隐藏，当你打开的是xml文件时，这个时候适用于Java的插件就应该变灰不可选或者隐藏。__
5. __编写一个弹消息的Java代码__
```java
package com.chenyi;

import com.intellij.openapi.actionSystem.AnAction;
import com.intellij.openapi.actionSystem.AnActionEvent;
import com.intellij.openapi.ui.Messages;

/**
 * @author: chenyi.zsq
 * @Date: 2018/11/5
 */
public class HelloAction extends AnAction{

    @Override
    public void actionPerformed(AnActionEvent anActionEvent) {
        Messages.showMessageDialog("Hello Plugin", "PluginTitle", null);
        // 以下代码用于输出当前java文件里的所有方法信息，在调试模式可以看到。
        VirtualFile virtualFile = e.getData(PlatformDataKeys.VIRTUAL_FILE);
        PsiFile psiFile = PsiManager.getInstance(e.getProject()).findFile(virtualFile);
        Arrays.stream(psiFile.getChildren())
            .filter(child -> child instanceof PsiClass)
            .map(child -> child.getChildren())
            .flatMap(psiChild -> Arrays.stream(psiChild))
            .filter(classChild -> classChild instanceof PsiMethod)
            .forEach(psiMethod -> {
                
                PsiMethod method = (PsiMethod)psiMethod;
                StringBuilder sb = new StringBuilder();
                sb.append("方法的名: ").append(method.getName())
                    .append(" 返回值类型：").append(method.getReturnType().getPresentableText())
                    .append(" 参数:").append(method.getParameters());
                System.out.println(sb.toString());
            });
    }

}
```
__编写 resources/META-INF/plugin.xml 文件。在<actions></actions> 标签中加入刚刚编写的Action__
```xml
<idea-plugin>
  <id>com.your.company.unique.plugin.id</id>
  <name>Plugin display name here</name>
  <version>1.0</version>
  <vendor email="support@yourcompany.com" url="http://www.yourcompany.com">YourCompany</vendor>

  <description><![CDATA[
      Enter short description for your plugin here.<br>
      <em>most HTML tags may be used</em>
    ]]></description>

  <change-notes><![CDATA[
      Add change notes here.<br>
      <em>most HTML tags may be used</em>
    ]]>
  </change-notes>

  <!-- please see http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/build_number_ranges.html for description -->
  <idea-version since-build="173.0"/>

  <!-- please see http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/plugin_compatibility.html
       on how to target different products -->
  <!-- uncomment to enable plugin in all products
  <depends>com.intellij.modules.lang</depends>
  -->

  <extensions defaultExtensionNs="com.intellij">
    <!-- Add your extensions here -->
  </extensions>

  <actions>
    <action class="com.chenyi.HelloAction" id="HelloAction" text="第一个插件">
      <add-to-group group-id="WindowMenu" anchor="first"/><!--添加到菜单栏的Window标签下,位置固定在第一位 -->
    </action>
  </actions>

</idea-plugin>
```
__ __
__6. 编写完之后，可以点击运行或者调试，Idea将会另起一个进程自动安装插件，可以在此调试，但是debug模式中发现，对actionPerformed 方法没响应，对update事件有响应，最后调试时将代码写到update里运行，打正式包时把代码放回actionPerformed。如果想要正式包，需要点击构建打包，会在当前工程根目录下 生成一个 .jar文件。__

![image.png | left | 827x294](https://cdn.nlark.com/yuque/0/2018/png/152699/1541416892798-de1ae4ed-1a5b-403a-adfe-24a310ac61e0.png "")


7. __从本地磁盘安装Plugin ， 打开首选项(或Setting)-> Plugins -> install Plugin from disk，重启__


![image.png | left | 304x239](https://cdn.nlark.com/yuque/0/2018/png/152699/1541419292047-d970d2fe-2a64-4f6c-8422-90f97de9f71a.png "")


![image.png | left | 561x393](https://cdn.nlark.com/yuque/0/2018/png/152699/1541419297170-44565d81-f5a5-4e12-810b-829da829de74.png "")

8. __ 重启后可以在菜单栏下 Window标签看到刚刚开发的Plugin，点击之后会弹出消息,并显示当前编辑的文件名。__


![image.png | left | 637x325](https://cdn.nlark.com/yuque/0/2018/png/152699/1541419377186-5f75bf14-d8ee-4360-9c8b-da789ef2f4b3.png "")



![image.png | left | 700x226](https://cdn.nlark.com/yuque/0/2018/png/152699/1541419640784-d3c4894e-17ec-43bb-a65d-a61214483066.png "")

#### 至此，一个简单的Plugin开发完啦，下一篇我们将基于现实需求开发一个自动生成格式化代码插件。
需求：进入公司后发现，在HSF(内部一个 分布式服务框架 ) 服务间会频繁的进行跨应用服务调用，由于规范性，每个服务在调用前 都要封装一层SPI代码。用于AOP拦截入参和结果，以及对异常的捕捉。而代码往往就是 Service类的一层封装，代码类似如下：


![image.png | left | 827x380](https://cdn.nlark.com/yuque/0/2018/png/152699/1541420092970-acd15d85-109f-4e5f-8fe3-7b5eb33ad908.png "")

xxxSPI 和 xxxService方法名，参数都一样，返回值只要在xxxService返回值基础上getModule拆一下，开发人员同时需要创建 xxxServiceSPI和xxxServiceImplSPI，然后挨个把方法调用一遍，设置拦截器，我觉得完全可以将这部分时间省掉。
目前已完成大部分编码，GitHub地址：[https://github.com/zhengshiqiang47/ChenyiSPIPlugin](https://github.com/zhengshiqiang47/ChenyiSPIPlugin)

