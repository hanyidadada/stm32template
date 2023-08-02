# STM32 linux开发环境配置
## 一、安装必要软件
### 1.下载必要安装包
(1)VSCode linux安装包：https://code.visualstudio.com/
(2)Stm32cubeMX linux安装包：https://www.st.com/zh/development-tools/stm32cubemx.html#st-get-software
(3)gcc-arm-none-eabi 安装包：https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads
### 2.Vscode插件
```cmake C/C++ Arm Assembly Cortex-Debug```
![这是vscode插件图片](.\\pic\\pic1.png )
### 3.安装STM32CubeMX，打开安装包安装即可
### 3.OpenOCD烧录工具
使用如下命令安装
```
sudo apt install openocd
```
使用lsusb可以查看连接的烧录器信息
### 5.gcc-arm-none-eabi安装
#### (1)解压下载的安装包到需要的位置
![这是图片2](.\\pic\\pic2.png )
#### (2)然后根据解压的位置，添加路径到环境变量
方法一（进当前用户生效）：
```
cd ~
vi .bashrc
#打开.bashrc后把下面这句添加到最后面
export PATH=$PATH:/opt/arm-gnu-toolchain/bin
#保存退出后需要使环境变量生效：
source .bashrc
```
方法二（所有用户生效）：
```
sudo vi /etc/profile
#打开/etc/profile后把下面这句添加到最后面
export PATH=$PATH:/opt/arm-gnu-toolchain/bin
#保存退出后需要重启使环境变量生效：
sudo reboot
```
==注意：path中的bin前面的路径根据自己解压到的路径修改==
#### (3)环境变量生效后，命令行可输入命令得到版本信息，如下
![这是arm-gcc版本信息](.\\pic\\pic3.png )
#### (4)检查gdb运行状况
若出现如下提示，则安装libncursesw5库
```arm-none-eabi-gdb
arm-none-eabi-gdb: error while loading shared libraries: libncursesw.so.5: cannot open shared object file: No such file or directory
```
执行如下命令：
```
sudo apt install libncursesw5
```
gdb运行还需要python3.8环境库，若无法通过apt直接安装，那么可以通过[添加仓库](https://zhuanlan.zhihu.com/p/55250294)的方式安装或者[手动编译](https://blog.csdn.net/mziing/article/details/124475877)安装
## 二、生成工程代码及配置
### 1.STM32CubeMX生成项目基本代码框架
选择需要的型号，在SYS中选择使用的烧录器，必须选择正确
![这是图片4](.\\pic\\pic4.png )
项目管理中选择Toolchain为Makefile
![这是图片4](.\\pic\\pic5.png )
选择合适的位置生成项目
### 2.VSCode打开生成后的项目
配置C/C++选项，define选项，按下快捷键Ctrl+Shift+p，输入C/C++然后点击Edit Configurrations(JSON)
![这是图片4](.\\pic\\pic6.png )

其中需要定义的选项与Makefile中的定义一致，以此保证VSCode不会报错

![这是图片4](.\\pic\\pic7.png )

根据需要编写源文件，修改后如有添加文件，记得添加到Makefile文件相应位置
编写完成后，make执行编译，生成elf，hex和bin文件

![这是图片4](.\\pic\\pic8.png )

## 三、OpenOCD烧录
首先烧录器连上STM32并打开电源，另外单独打开一个终端，输入以下指令
```
openocd -f interface/cmsis-dap.cfg -f target/stm32f1x.cfg
```
cmsis-dap.cfg 对应烧录器
支持的烧录器在/usr/share/openocd/scripts/interface
stm32f1x.cfg对应MCU，支持的芯片在/usr/share/openocd/scripts/target
烧录器成功连接上后回到VSCode刚才的内置终端，分别输入
```
telnet localhost 4444 通过telnet连接openocd
program {项目路径}/build/test01.hex 烧录hex文件
reset 复位STM32
exit 关闭连接
```
## 四、调试配置
主要涉及两个文件的编辑
### 1.launch.json
```
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Cortex Debug",
            "cwd": "${workspaceRoot}",
            "executable": "./build/stm32template.elf", // path of executable file
            "request": "launch",
            "type": "cortex-debug",
            "runToEntryPoint": "main",
            "servertype": "openocd",
            "configFiles": [
                "interface/cmsis-dap.cfg", // connector of your usb, config file at /usr/share/openocd/scripts
                "target/stm32f1x.cfg" // mcu of your board
            ],
            "armToolchainPath": "/opt/arm-gnu-toolchain/bin", // armToolChain path
            "svdFile": "STM32F103xx.svd",  //svd文件
            "preLaunchTask": "Build"
        }
    ]
}
```
### 2.tasks.json
```
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "options":{
		"cwd":"${workspaceFolder}" //cwd的作用是切换到目标路径下
    },
    "tasks": [
		{
			"label": "Build",
			"type": "shell",
			"command": [
				"make"
			],
			"args": [
				"-f",
				"${workspaceFolder}/Makefile"
			],
			"problemMatcher":"$gcc",
			"group":{
				"kind":"build",
				//"isDefault":true // 当选项为true时ctrl+b会自动执行该条任务，无法选择其他任务
			},
			"detail":"build"
		},
		{
			"label": "download",
			"type": "shell",
			"command": [
				"openocd"
			],
			"args": [
				"-f",
				"interface/cmsis-dap.cfg",
				"-f",
				"target/stm32f1x.cfg",
				"-c",
				"program build/${workspaceFolder}.elf verify reset exit"
			],
			"problemMatcher":"$gcc",
			"group":"build",
			"detail":"download"
		}
	]
}
```
### 3.添加芯片对应的svd文件到项目文件夹
在<[这个链接](https://github.com/posborne/cmsis-svd)>寻找对应的的svd文件。CMSIS-SVD是CMSIS的一个组件，它包含完整微控制器系统（包括外设）的程序员视图的系统视图描述 XML 文件。简单来说，VS Code可以通过它来知道外设寄存器的地址分布，从而把寄存器内容展示到窗口中。


## 注意
若调试时，一直提示
```
Failed to launch OpenOCD GDB Server: Timeout.
```
可能是openocd版本不正确，请更新openocd，详情见[链接](https://blog.csdn.net/qq_40839071/article/details/114700646)>