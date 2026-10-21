# pcl-amend
PCL2 修复工作报告

对象：Plain Craft Launcher 2（开源分支 2.99.0.0，源码包 mimoclaw_workspace.tar-1.gz）
时间：2026-10-20 ～ 2026-10-22 凌晨
执行：OpenClaw main 会话（DeepSeek / MiMo 模型），源码位于 C:\Users\Administrator\mcwork\PCL
最终交付：C:\Users\Administrator\Desktop\Plain Craft Launcher 2 - 修复版.exe（5,885,952 字节，SHA256 E127B3AC0C43C39A5F0B412EEE21D8DB14B1FE5A45877B25F0A429458786475D，版本 2.99.0.0，启动实测 140MB 内存 / 正常出窗）



一、任务背景

用户提供的开源分支源码包中，MC 下载、配置继承、主题切换、百宝箱、正版登录等功能被移除或清空为占位桩，且包内 exe 是未修改的原版。要求全部修复，并取消开源版本的启动提示弹窗。


二、修复内容

1. MC 下载功能（10-20 分析，10-21 编译验证）


源码侧已有 6 处改动（FormMain / ModModpack / ModSecret / PageDownloadInstall / PageDownloadModify / PageDownloadCompDetail），全仓库无 RestrictMcDownload 残留

但包内 exe 是原版，缺编译产物 → 核心工作是补齐编译链并完成构建

编译环境（不动系统全局）：
.NET 9 SDK → C:\Users\Administrator\dotnet9
.NET 4.8 引用程序集（NuGet Microsoft.NETFramework.ReferenceAssemblies.net48）→ C:\Users\Administrator\dotnet-local\refasm
VS BuildTools 18 MSBuild + MSBuildSDKsPath 指向 dotnet9 + MSBuildEnableWorkloadResolver=false

阶段交付：Plain Craft Launcher 2 - MC下载已恢复.exe


2. 配置继承


根因：ModSecret.vb 中 RegFolder = "PCLDebug"，官方为 "PCL"，两套注册表互不相通

隐藏风险：开源版 GetIdentify() 是返回全 0 的桩，而 DES 加密设置项的密钥由识别码派生——若只改目录名，解密失败会把官方加密设置重置覆盖

解法：先实现官方识别码算法，再改目录

验证：识别码 7624-91D6-9A1F-8DC0 → 密钥 92367793 → 成功解出官方数据 UiLauncherThemeHide2 = "2|0|1|3|4"、SystemCount = "7"

改动前注册表已备份至 mcwork\registry-backup\


3. 主题切换


根因两层：ThemeCheckAll / ThemeCheckOne / ThemeUnlock 全为空实现；且 ThemeRefresh 从不写 ColorHue（恒为 210），导致所有主题同一颜色

修复：新增 ThemeHue() 14 项色相表 + ThemeApplyColor()（含玄素黑压暗、橡木棕降饱和、欧皇彩提饱和、自定义 HSL 读取）+ 完整三个解锁/校验函数 + 标题装饰图（Images/Themes/*.png 10 张，原本闲置）


4. 百宝箱


PageOtherTest 原为提示桩，XAML 仅占位文本；整页重写：
今日人愿：按日期 + 识别码计算百分制
内存优化：真实调用 NtSetSystemInformation（信息类 80，操作 2/3/4），申请 SeProfileSingleProcessPrivilege，非管理员自动提权
清理垃圾：游戏 logs / crash-reports / debug / .mixin.out / hs_err_pid / natives / PCL 缓存，带二次确认
自定义下载：直链下载至缓存目录
随机一言


5. 正版登录


核查结论：登录代码从未被删。MsLoginStep1New / 1Refresh / 2 / 3 / 4 / 5 / 6 六步完整；设备码轮询、错误分支（封禁 / 未成年 / 区域限制 / 429 / 404）齐全；PageLoginMs 与官方反编译方法一一对应

唯一缺口：OAuthClientId 默认空串

提取：用 ilspycmd 反编译官方 exe（PCL.ModSecret），确认 OAuthClientId = "fe72edc2-3a6f-4280-90e8-e2beb64ce7e1"（Device Code 公共客户端，官方明文内嵌于 exe）

实测验证：POST login.microsoftonline.com/consumers/oauth2/v2.0/devicecode → HTTP 200，返回 user_code、verification_uri=https://www.microsoft.com/link、expires_in=900，ID 有效

写法：If(Environment.GetEnvironmentVariable("PCL_MS_CLIENT_ID"), "fe72edc2-...")，环境变量仍可覆盖


6. 启动提示弹窗


按用户要求整段删除 FormMain.xaml.vb 中 If VersionBranchMain = "OpenSource" Then MyMsgBox(...)

反编译确认全部文案（「已恢复的特性」「仍不可用的特性」「开源版本说明」等）从产物中消失，启动直接进主界面


三、验证记录

项目	方法	结果
Client ID 有效性	直接请求微软 devicecode 端点	✅ HTTP 200，下发设备码
登录六步代码	逐函数阅读 + 官方反编译对照	✅ 全部真实实现
识别码/DES 密钥	实解官方注册表数据	✅ 解出正常值
产物内容	ilspycmd 反编译 exe 检查代码	✅ 无提示文案，修复项齐全
程序可运行	启动实测	✅ 140MB，窗口正常
交付哈希	SHA256	✅ 已记录

未验证项：正版登录的端到端账号登录（需用户账号，不宜代操作）；GUI 内 MC 下载/整合包安装的实操点击。


四、仍不可用（外部凭证/服务限制，非代码问题）

功能	原因	变通
CurseForge API 直连	官方版同样只读环境变量 PCL_CURSEFORGE_API_KEY，无可提取的 Key	DlSourceModGet 已将 api.curseforge.com 重写至镜像 mod.mcimirror.top/curseforge，模组搜索下载基本可用
更新与联网通知	ServerConfig 全仓库从未赋值	无
联机功能	同上（依赖 ServerConfig）	无；PageLinkMain 仍提示「开源版无法联网获取配置」

五、踩坑与教训（已录入长期记忆）


VB 编译器把中文弯引号 “ ” 当字符串定界符 → 源码统一用 「」

MyImage.Source 是 String 类型，不能赋 BitmapImage

插值字符串占位符拼错（{vCrLf}）报 BC30451

源码文件一律无 BOM（UTF8Encoding($false)），否则可能引发 diff/编码问题

二进制字符串搜索验证 .NET 产物不可靠：字符串在元数据 #US 堆，UTF-16/ASCII/UTF-8 搜都可能假阴性 → 必须用 ilspycmd 反编译看代码

改注册表目录前必须先补密钥派生逻辑，否则解密失败会导致数据被重置覆盖

开源版桩函数无处不在，判断「功能是否真被删」要逐函数对照官方反编译，不能看注释或文件名下结论


六、交付物清单

文件	说明
桌面\Plain Craft Launcher 2 - 修复版.exe	最终版：MC 下载 + 配置继承 + 主题 + 百宝箱 + 正版登录，无启动弹窗
桌面\Plain Craft Launcher 2 - 主题百宝箱配置已恢复.exe	中间版（已被最终版取代，可删）
桌面\Plain Craft Launcher 2 - 配置主题百宝箱正版登录已恢复.exe	中间版（已被最终版取代，可删）
桌面\1\Plain Craft Launcher 2 - MC下载已恢复.exe	最早中间版（可删）
mcwork\registry-backup\	改动前注册表备份
mcwork\restore-mc-download.patch	MC 下载功能补丁

报告生成时间：2026-10-22 00:14 (Asia/Shanghai)
