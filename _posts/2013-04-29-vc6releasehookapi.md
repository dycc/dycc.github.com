---
title: '关于VC6release版本无法HOOK目标API的问题的探究'
layout: post
guid: urn:uuid:2031-04-17-peinfo-edit-and-qustion
tags:
    - Windows分析
---

<h3 style="text-align:right">————我的学习笔记</h3>

《windows程序设计》中有一节是关于HOOK API的实现，它拦截主模块中对于目标API的调用并替换成自己的函数。关键代码如下：

	//主函数
	::MessageBox(NULL, “原函数“, “HookDemo”, 0);//覆盖前调用一次MessageBoxA
	SetHook(::GetModuleHandle(NULL));//覆盖主模块对MessageBoxA的调用
	::MessageBox(NULL, “原函数“, “09HookDemo”, 0);//再次调用MessageBoxA
	//SetHook函数去保护并覆盖掉MessageBoxA的地址
	DWORD dwOldProtect;
	MEMORY_BASIC_INFORMATION mbi;
	VirtualQuery(lpAddr,&mbi,sizeof(mbi));
	VirtualProtect(lpAddr,sizeof(DWORD),PAGE_EXECUTE_READWRITE,&dwOldProtect);
	DWORD* lpNewProc = (DWORD*)MyMessageBoxA;
	::WriteProcessMemory(::GetCurrentProcess(),lpAddr, &lpNewProc, sizeof(DWORD), NULL);
	VirtualProtect(lpAddr,sizeof(DWORD),dwOldProtect,0);

此程序再VC6中编译成debug版本正常运行并能成功Hook掉MessageBoxA，但是编译成release版本后再调用SetHook函数后就无法实现对MessageBoxA的hook，vs2008ralease版貌似也有这种问题，现分析如下：

Vc6debug版本的程序中的.idata节是可写的，我们可以不用VirtualProtect而直接用WriteProcessMemory覆盖掉目标API的地址，但是在release版本下此节被编译器优化成了只读属性，所以在release下必须要用VirtualProtect加上写属性才能写内存成功，这是《windows程序设计》这本书给出的答案。

但是，这样在release下任然不能hook掉MessageBoxA函数，程序调用的还是SetHook函数调用前的MessageBoxA函数。。。。。

怎么办呢？下面我将对两个版本的文件分别进行反编译，如下：

Debug版本汇编代码：

第一次调用MessageBoxA函数：

	……
	18:       // 调用原API函数
	19:       ::MessageBox(NULL, “原函数“, “HookDemo”, 0);
	004010C8   mov         esi,esp
	004010CA   push        0
	004010CC   push        offset string “HookDemo” (00422024)
	004010D1   push        offset string “\xd4\xad\xba\xaf\xca\xfd” (0042201c)
	004010D6   push        0
	004010D8   call        dword ptr [__imp__MessageBoxA@16 (0042a2d4)]//这是第一次调用MessageBoxA函数
	004010DE   cmp         esi,esp
	004010E0   call        __chkesp (004013e0)
	……

第二次调用MessageBoxA函数：

	35:       // 挂钩后再调用
	37:       ::MessageBox(NULL, “原函数“, “HookDemo”, 0);
	004010FF   mov         esi,esp
	00401101   push        0
	00401103   push        offset string “HookDemo” (00422024)
	00401108   push        offset string “\xd4\xad\xba\xaf\xca\xfd” (0042201c)
	0040110D   push        0
	0040110F   call        dword ptr [__imp__MessageBoxA@16 (0042a2d4)] //这是第一次调用MessageBoxA函数
	00401115   cmp         esi,esp
	00401117   call        __chkesp (004013e0)
	……

从上面对API的两次调用可以看出来，两次都是取得IAT中的函数地址0042a2d4然后再以此为地址取出真正MessageBoxA地址，这样的话IAT中的项再调用SetHook函数时已经被覆盖掉了，所以当然能够实现拦截。

