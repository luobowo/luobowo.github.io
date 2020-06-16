---
layout: post
title:  "DirectX 9的安装与配置"
date:   2016-01-05 13:15:00
categories: 游戏开发
tags: DirectX
---


最近刚开始学Direct3D，首先要安装DirectX并配置环境，主要包括三个步骤：  
一、安装DirectX SDK.  
二、配置环境.  
三、链接.lib文件，运行示例   
四、Error Code s1023的解决方案.  



## 主要步骤

### 一. 安装DirectX SDK  
首先登陆Microsoft Download Center下载最新版本的DirectX SDK:   https://www.microsoft.com/en-us/download/details.aspx?id=6812  

### 二. 配置环境

安装完DirectX SDK后，我们需要配置VS的开发环境，以便能够找到SDK，主要包括包含目录和库目录。  
（1）打开属性管理器（“视图——其他窗口——属性管理器”），如图所示：   
![属性管理器](https://img-blog.csdnimg.cn/20200616093558315.png). 

(2) 双击Debug \| Win32下的“Microsoft.Cpp.Win32.user“，在弹出的配置框中配置。这个设置是对所有工程有效的。你可以打开其他的工程或者新建新的工程，可以看到都继承了此配置。  

(3) 点击VC++目录，如图所示：  
![c++ dir](https://img-blog.csdnimg.cn/20200616093616216.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZpdHp6aGFuZw==,size_16,color_FFFFFF,t_70)   

选择Include files, 这个是DirectX头文件所在的地方，点击下面的新建文件夹按钮将DirectX头文件所在的路径输入进去。在我这里是  
`C:\Program Files\Microsoft DirectX SDK (June 2010)\Include`.  
 选择Library fies，这是.lib文件所在的地方，如法炮制，将`DirectX .lib`文件的路径输入进去，在我这里是
`C:\Program Files\Microsoft DirectX SDK (June 2010)\Lib\x86`

这样，便配置好了环境。当然，如果不想对所有的工程进行配置的话，也可以只针对当前工程进行配置，配置方法为在解决方案中选中要配置的工程，右键属性-配置属性-VC++目录，分别按上面的方法配置包含目录和库目录。

### 三. 链接.lib文件，运行示例

新建一个空的Win32项目，然后对项目，进行右键属性-链接器-输入-附加依赖项，输入以下常用的附加lib文件： 
```
dxerr.lib 

dxguid.lib

d3d9.lib

d3dx9.lib

 d3dx9d.lib 

dinput8.lib

winmm.lib

comctl32.lib
```

如果不想这样添加附加依赖项的话，可以在代码中#include声明后加入:`#pragma comment (lib,"d3d9.lib")`等引入库文件。

然后，添加一个源文件，输入以下示例代码：  
```cpp
#include <d3d9.h>
#include <d3dx9.h>
 
//-----------------------------------------------------------------------------
// 全局变量
//-----------------------------------------------------------------------------
LPDIRECT3D9             g_pD3D       = NULL; // 用于创建D3D设备
LPDIRECT3DDEVICE9       g_pd3dDevice = NULL; // 用于创建渲染设备
 
 
//-----------------------------------------------------------------------------
// Name: InitD3D()
// Desc: 初始化D3D
//-----------------------------------------------------------------------------
HRESULT InitD3D( HWND hWnd )
{
    // 创建D3D对象
    if( NULL == ( g_pD3D = Direct3DCreate9( D3D_SDK_VERSION ) ) )
        return E_FAIL;
 
    // 创建结构用于创建D3D设备
    D3DPRESENT_PARAMETERS d3dpp;
    ZeroMemory( &d3dpp, sizeof( d3dpp ) );
 
    d3dpp.Windowed         = TRUE;
    d3dpp.SwapEffect       = D3DSWAPEFFECT_DISCARD;
    d3dpp.BackBufferFormat = D3DFMT_UNKNOWN;
 
    // 创建D3D设备
    if( FAILED( g_pD3D->CreateDevice( D3DADAPTER_DEFAULT, 
                                      D3DDEVTYPE_HAL, 
                                      hWnd,
                                      D3DCREATE_SOFTWARE_VERTEXPROCESSING,
                                      &d3dpp, 
                                      &g_pd3dDevice ) ) )
    {
        return E_FAIL;
    }
 
    // 设置设备状态
 
    // 设置渲染格式
    g_pd3dDevice->SetRenderState( D3DRS_CULLMODE, D3DCULL_NONE );
 
    // 关闭光照
    g_pd3dDevice->SetRenderState( D3DRS_LIGHTING, FALSE );
 
    return S_OK;
}
 
 
//-----------------------------------------------------------------------------
// Name: InitVB()
// Desc: 初始化顶点缓冲
//-----------------------------------------------------------------------------
HRESULT InitVB()
{
 
    return S_OK;
}
 
 
//-----------------------------------------------------------------------------
// Name: Cleanup()
// Desc: 释放对象资源
//-----------------------------------------------------------------------------
VOID Cleanup()
{
 
    if( g_pd3dDevice != NULL ) 
        g_pd3dDevice->Release();
 
    if( g_pD3D != NULL )       
        g_pD3D->Release();
 
}
 
 
//-----------------------------------------------------------------------------
// Name: Render()
// Desc: 场景渲染
//-----------------------------------------------------------------------------
VOID Render()
{
    // 清除场景背景
    g_pd3dDevice->Clear( 0, NULL, D3DCLEAR_TARGET, D3DCOLOR_XRGB(0,0,255), 1.0f, 0 );
 
    // 场景开始渲染
    if( SUCCEEDED( g_pd3dDevice->BeginScene() ) )
    {
 
        // 场景结束渲染
        g_pd3dDevice->EndScene();
    }
 
    // 提交缓冲
    g_pd3dDevice->Present( NULL, NULL, NULL, NULL );
 
}
 
 
//-----------------------------------------------------------------------------
// Name: MsgProc()
// Desc: 窗口消息句柄
//-----------------------------------------------------------------------------
LRESULT WINAPI MsgProc( HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam )
{
 
    switch( msg )
    {
        case WM_DESTROY:
            Cleanup();
            PostQuitMessage( 0 );
            return 0;
    }
 
    return DefWindowProc( hWnd, msg, wParam, lParam );
}
 
 
//-----------------------------------------------------------------------------
// Name: WinMain()
// Desc: 应用程序入口
//-----------------------------------------------------------------------------
INT WINAPI WinMain( HINSTANCE hInst, HINSTANCE, LPSTR, INT )
{
    // 注册窗口类
    WNDCLASSEX wc = { sizeof(WNDCLASSEX), 
                      CS_CLASSDC, 
                      MsgProc, 
                      0L, 
                      0L,
                      GetModuleHandle(NULL), 
                      NULL, 
                      NULL, 
                      NULL, 
                      NULL,
                      L"D3D Project", NULL };
 
    RegisterClassEx( &wc );
 
    // 创建应用程序窗口
    HWND hWnd = CreateWindow( L"D3D Project", 
                              L"D3D Project",
                              WS_OVERLAPPEDWINDOW, 
                              100, 
                              100, 
                              480, 
                              320,
                              GetDesktopWindow(), 
                              NULL, 
                              wc.hInstance, 
                              NULL);
 
    // 初始化D3D
    if( SUCCEEDED( InitD3D( hWnd ) ) )
    {
        // 创建顶点缓冲
        if( SUCCEEDED( InitVB() ) )
        {
            // 显示窗体
            ShowWindow( hWnd, SW_SHOWDEFAULT );
            UpdateWindow( hWnd );
 
            // 进入消息循环
            MSG msg;
            ZeroMemory( &msg, sizeof( msg ) );
 
            while( msg.message != WM_QUIT )
            {
                if( PeekMessage( &msg, NULL, 0U, 0U, PM_REMOVE ) )
                {
                    TranslateMessage( &msg );
                    DispatchMessage( &msg );
                }
                else
                {
                    //开始渲染
                    Render();
                }
                    
            }
        }
    }
 
    UnregisterClass( L"D3D Project", wc.hInstance );
 
    return 0;
}
```

运行，成功则表示环境配置成功，然后就可以开始学习Direct3D了。在安装好的sdk的`Documentation/DirectX9`文件夹下面有`windows_graphics.chm和directx_sdk.chm`可以学习，其中directx_sdk.chm里的samples and tutorials是很好的学习资源，建议从tutorials开始学习。还有两个网站，一个是 http://www.directxtutorial.com/ ，里面也是tutorial，讲的很详细；另一个是 http://www.codesampler.com/dx9src.htm ，里面的例子很适合快速入门。



### 四. Error Code s1023的解决方案

如果安装过程中碰到如题所示的错误，是因为 C++ 2010 Redistributable的版本引起的，可以参考 https://stackoverflow.com/questions/4102259/directx-sdk-june-2010-installation-problems-error-code-s1023 的解决方案

