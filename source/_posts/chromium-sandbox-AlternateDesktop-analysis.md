---
title: Chromium-sandbox-AlternateDesktop-analysis
date: 2018-05-19 08:16:11
categories: 源码剖析
tags:
	- chromium
	- chromium-sandbox
---

本篇是sandbox源码剖析的第六篇，主要分析了windows平台下，Chromium对alternate desktop的创建与使用。阅读本篇前，最好阅读前四篇（本篇相对独立）。

想要流程的阅读本系列你需要以下几个条件：
1. 较高水平的C++编码能力（至少通读C++ Primer 5th，刷过课后题，有一定编码量）。
2. 熟悉Windows API编程，尤其是安全相关的内容。
3. 对二进制安全有一定了解，熟悉各类型安全漏洞成因、漏洞缓解措施及bypass手法。

<!--more-->

# chromium-sandbox-AlternateDesktop-analysis

policy三大组件最后一个——Alternate Desktop，也是最为简单的一个。关于target进程为什么要设置alternate desktop，在前面其他篇章中已经不止一次的说过了。

依然是围绕policy和target对alternate desktop的行为，对代码进行展开。

## `SpawnTarget` related

除了job和token，alternate desktop也是在这里捏出来的：

```cpp
 base::string16 desktop = policy_base->GetAlternateDesktop();
  if (!desktop.empty()) {
    startup_info.startup_info()->lpDesktop =
        const_cast<wchar_t*>(desktop.c_str());
  }
```

通过policy的`GetAlternateDesktop`接口拿到宽字符形式的desktop，将其指派到startup_info的对应成员。

与job和token的Make方法不同，这里的名称是Get：

```cpp
base::string16 PolicyBase::GetAlternateDesktop() const {
  // No alternate desktop or winstation. Return an empty string.
  // 如果policy在SpawnTarget前就已经指明了既不使用alternate desktop也不
  // 使用alternate winstation，就返回空串
  if (!use_alternate_desktop_ && !use_alternate_winstation_) {
    return base::string16();
  }

  // 如果要使用的是alternate winstation
  if (use_alternate_winstation_) {
    // The desktop and winstation should have been created by now.
    // If we hit this scenario, it means that the user ignored the failure
    // during SetAlternateDesktop, so we ignore it here too.
    // 理论上走到这里之前，desktop和winstation都已经创建完毕了
    // 所以如果走到这里的话，就意味着用户已经忽略了失败
    if (!alternate_desktop_handle_ || !alternate_winstation_handle_) {
      return base::string16();
    }
    // 利用winstation和desktop的handle来调用关键Call
    return GetFullDesktopName(alternate_winstation_handle_,
                              alternate_desktop_handle_);
  } else {
    // 这就意味着不适用winstation，只用desktop，但还需要对
    // alternate_desktop_local_winstation_handle_检查一下
    // 如果其无效则表示设置上出现了差错
    if (!alternate_desktop_local_winstation_handle_) {
      return base::string16();
    }
    // 关键Call，此时winstation句柄是null
    return GetFullDesktopName(nullptr,
                              alternate_desktop_local_winstation_handle_);
  }
}
```

展开关键Call看一看：

```cpp
base::string16 GetFullDesktopName(HWINSTA winsta, HDESK desktop) {
  if (!desktop) {	// desktop是不可能没有的，这辈子不可能没有的
    NOTREACHED();
    return base::string16();
  }

  base::string16 name;
  // 如果使用winsta，则返回winsta做参数调用返回的name
  if (winsta) {
    name = GetWindowObjectName(winsta);
    name += L'\\';	//后面跟了个反斜杠
  }

  name += GetWindowObjectName(desktop);//使用desktop做参数
  return name;
}
```

继续call in：

```cpp
base::string16 GetWindowObjectName(HANDLE handle) {
  // Get the size of the name.
  // 又是经典的二次调用，首次调用获取对象名称的尺寸
  DWORD size = 0;
  ::GetUserObjectInformation(handle, UOI_NAME, nullptr, 0, &size);

  if (!size) {
    NOTREACHED();
    return base::string16();
  }

  // Create the buffer that will hold the name.
  std::unique_ptr<wchar_t[]> name_buffer(new wchar_t[size]);

  // 二次调用获取对象名称
  // Query the name of the object.
  if (!::GetUserObjectInformation(handle, UOI_NAME, name_buffer.get(), size,
                                  &size)) {
    NOTREACHED();
    return base::string16();
  }

  return base::string16(name_buffer.get());
}
```