下面再来看release版本程序的反编译代码：

	//这是OD反汇编后的代码
	00401020  /$  56            PUSH ESI
	00401021  |.  8B35 C0604000 MOV ESI,DWORD PTR DS:[<&USER32.MessageBo>];  USER32.MessageBoxA
	//下面四个PUSH为MessageBoxA函数参数，由于是遵循_cedel调用约定，参数从右往左
	00401027  |.  6A 00         PUSH 0          ; /Style = MB_OK|MB_APPLMODAL
	00401029  |.  68 38704000   PUSH 09HookDe.00407038        ; |Title = “HookDemo”
	0040102E  |.  68 30704000   PUSH 09HookDe.00407030        ; |Text = “原函数“
	00401033  |.  6A 00         PUSH 0                        ; |hOwner = NULL
	//这是第一次调用MessageBoxA函数
	00401035  |.  FFD6          CALL ESI                       ; \MessageBoxA
	00401037  |.  6A 00         PUSH 0                        ; /pModule = NULL
	//取GetModuleHandleA地址
	00401039  |.  FF15 00604000 CALL DWORD PTR DS:[<&KERNEL32.GetModuleH>]; \GetModuleHandleA
	0040103F  |.  50            PUSH EAX
	//调用SetHook函数
	00401040  |.  E8 3B000000   CALL 09HookDe.00401080
	00401045  |.  83C4 04       ADD ESP,4
	00401048  |.  6A 00         PUSH 0
	0040104A  |.  68 38704000   PUSH 09HookDe.00407038       ;  ASCII “HookDemo”
	0040104F  |.  68 30704000   PUSH 09HookDe.00407030
	00401054  |.  6A 00         PUSH 0
	//第二次调用MessageBoxA函数
	00401056  |.  FFD6          CALL ESI
	00401058  |.  5E            POP ESI
	00401059  \.  C3            RETN
	……

可以看出来在第一次调用MessageBoxA时将导入项（IAT里的地址）存入了ESI寄存器中，而再SetHook函数中是覆盖的导入表中目标API中的地址，并没有取改变寄存器ESI中的值，而两次对MessageBoxA的调用都是call  ESI，所以在release版本下是不能HOOK的。因为在这种情况下.idata是只读的，所以编译器为了能够更快的调用API干脆在第一次调用时在找到导入项后就将其保存在了寄存器（通常是esi）中，所以尽管我们已经在导入表将MessageBoxA的地址覆盖掉了，但是SetHook执行之后任然会调用寄存器esi中保存的导入项。

