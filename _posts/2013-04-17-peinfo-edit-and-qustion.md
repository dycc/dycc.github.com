---
title: '关于PEinfo编写过程中遇到的各种问题'
layout: post
guid: urn:uuid:2031-04-17-peinfo-edit-and-qustion
tags:
    - Windows
    - PE
    - VC++6.0
---

在deit控件输出回车换行应该使用”\r\n”，如果使用”\n的话输出的是“|”

    wsprintf(szBuffer,”%s  %06X  %06X  %04X  %06X  %06X\r\n”,(TCHAR *)pSH->Name,pSH->Misc.VirtualSize,pSH->VirtualAddress,pSH->SizeOfRawData,pSH->PointerToRawData,pSH->Characteristics);
	strcat(lpText,szBuffer);
	wsprintf(szBuffer1,”0X%06X”,pDH1->e_lfanew);
	SetDlgItemText(hwnd,IDC_EDIT1,szBuffer1);//这个输出的是0x000000E0
	TCHAR szBuffer[64];
	strcpy(szBuffer,(char*)pDH);
	DialogBoxParam(NULL,MAKEINTRESOURCE(IDD_IMAGE_DOS_HEADER),hwnd,Image_dos_header_Proc,(LPARAM)szBuffer);
	/////////////////////////////////////////////////////////////////////////////////////////////////////////
	strcpy(szBuffer,(TCHAR*)lParam);
	pDH1=(IMAGE_DOS_HEADER*)szBuffer;
	wsprintf(szBuffer1,”0X%06X”,pDH1->e_lfanew);
	SetDlgItemText(hwnd,IDC_EDIT1,szBuffer1);//0XCCCCCCCC

为什么会这样？？？？？？？？？？？？？？？？？？！！！！！！！！！！！！！！！！！

从函数中返回时并最好不要返回数据地址，而是返回数据值，否则指向的这个地址会很容易被覆盖而出现莫名其妙的错误

如果要使用打开对话框选择文件，要添加下面的头文件

	#include “commdlg.h”
    const char *chs;
	chs=GetFileName();
	SetDlgItemText(hwnd,IDC_SHOW,chs);

定义保存文件名的指针变量chs时必须加上const，否则显示路径时会出现

    C:\Users\Administrator\Desktop\PEinfo\MainDlg.cpp(65) : error C2440: ‘=’ : cannot convert from ‘const char *’ to ‘char *’Conversion loses qualifiers

为什么？？？？？？？？？？？？？？？？？？？！！！！！！！！！！！！！！！！！！！！！！！！！

//打开文件句柄 -> 创建文件映像 -> 加载文件映像到当前进程空间

	HANDLE hFile=CreateFile(chs,GENERIC_READ,FILE_SHARE_READ|FILE_SHARE_WRITE,NULL,OPEN_EXISTING,FILE_ATTRIBUTE_NORMAL,NULL);
	HANDLE hMap=CreateFileMapping(hFile,NULL,PAGE_READONLY,0,0,NULL);
	void *lpFileAddress=MapViewOfFile(hMap,FILE_MAP_READ,0,0,0);
	//TCHAR szBuffer[64];
	TCHAR szBuffer1[64];
	//strcpy(szBuffer,(TCHAR*)lParam);
	pDH1=(IMAGE_DOS_HEADER*)lParam;

参数lParam可以直接转换为(IMAGE_DOS_HEADER*)，不能用strcpy将其赋值给szBuffer再转换为(IMAGE_DOS_HEADER*)，否者会出现转换错误

数据按类型和结构有不同的存储空间，转换地址引起的错误不一定是全部。

//这里要显示((&pOH3->Magic)-(&pDH4->e_magic))表示此字段RAV（虚拟偏移地址），但是应该结果为0X00000118=280D，超过了TCHAR表示的最,2^8=256，产生溢出，会丢弃进位，实际结果为8C=140，所以这种写法不对

	WORD i=((WORD)(&pOH3->Magic)-(WORD)(&pDH4->e_magic));//所以，要将两个地址强制转换为WORD及以上类型格式在进行计算才正确！！！
	wsprintf(szBuffer1,”0X%08X”,i);
	SetDlgItemText(hwnd,IDC_EDIT3,szBuffer1)

新建一个头文件，声明所有结构体变量，加到附加段文件头，如果直接输出&pOH3->MajorLinkerVersion=0X02E3011A，如果直接输出&pDH4->e_magic=0X00000000，但是如果输出((WORD)(&pOH3->MajorLinkerVersion)-(WORD)(&pDH4->e_magic))=0X0000011A

这是为什么？？？？？？？？？？？？？？？？？？？？？？？？？？？！！！！！！！！！！！！！！！！！！

struct.h为什么只能包含一次？？？？？？？

