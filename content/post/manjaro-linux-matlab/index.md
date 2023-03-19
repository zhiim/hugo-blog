+++

title = "Manjaro Linux 安装 MATLAB"
date = 2023-03-19T18:40:57+08:00
slug = "manjaro-linux-matlab"
description = "Manjaro Linux安装MATLAB时遇到的各种问题，包括无法启动安装程序、无法显示中文、无法保存设置、无法打开文本编辑器、界面过小、无法使用硬件加速"
tags = ["Linux"]
categories = ["Tech"]
image = ""

+++

最近从Windows换到了Manjaro，日常离不开MATLAB，虽然MATLAB官方提供了Linux版本，但是没有对Manjaro做适配，所以在安装MATLAB的过程中遇到了各种问题，以此记录。任何其他问题建议查阅[ArchWiki](wiki.archlinux.org)。
## MATLAB安装报错
最初安装的时候，我下载的是MATLAB 2020b的iso镜像，即使按照ArchWiki的指导，移除了`libfreetype.so*`还是会继续报错。后来换成MATLAB 2022a的镜像才成功安装。  
进入MATLAB 2022a安装文件所处的文件夹，运行`./install`，出现报错
```bash
terminate called after throwing an instance of 'std::runtime_error'
  what():  Failed to launch web window with error: Unable to launch the MATLABWindow application. The exit code was: 127
[1]    IOT instruction (core dumped)  ./install
```
此时可以运行
```bash
./bin/glnxa64/MATLABWindow 
```
应该会出现报错信息
```bash
bin/glnxa64/MATLABWindow: symbol lookup error: /usr/lib/libcairo.so.2: undefined symbol: FT_Get_Color_Glyph_Layer
```
`FT_Get_Color_Glyph_Layer`是freetype2里的一个符号，只要删除MATLAB安装文件夹里的`libfreetype.so*`，执行
```bash
rm ./bin/glnxa64/libfreetype.so*
```
或者也可以添加环境变量
```bash
LD_PRELOAD=/lib64/libfreetype.so
```
此时再执行`./install`就可以成功打开MATLAB安装引导。  
解决方案来源：[ArchWiki-MATLAB](https://wiki.archlinux.org/title/MATLAB#Unable_to_launch_the_MATLABWindow_application)
## MATLAB界面内中文无法显示
安装完成，打开MATLAB却发现中文乱码，所有的中文都是方块。MATLAB的界面是用JAVA开发的，中文无法显示是因为JAVA的中文字体配置问题。  
此时进入MATLAB的安装路径下的jre目录，我的MATLAB安装在`/usr/local/MATLAB/R2022a`，进入该文件夹
```bash
cd /usr/local/MATLAB/R2022a/sys/java/jre/glnxa64/jre/lib/fonts
```
在此目录下新建`fallback`文件夹
```bash
mkdir fallback
```
将中文字体移动到fallback文件夹，我使用的字体是NotoSansCJK-Regular.ttc，想要获取改字体可以先运行`sudo pacman noto-fonts-cjk`，然后在`/usr/share/fonts/noto-cjk`就可找到字体文件。
```bash
sudo cp /usr/share/fonts/noto-cjk/NotoSansCJK-Regular.ttc /usr/local/MATLAB/R2022a/sys/java/jre/glnxa64/jre/lib/fonts/fallback
```
复制完字体文件后，在`fallback`文件夹内，运行
```bash
mkfontscale
```
此时会生成`fonts.scale`，打开`fonts.scale`，将其中内容复制到上级目录`/usr/local/MATLAB/R2022a/sys/java/jre/glnxa64/jre/lib/fonts`中的`fonts.dir`  
注意`fonts.dir`没有可写权限，需要先授予可写权限
```bash
sudo chmod 6 fonts.dir
```
此时进入MATLAB就可以正常显示中文了。  
解决方案来源：[Linux下Matlab的安装和中文显示支持](https://developer.aliyun.com/article/398395)。  
如果MATLAB的终端无法显示中文，可以在MATLAB内`主页-预设-字体`下的两个字体选项里选择中文字体（如果设置无法保存请参考下节）。
## MATLAB的设置无法保存
在MATLAB的设置界面更改设置选项，发现退出MATLAB后在重新打开，所有的设置会恢复默认值，这是因为MATLAB无法使用/tmp文件夹。新建文件夹
```bash
mkdir ~/.cache/matlab-tmp
```
在`/home/xuq/.local/share/applications`为MATLAB建立一个启动方式`matlab.desktop`，文件内容为
```bash
[Desktop Entry]
Type=Application
Version=R2022b
Name=MATLAB
Comment=Scientific computing software
Icon=/home/<username>/.local/share/applications/matlab_logo.png
Exec=bash -c "export TMPDIR=/home/<username>/.cache/matlab-tmp; /usr/local/MATLAB/R2022a/bin/glnxa64/MATLAB -desktop; rm -r $TMPDIR"
Terminal=False
```
注意替换MATLAB可执行文件的路径，以及MATLAB的Icon路径。  
解决方案来源：[Why aren't my MATLAB preferences saved on Linux?](https://ww2.mathworks.cn/matlabcentral/answers/1806255-why-aren-t-my-matlab-preferences-saved-on-linux)。
## MATLAB无法打开文本编辑器
在MATLAB中新建脚本或者打开脚本时，出现报错`Unable to open this file in the current system configuration`，终端输出
```bash
Exception in thread "AWT-EventQueue-0": java.lang.NullPointerException
	at com.mathworks.mde.desk.MLDesktop.updateTemplate(MLDesktop.java:3665)
	at com.mathworks.mde.desk.MLDesktop.access$2000(MLDesktop.java:225)
	at com.mathworks.mde.desk.MLDesktop$NewMFileAction.actionPerformed(MLDesktop.java:2853)
	at com.mathworks.mwswing.ChildAction.actionPerformed(ChildAction.java:214)
	at javax.swing.AbstractButton.fireActionPerformed(AbstractButton.java:2022)
	at javax.swing.AbstractButton$Handler.actionPerformed(AbstractButton.java:2348)
	at javax.swing.DefaultButtonModel.fireActionPerformed(DefaultButtonModel.java:402)
......
```
此时只需要进入MATLAB的安装路径，去除`libfreetype.so.6`
```bash
cd /usr/local/MATLAB/R2022a/bin/glnxa64
mv libfreetype.so.6 libfreetype.so.6.old
```
解决方案来源：[MATLAB cannot create or open script files](https://bbs.archlinux.org/viewtopic.php?id=275084)。
## MATLAB界面缩放过小
MATLAB的界面、字体和图标在Linux下很小，无法随着系统缩放而改变。
以MATLAB界面放大1.25倍为例，在MATLAB终端输入
```bash
>> s = settings;s.matlab.desktop.DisplayScaleFactor
>> s.matlab.desktop.DisplayScaleFactor.PersonalValue = 1.25
```
解决方案来源：[Does MATLAB support High DPI screens on Linux?](https://ww2.mathworks.cn/matlabcentral/answers/406956-does-matlab-support-high-dpi-screens-on-linux)。
## MATLAB无法使用OpenGL加速
MATLAB绘图时终端输出
```bash
Warning: MATLAB previously crashed due to a low-level graphics error. 
To prevent another crash in this session, MATLAB is using software OpenGL instead of using your graphics hardware. 
To save this setting for future sessions, use the opengl('save', 'software') command. 
For more information, see Resolving Low-Level Graphics Issues.

```
设置由于MATLAB无法使用OpenGL硬件加速，导致图形渲染能力下降，在3D绘图时可能出错。  
进入MATLAB可执行文件所在的路径
```bash
cd /usr/local/MATLAB/R2022a/bin/glnxa64
MATLAB -nodesktop -nosplash -r "opengl info; exit" | grep Software
```
如果终端中输出的`software rendering`值不是`false`（0也不行）,说明此时未启用硬件加速。查看系统的OpenGL配置
```bash
glxinfo | grep "direct rendering"
```
如果`direct rendering`的值为`yes`说明配置正确，否则先要解决OpenGL的系统配置。如果系统的OpenGL配置正确，更改先前创建的MATLAB启动方式，在`/home/xuq/.local/share/applications`打开`matlab.desktop`，在Exec项添加MATLAB运行时的环境变量，将其更改为
```bash
Exec=bash -c "export TMPDIR=/home/xuq/.config/matlab-temp; export LD_PRELOAD=/usr/lib/libstdc++.so; export LD_LIBRARY_PATH=/usr/lib/dri/; /usr/local/MATLAB/R2022a/bin/glnxa64/MATLAB -desktop; rm -r $TMPDIR"
```
然后在MATLAB可执行文件所在的路径下新建文件
```bash
touch /usr/local/MATLAB/R2022a/bin/glnxa64/java.opts
```
在`java.opts`中写入`-Djogl.disable.openglarbcontext=1`。
解决方案来源：[ArchWiki-MATLAB](https://wiki.archlinux.org/title/MATLAB#OpenGL_acceleration)。