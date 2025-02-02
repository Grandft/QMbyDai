# AHK 脚本

本课程使用 OBS Studio 录制并推流（直播）。借助轻量级的 AutoHotkey，可以花费相对少的精力对 OBS Studio 的功能做出一些有针对性的补充，提升直播和录课的质量。这里上传我们的脚本并留下适当说明，为通常计算机水平但对直播、录课质量略有要求的朋友们提供参考，同时也方便我们下一轮使用前复习用法。

## 功能说明

在我们的使用场景里，一台 Surface Pro 全屏运行 OneNote for Windows 10，它与手写笔配合充当黑板和粉笔。学生看到的画面是 Surface Pro 显示器的内容，听到的声音是老师说话的声音。我们使用 OBS Studio 推流到哔哩哔哩，同时录制得到视频文件。我们使用 AutoHotkey 要实现的功能是：学生只能看到全屏的 OneNote。

这件事并没有乍看上去那么简单。以下方法只要你用过，就会知道其瓶颈所在：

* 主流软件的指定窗口的屏幕共享
* 多显示器或虚拟桌面
* OBS Studio + 高级场景切换器

前两个方案的缺陷很难几句话描述准确，说得太多又偏离了重点。总之，最接近完美的是最后一个。高级场景切换器是 OBS Studio 的一个插件，基于它的功能，我们布置了两个场景：

* 第一个场景叫 onenote，由 OneNote 全屏这一条件触发，它的视频源是整个屏幕
* 第二个场景叫 outside，预设条件都不被满足时回落到它，即 OneNote 没全屏时的默认场景，它没有视频源

这样做的不完美之处主要在于，由于回落到 outside 是在 OneNote 脱离全屏后发生的，就会有一瞬间学生看到的是 OneNote 之外的东西。因此，原理上，要让回落在脱离全屏之前发生，那么脱离全屏所能引起的任何变化都不能用于触发回落。最简单的设想是有一个快捷键，它先触发回落，然后让 OneNote 脱离全屏，这就是使用 AutoHotkey 的初衷。

除此之外，我们还有更进一步的想法：让这个快捷键也有截图的动作，这样就可能做到紧随其后地用这张图片替换视频源，在学生眼里就是画面暂停了，这比突然全黑要好。至此我们的目标已描述清楚：

* 开始录制后、OneNote 全屏前，视频内容是黑屏
* OneNote 全屏时，视频内容是整个屏幕
* 按下快捷键后，先截图，再把视频源替换成图片，然后退出全屏
* 由于按下快捷键之外的原因意外退出 OneNote 全屏后，视频尽快黑屏

最后一条里包含了更高阶小的不完美，这种情况下前面提到的一瞬间 OneNote 之外的东西会出现在视频里。原则上这个问题可以通过缓存视频的思路解决，但真的操作起来想必性价比不高，因此忽略。除了上面明确的目标，下面的具体配置中包括了一些我们自己的偏好：

* 最小化到托盘
* 推流自动录像
* 录像不录光标
* 使用噪音阈值
* 不用全局音频源：使 outside 场景静音

这些内容不是重点，只有备忘性质，请按喜好取舍。

下面的内容相对细碎，给出的是可以照搬的步骤，而不是像上面的连贯思路。这样尽管简洁却略去了步骤之间的关联，让人操作起来摸不着头脑。我们建议先照搬，搬到最后对每一部分都有了印象，它们是如何联系的自然就会清晰起来。

## OBS Studio 的配置

安装软件和插件：