两次及以上包含会出现

	Compiling…
	Image_add_header.cpp
	Linking…
	Image_add_header.obj : error LNK2005: “struct _IMAGE_DOS_HEADER * pDH4″ (?pDH4@@3PAU_IMAGE_DOS_HEADER@@A) already defined in Image_optional_header.obj
	Image_add_header.obj : error LNK2005: “struct _IMAGE_NT_HEADERS * pNH4″ (?pNH4@@3PAU_IMAGE_NT_HEADERS@@A) already defined in Image_optional_header.obj
	Image_add_header.obj : error LNK2005: “struct _IMAGE_FILE_HEADER * pFH4″ (?pFH4@@3PAU_IMAGE_FILE_HEADER@@A) already defined in Image_optional_header.obj
	Image_add_header.obj : error LNK2005: “struct _IMAGE_OPTIONAL_HEADER * pOH4″ (?pOH4@@3PAU_IMAGE_OPTIONAL_HEADER@@A) already defined in Image_optional_header.obj
	Image_add_header.obj : error LNK2005: “struct _IMAGE_DATA_DIRECTORY * pDD4″ (?pDD4@@3PAU_IMAGE_DATA_DIRECTORY@@A) already defined in Image_optional_header.obj
	Image_add_header.obj : error LNK2005: “struct _IMAGE_EXPORT_DIRECTORY * pED4″ (?pED4@@3PAU_IMAGE_EXPORT_DIRECTORY@@A) already defined in Image_optional_header.obj
	Image_add_header.obj : error LNK2005: “struct _IMAGE_IMPORT_DESCRIPTOR * pID4″ (?pID4@@3PAU_IMAGE_IMPORT_DESCRIPTOR@@A) already defined in Image_optional_header.obj
	Image_add_header.obj : error LNK2005: “struct _IMAGE_RESOURCE_DIRECTORY * pRH4″ (?pRH4@@3PAU_IMAGE_RESOURCE_DIRECTORY@@A) already defined in Image_optional_header.obj
	Image_add_header.obj : error LNK2005: “struct _IMAGE_RESOURCE_DIRECTORY_ENTRY * pRcDE4″ (?pRcDE4@@3PAU_IMAGE_RESOURCE_DIRECTORY_ENTRY@@A) already defined in Image_optional_header.obj
	Image_add_header.obj : error LNK2005: “struct _IMAGE_RESOURCE_DATA_ENTRY * pRDaE4″ (?pRDaE4@@3PAU_IMAGE_RESOURCE_DATA_ENTRY@@A) already defined in Image_optional_header.obj
	Image_add_header.obj : error LNK2005: “struct _IMAGE_RESOURCE_DIR_STRING_U * pRDSU4″ (?pRDSU4@@3PAU_IMAGE_RESOURCE_DIR_STRING_U@@A) already defined in Image_optional_header.obj
	Image_add_header.obj : error LNK2005: “struct _IMAGE_SECTION_HEADER * pSH4″ (?pSH4@@3PAU_IMAGE_SECTION_HEADER@@A) already defined in Image_optional_header.obj
	Debug/PEinfo.exe : fatal error LNK1169: one or more multiply defined symbols found

执行 link.exe 时出错.

PEinfo.exe – 1 error(s), 0 warning(s)


（因为包含一次就定义一次，如果要在同一个项目中的不同的.c文件中使用全局变量或者其他文件定义过的变量，可以使用extern声明此变量已经在其他文件中定义过，不用重复定义，完美解决）

但是，使用多次extern声明其他文件的指针变量很可能引起其他问题，比如我在a窗口将全局指针变量p指定了一个地址，这时，我想在另一个窗口b使用此变量，那么，我必须先关闭a窗口，再打开b窗口，第一次打开使用p变量正常，但是我改变了p变量的值，这时我再次打开b窗口，p变量已经被改变，这时使用的就是被改变过的p变量的值，那么此时程序会有百分之99的可能崩溃，并且是运行时错误，再想要发现此错误就会变得非常非常困难，一定要注意并且避免这种情况的发生！！！！！！！！！！！！避免这种错误的最好的方法是当我们在第一次给这个全局变量复制之后就不要在改变这个变量的值了，也就是尽量不要使用+=，*=，-=，*=，++，–等等赋值运算符（纯属个人观点）

该如何创建全局结构体变量？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？？

我真的错了，我应该将PE结构信息先读取完后再显示的

变量申请后一定要先初始化，一定要一定要一定要！！！！！！！！！！！！！！！！！！！！！！！！！！！！

再不初始化变量我就是他妈的脑残！！！！！！！！！！！！！！！！！！！

在不关闭程序的前提下不能重复载入不同的含有输入表的文件，否则只能显示首先载入的文件的输入表信息，如何解决？？？？？？？？？？？？？？？？？？？？