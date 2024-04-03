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

## Premake
- [premake5 alpha16](https://github.com/premake/premake-core)
- 留存 License
- 使用Premake进行编译，以适应不同平台
- 使用lua编写Premake文件
- 直接点击`GenerateProject.bat`生成对应的sln和其他必要文件
- 进行<font color="red">{COPY} </font>时，由于第一次并没有生成对应目录 ，需要执行第二次才能够正常运行程序
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

project "MiniEngine"
	location "MiniEngine"   //所在文件夹(MiniEngine.sln目录下的MiniEngine文件夹)
	kind "SharedLib"        //设置生成类型，因为是dll 所以使用SharedLib
	language "C++"          //设置语言

	targetdir ("bin/" .. outputdir .. "/%{prj.name}")   //二进制文件生成路径 例如:bin/Debug-Winodws-x64/MiniEngine
	objdir ("bin-int/" .. outputdir .. "/%{prj.name}")   //中间文件生成路径

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


## Event Sysytem
- 设置事件发送Application
    - 鼠标事件：鼠标点击时的position以及button
- 为窗口类设置回调函数，如果窗口触发了某些事件，则会回调并告诉Application
- Application需要设置事件监听
- Event类需要设置不同的Event事件(设置事件调度程序，根据参数实现自动调用不同函数(*通过虚函数实现？？*))，例如Mouse、KeyBoard等