* [OBS Studio](https://obsproject.com/download)
* [InfoWriter v1.2](https://github.com/partouf/OBSInfoWriter/releases/tag/v1.2)
* [Advanced Scene Switcher](https://obsproject.com/forum/resources/advanced-scene-switcher.395)

InfoWriter 插件的功能是记录每次录制的起止时间和每次场景切换的时间，使用它有两个原因：一是方便封装时用这些信息分章节，二是为 AHK 脚本判断录制是否已经结束提供信息。这个插件有更新的版本，但并不适合我们，这里相关功能的准确实现也依赖于这个特定版本，如果使用新版需要对配置稍加调整。另外，高级场景切换器也在以不低的频率更新着，这里使用的是目前最新的版本 1.9 版。

全局设置，位于 OBS Studio 的文件 - 设置：

* 通用 - 输出  - 勾选 “推流时自动录像”
* 通用 - 系统托盘 - 勾选 “总是最小化到系统托盘，而不是任务栏”
* 音频 - 全局音频设备：全部禁用

创建临时文件，这样做的意义以后会清楚。按下微软徽标键 + PrtSc 截屏，找到图片的保存位置，一般在 C:\Users\用户名\Pictures\Screenshots。在这个路径下新建 4 个文本文档，分别改名为：

* log.txt
* ps.png
* status.txt
* ready.txt

注意也要修改扩展名。

布置场景：

* 场景 1 重命名为 onenote：添加来源 1 显示器采集，属性中不勾选 “显示鼠标指针”；添加来源 2 音频输入采集，属性中不勾选 “使用设备时间戳”，滤镜中添加噪音阈值；添加来源 3 InfoWriter，属性中第一项定位到上面的 log.txt
* 场景 2 重命名为 outside：添加来源 1 图像，属性中第一项定位到上面的 ps.png，勾选 “当不显示时卸载图像”

高级场景切换器的设置，位于 OBS Studio 的工具 - 高级场景切换器：

* 通用 - 状态 - 在 OBS 启动时：总是自动启动
* 通用 - 状态 - 自动启动场景切换器当：录制和推流
* 通用 - 一般表现 - 如果不符合任何切换条件 0.30 秒，切换到：outside
* 通用 - 优先级：文件内容 > 窗口标题 > 其他
* 窗口标题 - 添加一条：这里选正确的 OneNote 窗口标题 - onenote - 直接切换 - 全屏时
* 文件 - 将当前场景写入到这个文件：status.txt（定位到具体路径，这即是上面新建文本文档得到的 status.txt）
* 文件 - 根据文件内容切换场景 - 添加一条：如果本地文件 ready.txt（定位到具体路径，这即是上面新建文本文档得到的 ready.txt）匹配下列规则，使用转场特效直接切换切换到场景 outside：go

注意，这里的 OneNote 窗口标题是选择现有窗口得到的，是一字不差的严格匹配，如有不同需求建议使用正则表达式。OBS Studio 的配置到这里就齐全了，不过我们略去了对输出音视频参数的配置，请按需自行选取。

## AHK 脚本的原理

关闭再重新打开 OBS Studio，高级场景切换器就应该自动启动了。现在，AHK 脚本以外的准备都已就绪，我们开始关注脚本。

安装过 [AutoHotkey](https://www.autohotkey.com/download/ahk-install.exe) 之后，把这里提供的脚本 pause.ahk 放在 C:\Users\用户名\Pictures\Screenshots 下就可以直接运行，也可以编译得到可执行文件再在这个路径下运行。但是，直接运行未知的脚本是不安全的，本文的目的也在于提供参考而不是给出一个通用的成品，所以请用文本编辑器打开脚本，看一看它的内容。

脚本十分简陋，前四行含义明确，让脚本每次运行时删掉可能残留的旧的临时文件。

中间是一个循环，大致是说每秒钟去看一眼目录下有没有 png 图片，如果有，就看看现在 OBS Studio 处于哪个场景。

如果在 onenote 场景中，就说明这个图片可能是旧的了，已经用来唬过学生了。但也可能是刚刚生成的，正要用来唬人，只是此时场景还没来得及切换到 outside。于是等一秒，再看一眼，如果仍然处于 onenote，就排除了后一种情况，删掉图片。

如果在 outside 场景中，就去看看 InfoWriter 写的日志，看最后是不是说停止录制了，如果是，那么图片也是用旧了的，删掉。及时删掉用旧的图片，是为了防止视频在应该黑屏时没有黑屏，却显示了旧的截图。

最后一段是快捷键，这里设置的是 Ctrl + H。这部分的主体被一个判断裹挟，限制了快捷键仅在 onenote 场景下可用。这里把任务分成了几步：先清理旧文件，然后截屏，等待图片文件生成，然后改名，在 ready.txt 里写下 go 表示准备就绪，然后等待高级场景切换器被 go 触发直到发现 OBS Studio 确实已经切换到 outside 场景，最后 F11 退出全屏、删除 ready go。

## 用法

推流或录制时要确保 C:\Users\用户名\Pictures\Screenshots 下的 pause 脚本正在运行。讲课时，把 OneNote 撑开到全屏，学生可以看到屏幕。想要暂停时，按下 Ctrl + H，学生端维持静止画面直到下次把 OneNote 撑开到全屏。意外退出 OneNote 全屏时，学生端会在 0.3 秒之内切换到纯黑画面。

手写使用平板电脑时一般不外接键盘，此时建议连接一个具备可编程按键的鼠标，用以按下 Ctrl + H 或其他你设定的快捷键。

## chapter.ahk

这个脚本是一个例子，用来把 log.txt 里的日志转成 MKVToolNix 支持的章节文件。用法是复制 log.txt 里某对 START RECORDING 和 STOP RECORDING 之间的那些行到剪贴板，然后运行脚本，转换后的章节信息就会被写入同目录下的 chapter.txt。
