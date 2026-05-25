# Chromium-huge-memory-portable-build
Chromium 146 portable build with pointer compression disabled. Runs from any directory without installation,  supports >4GB heap. Based on Chromium M146 (v8 14.6), for high memory usage.

- 便携的、不受安装路径限制的、不受用户文件夹路径限制的、不受压缩指针内存限制的chromium。
- js堆内存不受4GB限制，可以在命令里自行修改限制。单个TypedArray最大2GB限制仍然存在。
- 便于使用一些基于js的内存开销巨大的网页。
- 如果使用原来的没关指针压缩的chromium的数据文件夹，可能闪退，需要删除浏览器数据、恢复设置，或使用下面命令里的自定义用户数据文件夹
- 初次使用本项目访问google可能会被认为是机器人。通过一些测试后恢复正常。
- Portable Chromium, independent of installation path, user directory, and disabled pointer compression.
- JS heap limit exceeds 4GB and is configurable via command line. The single TypedArray limit remains at 2GB.
- Optimized for memory-intensive web applications.
- Using existing data from a pointer-compressed Chromium may cause crashes. Clear browser data, reset settings, or use a custom user data directory as specified in the command.
- Initial access to Google may be flagged as bot activity. Normal access resumes after passing verification tests.

## Base Version
- **Chromium**: M146 (`146.0.7680.212`)
- **Git commit**: `fea2e0ad837e0`
- **v8**: 14.6.202.33

## run
```sh
chrome --js-flags="--max-old-space-size=65536 --no-compress-pointers" --user-data-dir=usrData
```

## Build

```sh
# 1. 克隆 depot_tools
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git

# 2. 配置环境变量
$env:PATH = "E:\prj\chromium\depot_tools;" + $env:PATH
$env:DEPOT_TOOLS_WIN_TOOLCHAIN = "0"

# 3. 配置 Git
git config --global core.autocrlf false
git config --global core.filemode false
git config --global core.preloadindex true
git config --global core.fscache true
git config --global branch.autosetuprebase always
git config --global core.longpaths true

# 4. 拉取 Chromium 源码
mkdir chromium && cd chromium
fetch chromium

# 5. 切换到目标版本
git checkout fea2e0ad837e0
gclient sync -D

# 6. 修复 libjpeg_turbo
cd src\third_party\libjpeg_turbo
git checkout d1f5f2393e0d51f840207342ae86e55a86443288
cd ..\..\..

# 7. 设置 VS 路径 (声称需要2022但实际使用2026)
$env:vs2022_install = "C:\Program Files\Microsoft Visual Studio\18\Community"

# 8. 配置构建
gn gen out\Release --args="is_component_build = true is_debug = false symbol_level = 0 blink_symbol_level = 0
v8_symbol_level = 0"

# 9. 编译
autoninja -C out\Release chrome

# 10. 生成安装包
autoninja -C out\Release mini_installer
```

## Build Configuration
```gn
is_component_build = false
is_debug = false
target_cpu = "x64"
symbol_level = 0
blink_symbol_level = 0
v8_symbol_level = 0
v8_enable_pointer_compression = false
```

The key change is `v8_enable_pointer_compression = false`, which removes the
4GB heap limit inherent in pointer compression, allowing Chromium to use
unlimited system memory.

## Source Code Modifications

### File: `sandbox/policy/win/sandbox_win.cc`

**Function**: `SandboxWin::AddAppContainerProfileToConfig()`

**Original code** (lines 781-795):
```cpp
  DWORD granted_access;
  BOOL granted_access_status;
  const base::FilePath program = command_line.GetProgram();
  bool access_check =
      config->GetAppContainer()->AccessCheck(
          program.value().c_str(), base::win::SecurityObjectType::kFile,
          GENERIC_READ | GENERIC_EXECUTE, &granted_access,
          &granted_access_status) &&
      granted_access_status;
  if (!access_check) {
    PLOG(ERROR) << "Sandbox cannot access executable " << program
                << ". Check filesystem permissions are valid. See "
                   "https://bit.ly/31yqMJR.";
    return SBOX_ERROR_CREATE_APPCONTAINER_ACCESS_CHECK;
  }
```

**Modified code**:
```cpp
  // Skip the access check for portable installations. The check simulates
  // whether an LPAC sandboxed process can access the exe, but fails when
  // the exe directory lacks the LPAC capability ACEs set by a formal install.
  // Since we already created the AppContainer profile and set up capabilities
  // above, the actual process will have the correct sandbox restrictions.
  const base::FilePath program = command_line.GetProgram();
  LOG(WARNING) << "Skipping AppContainer AccessCheck for " << program
               << ". LPAC capabilities are already configured.";
```

**Why this change**:

When Chromium is formally installed on Windows, the installer sets LPAC
(Low-Privilege App Container) capability ACEs on the executable directory.
The `AccessCheck` in this function is a pre-launch simulation that verifies
whether a sandboxed process would be able to access the exe file.

In a portable installation (extracted to any directory), these ACEs are not
present because no formal installation occurred. The simulation fails with
`SBOX_ERROR_CREATE_APPCONTAINER_ACCESS_CHECK`, causing a DCHECK and immediate
process termination -- specifically affecting the Network Service sandbox
initialization.

**Security impact**: None. This `AccessCheck` is a broker-side
configuration validation, not actual sandbox enforcement. The real sandbox
restrictions (AppContainer profile, LPAC capabilities, token levels, job
objects) are applied during `CreateProcess` and remain fully active. Skipping
this check merely removes a pre-flight assertion that cannot succeed in a
portable context.



## Contact

For questions or to request additional verification, please open an issue
in this repository.
