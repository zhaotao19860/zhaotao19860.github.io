---
title: pclint
date: 2021-02-14 12:40:00 +0800
category: Tools
---

pclint提供c/c++的静态代码检查功能，不需要编译，检查变量定义问题及代码逻辑问题，是一款优秀的商业软件。

### 下载
```bash
https://gimpel.com/
```
### 安装
```bash
一路next，不生成配置
```
### 基本使用
```bat
lint-nt.exe options file1 [ file2 file3... ]
其中：options/files都可以放入lnt文件，方便批量处理。
```
### 批量处理
    - 安装系统头文件(c:/lint/include/linux)
      ```bash
      从linux系统中下载(需要啥下啥)：
      \usr\include
      \usr\include\linux
      \usr\include\hiredis
      \usr\lib\gcc\x86_64-redhat-linux\4.9.2\include
      \dpdk\x86_64-native-linuxapp-gcc\include
      ```
    - 手动配置各种lnt文件
      ```bash
      std.lnt     //包含其他lnt文件及全局配置如告警级别(根据需要修改)
      option.lnt  //包含pclint的各种选项，如-e禁止生成某类错误信息，+e恢复生成某类错误信息(根据需要修改)
      co-gcc.lnt  //生成lint_cmac.h相关选项的种子文件(直接使用)
      co-gcc.h    //gcc头文件(直接使用)
      lint_cmac.h //gcc相关宏定义(根据co_gcc.lnt说明生成)
      include.lnt //指定源文件及库文件的头文件路径
      src.lnt     //指定需要检查的源文件名
      ```
	- 配置脚本(config.bat)
	  ```bat
	  set /p lint_dir=PcLint install path:
	  set /p include_dir=Dependency header file download path:

	  git clone git@xxx:xxx/include.git %include_dir%
		
	  @echo off
	  
	  ::start.bat
	  echo %lint_dir%\lint-nt.exe -i".\config" -u std.lnt      .\config\adns.lnt ^> .\result.txt > .\start.bat
	  echo.       >> .\start.bat
	  echo %lint_dir%\lint-nt.exe -i".\config" -u std.lnt .\config\adns-adm.lnt ^>^> .\result.txt >> .\start.bat
	  echo %lint_dir%\lint-nt.exe -i".\config" -u std.lnt .\config\adns-check.lnt ^>^> .\result.txt >> .\start.bat
	  
	  ::pclint config
	  echo -i"%include_dir%\linux\usr\include"                                        >  .\config\include.lnt
	  echo -i"%include_dir%\linux\usr\include\linux"                                  >> .\config\include.lnt
	  echo -i"%include_dir%\linux\usr\include\hiredis"                                >> .\config\include.lnt
	  echo -i"%include_dir%\linux\usr\lib\gcc\x86_64-redhat-linux\4.9.2\include"      >> .\config\include.lnt
	  echo -i"%include_dir%\dpdk\x86_64-native-linuxapp-gcc\include"                  >> .\config\include.lnt
	  echo.                                                                           >> .\config\include.lnt
	  echo +libdir(%include_dir%\linux\usr\include)                                   >> .\config\include.lnt
	  echo +libdir(%include_dir%\linux\usr\include\linux)                             >> .\config\include.lnt
	  echo +libdir(%include_dir%\linux\usr\include\hiredis)                           >> .\config\include.lnt
	  echo +libdir(%include_dir%\linux\usr\lib\gcc\x86_64-redhat-linux\4.9.2\include) >> .\config\include.lnt
	  echo +libdir(%include_dir%\dpdk\x86_64-native-linuxapp-gcc\include)             >> .\config\include.lnt

	  @pause
	  ```
    - 启动pclint(start.bat-执行config.bat生成)
      ```bat
      c:/lint\lint-nt.exe -i".\config" -u std.lnt .\config\adns.lnt > .\result.txt 
      c:/lint\lint-nt.exe -i".\config" -u std.lnt .\config\adns-adm.lnt >> .\result.txt 
      c:/lint\lint-nt.exe -i".\config" -u std.lnt .\config\adns-check.lnt >> .\result.txt 

      -i:指定所有配置文件所在路径
      -u:指定各种lnt文件
      std.lnt:标准配置文件
      adns.lnt:需要检查的代码文件
      ```
    - 各种lnt文件说明
        - std.lnt(标准配置文件)
          ```
          //  Unix C, C++, -si4 -sl4 -sp8, 
          //  Standard lint options

          .\options.lnt  -si4 -sl4 -sp8
          .\co-gnu.lnt
          .\co-gcc.lnt

          //本程序头文件路径
          -i"..\include\common"
          -i"..\include\datapath"
          -i"..\src\adns"
          -i"..\src\common"
          -i"..\src\libadns"
          //系统头文件路径
          .\include.lnt

          //pclint告警级别
          //-w0 不产生信息（除了遇到致命的错误） 
          //-w1 只生成错误信息 没有告警信息和其它提示信息 
          //-w2 只有错误和告警信息 
          //-w3 生成错误、告警和其它提示信息（这是默认设置） 
          //-w4 生成所有信息
          -W2

          //外部依赖 不告警
          -wlib(0)

          //输出文件路径
          //-os(.\result.txt)
          ```
		- include.lnt(用于指定系统头文件)
		  ```
		  -i"c:/lint/include\linux\usr\include"                                   
		  -i"c:/lint/include\linux\usr\include\linux"                             
		  -i"c:/lint/include\linux\usr\include\hiredis"                           
		  -i"c:/lint/include\linux\usr\lib\gcc\x86_64-redhat-linux\4.9.2\include" 
		  -i"c:/lint/include\dpdk\x86_64-native-linuxapp-gcc\include"             
		  +libdir(c:/lint/include\linux\usr\include)                               
		  +libdir(c:/lint/include\linux\usr\include\linux)                         
		  +libdir(c:/lint/include\linux\usr\include\hiredis)                       
		  +libdir(c:/lint/include\linux\usr\lib\gcc\x86_64-redhat-linux\4.9.2\include) 
		  +libdir(c:/lint/include\dpdk\x86_64-native-linuxapp-gcc\include)   
		  ```
		- adns-adm.lnt(需要检查的代码文件)
		  ```
		  ..\src\adns_adm\adns_adm.c
		  ..\src\adns_adm\adm_lib.c
		  ```
		- options.lnt(配置可忽略的告警或其他选项)
		  ```
		  //    options.lnt
		  -e716 // allow use while (1)

		  -efile(537,stdarg.h) // bug in syslog.h, it define __need_va_list_ macro
		  -efile(451, stdarg.h, stdlib.h, stdio.h, assert.h, time.h, getopt.h) // no include guard present

		  -emacro(530,va_start) // do not init first parameter

		  -e801 //allow use goto
		   -e641 //allow convert enum to int
		  -e413 //空指针异常
		  -e613 //空指针异常
		  -e537 //头文件重复引用
		  -e534 //返回值可以忽略

		  -emacro(506, RTE_LOG)

		  -emacro(644, __bswap_16)
		  -emacro(644, __bswap_32)

		  -esym(516, __builtin_constant_p) //gnu check whether its argument evaluates to a constant value
		  -esym(530, __s2_len, __s1_len)  //strcmp, strncmp Symbol 'xxx' notinitialized
		  -esym(529, __d0, __d1)
		  -esym(449, cfg) //cfg_file.c  209  Warning 449: Pointer variable 'cfg' previously deallocated
		  -esym(593, cfg) //cfg_file.c  212  Warning 593: Custodial pointer 'cfg' (line 92) possibly not freed or returned

		  // gnu function
		  -esym(578, index)
		  -esym(578, socket)
		  -esym(578, link)

		  +fan //ANonymous union flag
		  ```