## Entry Point
- 为了能够支持多平台(当然现在仅限WINDOWS)，在MiniEngine中构建EntryPoint头文件，并通过声明外部函数`extern MG::Application* MG::CreateApplication();`实现根据客户端不同，自动创建对应对象
```
//MiniEngine/Application.h
namespace MG {
    //class Application
    // ...

    //To be defined in CLIENT
    Application* CreateApplication();
}

//MiniEngine/EntryPoint.h
#ifdef MG_PLATFORM_WINDOWS

extern MG::Application* MG::CreateApplication();

int main(int argc, char** argv)
{
	auto app = MG::CreateApplication();
	app->Run();
	delete app;
}

#endif // MG_PLATFORM_WINDOWS

//CLIENT SandBox/SandBoxAPP.cpp
//在客户端实现
MG::Application* MG::CreateApplication()
{
	return new SandBox();
}
```

## Logging
[第三方库](https://github.com/gabime/spdlog)
- 设定想要的输出样式，并宏定义一些输入函数
- 使用`shared_ptr`防止内存溢出


## Premake
- [premake5 alpha16](https://github.com/premake/premake-core)
- 留存 License
- 使用Premake进行编译，以适应不同平台
- 使用lua编写Premake文件
- 直接点击`GenerateProject.bat`生成对应的sln和其他必要文件
- 进行<font color="red">{COPY} </font>时，由于第一次并没有生成对应目录 ，需要执行第二次才能够正常运行程序
- <font color="red">必须使用`#include"MGpch.h`而不能使用`#include"../MGpch.h`，否则会报找不到MGpch.h的错误</font>
```
//premake5.lua

workspace "MiniEngine"
	architecture "x64"

	configurations
	{
		"Debug",
		"Release",
		"Dist"  //  distribution  不会有任何日志系统等其他类似的东西
	}

outputdir = "%{cfg.buildcfg}-%{cfg.system}-%{cfg.architecture}" //输出路径(模式-系统-架构)，例如Debug-Windows-x64


-- Include directories relative to root folder (solution directory)
IncludeDir = {}                                         //创建一个lua table，用来存储需要包含的头文件目录路径
IncludeDir["GLFW"] = "MiniEngine/vendor/GLFW/include"   //加入到头文件目录路径中，相当于是编译器头文件路径，可以直接使用#include"glfw.h"等等，而无需写出相对路径

include "MiniEngine/vendor/GLFW"                        //负责将该目录下的premake5.lua复制到该文件中

project "MiniEngine"
	location "MiniEngine"   //所在文件夹(MiniEngine.sln目录下的MiniEngine文件夹)
	kind "SharedLib"        //设置生成类型，因为是dll 所以使用SharedLib
	language "C++"          //设置语言

	targetdir ("bin/" .. outputdir .. "/%{prj.name}")   //二进制文件生成路径 例如:bin/Debug-Winodws-x64/MiniEngine
	objdir ("bin-int/" .. outputdir .. "/%{prj.name}")   //中间文件生成路径

    pchheader "MGpch.h"                     //相当于对项目MiniEngine 设置使用预编译头
    pchsource "MiniEngine/src/MGpch.cpp"    //相当于VS中对该MGpch.cpp文件设置属性，创建预编译头

    files       //所需的文件类型
    {
        "%{prj.name}/src/**.h"      //**是为了让其能够在src文件夹下访问所有h文件，因为该文件夹下还有多层文件夹
        "%{prj.name}/src/**.cpp"
    }

    includedirs
    {
        "%{prj.name}/vendor/spdlog/include" //包含路径，MiniEngine项目中使用了该默认路径
    }

    filter "system:windows"     //在该filter以下的内容仅适用于windows，除非碰到另外的filter或者project
	    cppdialect "C++17"
        staticruntime "On"      //令其静态链接
        systemversion "latest" //指定windowsSDK，若不指定，premake会默认生成8.1，需要手动更改

        defines     //定义需要的宏
        {
            "MG_PLATFORM_WINDOWS",
            "MG_BUILD_DLL"
        }

        postbuildcommands           //将MiniEngine生成的dll文件复制到SandBox文件夹中，使其能够正常运行而无需手动复制
        {
            ("{COPY} %{cfg.buildtarget.relpath} ../bin/" .. outputdir .. "/SandBox")
        }


        //指定不同的模式所需的filter(定义不同的宏，符号，以及是否开启优化)
        filter "configurations:Debug"
            defines "MG_DEBUG"
            symbols "On"
            
        filter "configurations:Release"
            defines "MG_RELEASE"
            optimize "On"

        filter "configurations:Dist"
            defines "MG_DIST"
            optimize "On"

project "SandBox"
	location "SandBox"
	kind "ConsoleApp"   //生成.exe
	language "C++"

	targetdir ("bin/" .. outputdir .. "/%{prj.name}")
	objdir ("bin-int/" .. outputdir .. "/%{prj.name}")

	files
	{
		"%{prj.name}/src/**.h",
		"%{prj.name}/src/**.cpp"
	}

	includedirs
	{
		"MiniEngine/vendor/spdlog/include",
		"MiniEngine/src"
	}

	links           //设置项目引用 链接
	{
		"MiniEngine"
	}

	filter "system:windows"
		cppdialect "C++17"
		staticruntime "On"
		systemversion "latest"

		defines
		{
			"MG_PLATFORM_WINDOWS"
		}


	filter "configurations:Debug"
		defines "MG_DEBUG"
		symbols "On"
		
	filter "configurations:Release"
		defines "MG_RELEASE"
		optimize "On"

	filter "configurations:Dist"
		defines "MG_DIST"
		optimize "On"

```


## Event System
- 设置事件发送Application
    - 鼠标事件：鼠标点击时的position以及button
    - 按键事件：键盘点击 释放、以及重复
    - App事件：窗口大小改动、关闭
- 为窗口类设置回调函数，如果窗口触发了某些事件，则会回调并告诉Application
    - 在WindowsWindow.cpp中实现，通过glfw自带的callback函数设置对应数据并创建对应的event类，通过设置WindowData中的EventCallback(event)来实现回调
        ```
        //WindowsWindow::Init()
        
        //Set GLFW callback
        glfwSetWindowSizeCallback(m_Window, [](GLFWwindow* window, int width, int height)
            {
                WindowData& data = *(WindowData*)glfwGetWindowUserPointer(window);
                data.Width = width;
                data.Height = height;

                WindowResizeEvent event(width, height);
                data.EventCallback(event);
            });

        ```
- Application需要设置事件监听
- Event类需要设置不同的Event事件, 设置事件调度程序，根据参数实现自动调用不同函数，例如Mouse、KeyBoard等

## Window
- 创建Window接口，供其他平台开发(目前仅实现了Windows)

## Layer
- 需要保证overlay是最后才被渲染的
- 使用一个iterator记录Layer可以插入的位置，并进行更新
- 将overLayer插入到vector的结尾<font color = "red"> 但是这样在进行vector扩充时会产生大量开销</font>

## Modern OpenGL
[glad instead of glew](https://glad.dav1d.de/)
- glad(OpenGL Loader Generator)：加载OpenGL的函数地址进行调用，可以帮助我们自动生成这些加载函数的代码，从而简化了 OpenGL 函数的加载过程。
- GLAD 可以自动生成针对不同操作系统和编译器的代码，因此它在跨平台开发中非常有用。同时，GLAD 还支持加载 OpenGL 的核心功能（Core Profile）以及扩展功能（Extensions），使得开发者可以方便地使用 OpenGL 的各种特性

- GLAD（GL Loader Generator）和 GLEW（OpenGL Extension Wrangler）都是用于加载 OpenGL 函数的库，它们的主要功能是类似的，但是在一些方面有所不同：
    **1. 生成方式：**

    - GLAD：GLAD 通过一个在线服务生成加载器代码，你可以选择需要的 OpenGL 版本和扩展，然后生成相应的加载器代码。
    - GLEW：GLEW 则是一个预编译的库，你需要将它的头文件和库文件添加到你的项目中，然后在编译时链接 GLEW 库。

    **2. 支持的OpenGL版本：**

    - GLAD：GLAD 支持加载 OpenGL 的核心功能（Core Profile）和扩展功能（Extensions），可以根据需要选择加载的 OpenGL 版本和扩展。
    - GLEW：GLEW 主要用于加载 OpenGL 的扩展功能，不支持加载核心功能。它通常用于旧版本的 OpenGL 开发，不支持现代 OpenGL 的核心特性。

    **3. 跨平台性：**

    - GLAD：GLAD 支持跨平台，并且可以为不同的操作系统和编译器生成相应的加载器代码。
    - GLEW：GLEW 也支持跨平台，但需要手动编译和链接 GLEW 库。

    **4. 库的大小和依赖：**

    - GLAD：GLAD 的代码生成方式意味着你只会包含你需要的函数的加载代码，因此可以避免不必要的依赖和代码。
    - GLEW：GLEW 是一个较大的预编译库，可能会增加项目的大小，并且需要额外的库文件和链接。

    总的来说，GLAD 更加灵活和轻量化，适用于现代 OpenGL 开发，并且可以根据需要自动生成加载器代码。而 GLEW 则适用于旧版本的 OpenGL 开发，对于不需要核心功能的项目也是一个可选的方案。

<font color = "red">由于glfw以及glad均包含了OpenGL的头文件，因此会报重复包含的错误；此时通过预处理器定义中添加`GLFW_INCLUDE_NONE`可以在包含glfw3.h时排除所依赖的OpenGL头文件</font>

## ImGUI
- 通过dispatch机制，使得能够触发ImGui事件
- docking：停靠窗口，将界面放到任何地方
- Viewports：
**TODO**：存在频闪问题，且触发一种按键后，无法再次进行触发

## Input
- 设置Input Poll，设置Input接口，并根据具体平台实现不同的Input类(仅实现了Windows)
- 为了后续能够支持多种图形API，需要设置专属于自己引擎的KeyCodes；通过#ifdef等操作，根据具体平台使用不同的`KeyCodes.h`文件(目前仅实现了基于`glfw3.h`)


## Rendering
#### Renderer context
- 将初始化context的OpenGL APi 抽象为一个函数，并设置接口，供以后支持不同图形库

#### VB IB Layout
- 建立一个VB和IB接口 并在具体的图形库API去实现VB以及IB
- VA与shader密切相关，需要创建VextexBufferLayerout设置VA的布局
- 创建一个BufferLayout类来存储VA的实际布局
- 使用`std::initializer_list`设置Layout的构造函数，使得能够直接使用{{message}}的方式构建Layout，同时提高代码可读性，了解VB的实际布局
#### VA
- VA与shader密切相关，需要创建VextexBufferLayerout设置VA的布局

#### RendererAPI
- `RendererAPI`作为接口设置函数`SetClearColor`, `Clear`, `DrawIndexed`以及自己的`s_API`
- `RenderCommand`则调用`RendererAPI`，并创建对应图形库的RendererAPI(目前为OpenGL)，用指针记录，就算发生内存泄漏也仅一个字节
- `Renderer`则调用`RenderCommand`，负责开始和结束场景，并且调用`RenderCommand`进行图像绘制
- `OpenGLRendererAPI`则作为`RendererAPI`的派生类，实现专注于OpenGL库的方法调用
<font color = "red"> 为什么要多此一举创建RenderCommand并调用RendererAPI已经设置的方法，难道说只是为了将RendererAPI抽象为一个接口嘛 </font>


#### ISSUE
- 切换图形库API时，会导致需要编译整个渲染模块甚至是整个引擎，这并不是想要实现的
    - 可以将采用不同图形库API的渲染模块编译成DLL，这样在运行过程中可以选择合适的进行调用，但是这样会极大降低性能 
    - 创建一个Shader的接口(纯虚函数)，并使用不同图形库API 编写其派生类，在其中实现支持多图形库API；但是对于客户端而言，只能够看见Shader类

## Camera
- 计算实际的画面使用 projection * view * model * vertexPos(针对OpenGL 和glm 因为是按列存储；例如DirectX 则是按行存储，需要使用vertexPos * model * view * projection)
- Camera实际上就是projection * view ，而model则是在object身上，vertexPos则是在mesh上
- 在`BeginScene`中初始化Camera，同时将Camera计算获得的PV(projection and view)作为uniform输入到Shader中
- 正交相机(orthographic camera)：适用于2D环境，并不存在近大远小的透视
- 为了使得相机移动尽可能smooth，不能够使用键盘事件回调函数

## DeltaTime
- 为了防止某些功能模块更新频率过快，导致影响体验(例如，相机移动，帧率越高 移动越快，同时也会影响性能<font color ="blue">(我猜的)</font>)，使用DeltaTime来定时更新功能模块，在所有机器上都设置固定更新频率，提高体验感
- 使用移动速度乘上DeltaTime，即可实现在不同帧率的机器上，在同一时间内到达相同位置，DeltaTime越短，说明帧率越高，渲染的画面次数越多


## Material
- 本质就是一个shader 加上一堆uniforms，通过设置不同的uniforms渲染出不同的效果
- 通过将shader抽象，并调试其中的uniforms，同时调用`MG::Renderer::Submit()`，传入matrial而不是shader，可以进一步简化步骤；并且通过将使用相同shader的object整合到一起，可以有效地降低性能消耗，而不用来回绑定shader


## Texture
- 读取`texture.glsl`文件，获得对应的vertexShader和fragmentShader
- **小Tips:**通过在`texture.glsl`文件中，加入`uniform u_texture`以及`uniform u_Color`，并在输出Color时进行两者相乘(`color = texture(u_Texture, v_TexCoord) * u_Color;`)实现Bind单个Shader，但是能够绘制两种不同的图形(使用Texture的 以及纯色)；**具体做法：**
    1. 如果想要使用Texture，，则将u_Color设置为全白即可(即glm::vec4(1.0f))，这样只会出现Texture，而不会有颜色的影响
        ```
        //set color to white
        s_Data->TextureShader->SetFloat4("u_Color", glm::vec4(1.0f));
        ```
    2. 如果只想使用纯色，则在`BeginScene`期间，就创建一个`WhiteTexture`，通过手动创建像素为1 * 1的白色像素点，实现白色的Texture，这样只会出现设置的纯色，而不会使用Texture
        ```
        s_Data->TextureShader->SetFloat4("u_Color", color);
        //bind white texture
        s_Data->WhiteTexture->Bind();
        ```
    3. 如果两者都想使用，则同时输入即可

## Batch Rendering
- 本质就是将所有的顶点信息以及索引信息存储到一个确定的数组中，该数组有提前预设的大小；在绘制过程中，该数组只会调用一次DrawCall来减小DrawCall的性能损失
- 当顶点数量大于预设的数组大小时，则需要另外创建一个数组并额外调用一次DrawCall，以此类推
- Texture slot：一般来说有32个slot；其中0号slot插槽基本是用来存储white_texture的，剩下的31个slot则用来绑定不同的texture；当一个数组调用DrawCall时，则会调用里面需要用到的textures；而当该数组中调用的textures数量大于31个，则说明在此次DrawCall中无法完全全部绘制，因为需要重新绑定新的texture，那么则需要将使用新的texture的顶点重新构建一个数组并执行一次DrawCall