那么，我们如何能够在release下成功hook呢？既然寄存器可以保存函数地址，而且寄存器也就这么几个，那么为了强迫PE装载器能够去导入表查找MessageBoxA的地址，我们可以写一些无关紧要的API去占用寄存器的空间，例如：

	GetProcessHeap();GetCurrentProcess();GetCurrentThread();GetThreadLocale();
	GetProcessHeap();GetCurrentProcess();GetCurrentThread();GetThreadLocale();
	GetProcessHeap();GetCurrentProcess();GetCurrentThread();GetThreadLocale();
	GetProcessHeap();GetCurrentProcess();GetCurrentThread();GetThreadLocale();
	GetProcessHeap();GetCurrentProcess();GetCurrentThread();GetThreadLocale();
	……
         再来用OD载入查看下它的反编译代码，如下：
	//保存寄存器的值
	00401020  /$  53            PUSH EBX
	00401021  |.  55            PUSH EBP
	00401022  |.  56            PUSH ESI
	00401023  |.  57            PUSH EDI
	00401024  |.  6A 00         PUSH 0               ; /Style = MB_OK|MB_APPLMODAL
	00401026  |.  68 38704000   PUSH 09HookDe.00407038        ; |Title = “HookDemo”
	0040102B  |.  68 30704000   PUSH 09HookDe.00407030        ; |Text = “原函数“
	00401030  |.  6A 00         PUSH 0                        ; |hOwner = NULL
	//调用第一次USER32.MessageBoxA
	00401032  |.  FF15 CC604000 CALL DWORD PTR DS:[<&USER32.MessageBoxA>]     ; \MessageBoxA
	//以下为垃圾代码
	00401038  |.  8B35 10604000 MOV ESI,DWORD PTR DS:[<&KERNEL32.GetProcessHe>];  kernel32.GetProcessHeap
	0040103E  |.  FFD6          CALL ESI              ; [GetProcessHeap
	00401040  |.  8B3D 0C604000 MOV EDI,DWORD PTR DS:[<&KERNEL32.GetCurrentPr>];  kernel32.GetCurrentProcess
	00401046  |.  FFD7          CALL EDI             ; [GetCurrentProcess
	00401048  |.  8B1D 08604000 MOV EBX,DWORD PTR DS:[<&KERNEL32.GetCurrentTh>];  kernel32.GetCurrentThread
	0040104E  |.  FFD3          CALL EBX             ; [GetCurrentThread
	00401050  |.  8B2D 04604000 MOV EBP,DWORD PTR DS:[<&KERNEL32.GetThreadLoc>];  kernel32.GetThreadLocale
	00401056  |.  FFD5          CALL EBP             ; GetThreadLocale
	00401058  |.  FFD6          CALL ESI              ; GetProcessHeap
	0040105A  |.  FFD7          CALL EDI             ; GetCurrentProcess
	0040105C  |.  FFD3          CALL EBX             ; GetCurrentThread
	0040105E  |.  FFD5          CALL EBP                 ; GetThreadLocale
	00401060  |.  FFD6          CALL ESI                  ; GetProcessHeap
	00401062  |.  FFD7          CALL EDI                 ; GetCurrentProcess
	00401064  |.  FFD3          CALL EBX                 ; GetCurrentThread
	00401066  |.  FFD5          CALL EBP                 ; GetThreadLocale
	00401068  |.  FFD6          CALL ESI                  ; GetProcessHeap
	0040106A  |.  FFD7          CALL EDI                 ; GetCurrentProcess
	0040106C  |.  FFD3          CALL EBX                 ; GetCurrentThread
	0040106E  |.  FFD5          CALL EBP                 ; GetThreadLocale
	00401070  |.  FFD6          CALL ESI                  ; GetProcessHeap
	00401072  |.  FFD7          CALL EDI                  ; GetCurrentProcess
	00401074  |.  FFD3          CALL EBX                  ; GetCurrentThread
	00401076  |.  FFD5          CALL EBP                  ; GetThreadLocale
	//////////////////////////////////////////////////////////////////////////////
	00401078  |.  6A 00         PUSH 0                    ; /pModule = NULL
	0040107A  |.  FF15 00604000 CALL DWORD PTR DS:[<&KERNEL32.GetModuleHandle>]; \GetModuleHandleA
	00401080  |.  50            PUSH EAX
	//SetHook
	00401081  |.  E8 4A000000   CALL 09HookDe.004010D0
	00401086  |.  83C4 04       ADD ESP,4
	00401089  |.  6A 00         PUSH 0            ; /Style = MB_OK|MB_APPLMODAL
	0040108B  |.  68 38704000   PUSH 09HookDe.00407038         ; |Title = “HookDemo”
	00401090  |.  68 30704000   PUSH 09HookDe.00407030         ; |Text = “原函数“
	00401095  |.  6A 00         PUSH 0                         ; |hOwner = NULL
	//第二次调用MessageBoxA
	00401097  |.  FF15 CC604000 CALL DWORD PTR DS:[<&USER32.MessageBoxA>]     ; \MessageBoxA
	0040109D  |.  5F            POP EDI
	0040109E  |.  5E            POP ESI
	0040109F  |.  5D            POP EBP
	004010A0  |.  5B            POP EBX
	004010A1  \.  C3            RETN
	……

可以看到，此时由于寄存器不够用，PE装载器不得不查找导入表调用MessageBoxA函数，既然此时我们已经hook掉了MessageBoxA，程序当然调用的就是我们自定义的函数咯。

<h3 style="text-align:right">——dycc</h3>