看到这里，再通过前面的注释提示，就明白了alternate desktop在`SpawnTarget`之前就已经做出来了，这也难怪job和token是Make，而alternate desktop却是Get。

## `SetAlternateDesktop` related

从AlternateDesktop的Create方法层层回溯，最终找到policy的`SetAlternateDesktop`方法。

sandbox本身没有new出policy对象来做这样一件事，这是使用sandbox的驱动者的事，但是sandbox有一些单元测试脚本，给出了调用的示范，其实此前在token中也看过了，比如：

```cpp
TEST(IntegrityLevelTest, TestLowILReal) {
  TestRunner runner(JOB_LOCKDOWN, USER_INTERACTIVE, USER_INTERACTIVE);

  runner.SetTimeout(INFINITE);

  runner.GetPolicy()->SetAlternateDesktop(true);//true表示使用winstation
  runner.GetPolicy()->SetIntegrityLevel(INTEGRITY_LEVEL_LOW);

  EXPECT_EQ(SBOX_TEST_SUCCEEDED, runner.RunTest(L"CheckIntegrityLevel"));

  runner.SetTestState(BEFORE_REVERT);
  EXPECT_EQ(SBOX_TEST_SUCCEEDED, runner.RunTest(L"CheckIntegrityLevel"));
}
```

于是，我们跟进`SetAlternateDesktop`看看：

```cpp
ResultCode PolicyBase::SetAlternateDesktop(bool alternate_winstation) {
  use_alternate_desktop_ = true;//这个肯定得true啊，还信誓旦旦搞个成员变量
  use_alternate_winstation_ = alternate_winstation;
  return CreateAlternateDesktop(alternate_winstation);
}
```

步入`CreateAlternateDesktop`：

```cpp
ResultCode PolicyBase::CreateAlternateDesktop(bool alternate_winstation) {
  // 需要winstation的分支
  if (alternate_winstation) {
    // Check if it's already created.
    // 防止二次create
    if (alternate_winstation_handle_ && alternate_desktop_handle_)
      return SBOX_ALL_OK;

    DCHECK(!alternate_winstation_handle_);
    // Create the window station.
    // 创建winstation的关键Call，传入的句柄作为OUT型参数
    ResultCode result = CreateAltWindowStation(&alternate_winstation_handle_);
    if (SBOX_ALL_OK != result)
      return result;

    // Verify that everything is fine.
    if (!alternate_winstation_handle_ ||
        GetWindowObjectName(alternate_winstation_handle_).empty())
      return SBOX_ERROR_CANNOT_CREATE_DESKTOP;

    // Create the destkop.
    // 创建desktop的关键Call
    result = CreateAltDesktop(alternate_winstation_handle_,
                              &alternate_desktop_handle_);
    if (SBOX_ALL_OK != result)
      return result;

    // Verify that everything is fine.
    if (!alternate_desktop_handle_ ||
        GetWindowObjectName(alternate_desktop_handle_).empty())
      return SBOX_ERROR_CANNOT_CREATE_DESKTOP;
  } else {
    // Check if it already exists.
    // 这个成员作为无winstation（意味着使用local winstation）的desktop句柄
    if (alternate_desktop_local_winstation_handle_)
      return SBOX_ALL_OK;

    // Create the destkop.
    // 第一个参数是nullptr
    ResultCode result =
        CreateAltDesktop(nullptr, &alternate_desktop_local_winstation_handle_);
    if (SBOX_ALL_OK != result)
      return result;

    // Verify that everything is fine.
    if (!alternate_desktop_local_winstation_handle_ ||
        GetWindowObjectName(alternate_desktop_local_winstation_handle_).empty())
      return SBOX_ERROR_CANNOT_CREATE_DESKTOP;
  }

  return SBOX_ALL_OK;
}
```

到此，就明白了这三个句柄成员的用途与联系了。

进入两个关键的Create中看看：

```cpp
ResultCode CreateAltDesktop(HWINSTA winsta, HDESK* desktop) {
  base::string16 desktop_name = L"sbox_alternate_desktop_";

  // 没有create winstation就接local_winstation_
  if (!winsta) {
    desktop_name += L"local_winstation_";
  }

  // Append the current PID to the desktop name.
  // 把当前进程的PID追加到desktop_name上
  wchar_t buffer[16];
  _snwprintf_s(buffer, sizeof(buffer) / sizeof(wchar_t), L"0x%X",
               ::GetCurrentProcessId());
  desktop_name += buffer;

  // 以当前线程ID获取当前的Desktop
  HDESK current_desktop = GetThreadDesktop(GetCurrentThreadId());

  if (!current_desktop)
    return SBOX_ERROR_CANNOT_GET_DESKTOP;

  // Get the security attributes from the current desktop, we will use this as
  // the base security attributes for the new desktop.
  // 从当前desktop获取安全属性，用来给新的desktop当模板
  SECURITY_ATTRIBUTES attributes = {0};
  if (!GetSecurityAttributes(current_desktop, &attributes))
    return SBOX_ERROR_CANNOT_QUERY_DESKTOP_SECURITY;

  // Back up the current window station, in case we need to switch it.
  // 获取当前的winstation
  HWINSTA current_winsta = ::GetProcessWindowStation();

  // 如果此前创建了winstation，就需要先切换到这个winstation上再创建desktop
  if (winsta) {
    // We need to switch to the alternate window station before creating the
    // desktop.
    // 这个Windows API负责切换进程的winstation
    if (!::SetProcessWindowStation(winsta)) {
      ::LocalFree(attributes.lpSecurityDescriptor);
      return SBOX_ERROR_CANNOT_CREATE_DESKTOP;
    }
  }

  // Create the destkop.
  // 此时就可以创建desktop了，名称已经填充好，标志也很明确
  // 具体意义参考MSDN，和对象安全访问有关
  *desktop = ::CreateDesktop(desktop_name.c_str(), nullptr, nullptr, 0,
                             DESKTOP_CREATEWINDOW | DESKTOP_READOBJECTS |
                                 READ_CONTROL | WRITE_DAC | WRITE_OWNER,
                             &attributes);
  ::LocalFree(attributes.lpSecurityDescriptor);

  if (winsta) {
    // Revert to the right window station.
    // 切回来
    if (!::SetProcessWindowStation(current_winsta)) {
      return SBOX_ERROR_FAILED_TO_SWITCH_BACK_WINSTATION;
    }
  }

  if (*desktop) {
    // Replace the DACL on the new Desktop with a reduced privilege version.
    // We can soft fail on this for now, as it's just an extra mitigation.
    // 替换新desktop的DACL，做了降权处理
    static const ACCESS_MASK kDesktopDenyMask =
        WRITE_DAC | WRITE_OWNER | DELETE | DESKTOP_CREATEMENU |
        DESKTOP_CREATEWINDOW | DESKTOP_HOOKCONTROL | DESKTOP_JOURNALPLAYBACK |
        DESKTOP_JOURNALRECORD | DESKTOP_SWITCHDESKTOP;
    // 涉及到对象DACL的意义以及存储结构
    // 可以自行参考《Windows Internals》的描述，阅读Acl.cc相关的代码
    // 理解SID和DACL之间判定方法、顺序影响等
    AddKnownSidToObject(*desktop, SE_WINDOW_OBJECT, Sid(WinRestrictedCodeSid),
                        DENY_ACCESS, kDesktopDenyMask);
    return SBOX_ALL_OK;
  }

  return SBOX_ERROR_CANNOT_CREATE_DESKTOP;
}
```

winstation的创建：

```cpp
ResultCode CreateAltWindowStation(HWINSTA* winsta) {
  // Get the security attributes from the current window station; we will
  // use this as the base security attributes for the new window station.
  // 老套路，先用当前的winstation的安全属性做模板
  HWINSTA current_winsta = ::GetProcessWindowStation();
  if (!current_winsta)
    return SBOX_ERROR_CANNOT_GET_WINSTATION;

  SECURITY_ATTRIBUTES attributes = {0};
  // 不过是GetSecurityInfo WinAPI的封装
  if (!GetSecurityAttributes(current_winsta, &attributes))
    return SBOX_ERROR_CANNOT_QUERY_WINSTATION_SECURITY;

  // Create the window station using nullptr for the name to ask the os to
  // generate it.
  // 不指定名称，由OS生成，创建出一个winstation
  *winsta = ::CreateWindowStationW(
      nullptr, 0, GENERIC_READ | WINSTA_CREATEDESKTOP, &attributes);
  if (!*winsta && ::GetLastError() == ERROR_ACCESS_DENIED) {
    *winsta = ::CreateWindowStationW(
        nullptr, 0, WINSTA_READATTRIBUTES | WINSTA_CREATEDESKTOP, &attributes);
  }
  LocalFree(attributes.lpSecurityDescriptor);

  if (*winsta)
    return SBOX_ALL_OK;

  return SBOX_ERROR_CANNOT_CREATE_WINSTATION;
}
```

