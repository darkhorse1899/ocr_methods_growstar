
User: I'm using abbyy finereader to extract text from pdf files which sometimes have image-based pages inside.
and I'm controling this app via an MFC app that I built.
but the abbyy finereader does not support cli mode so I need to control it like automation.
how can I post a message from mfc app to abbyy finereader to click such buttons on the finereader app?

Assistant: Got it 👍 — since ABBYY FineReader doesn’t expose a **command-line interface (CLI)** and you’re not using the **FineReader Engine SDK**, you’ll need to fall back on **Windows GUI automation** techniques to control its UI (click buttons, send keystrokes, etc.) directly from your **MFC app**.  

Here are your main options:

---

### 1. **Win32 API: PostMessage / SendMessage**
If you know the **window handle (HWND)** of FineReader’s main window or a child control (like a button), you can send standard Windows messages to simulate clicks or keystrokes.

Example steps:
1. Use `FindWindow` or `FindWindowEx` to locate the FineReader window.
2. Enumerate child windows (using `EnumChildWindows`) to find specific controls.
3. Send a **BM_CLICK** (`SendMessage(hwndButton, BM_CLICK, 0, 0)`) to simulate pressing a button.

👉 Downside: FineReader may not use standard Win32 controls (it might be custom-drawn), so `BM_CLICK` may not always work.

---

### 2. **Keyboard/Mouse Simulation**
If FineReader uses custom controls that don’t respond to BM_CLICK:
- Use `SendInput` (preferred) or `keybd_event` / `mouse_event` to simulate user input.
- Steps:
  - Bring the FineReader window to the foreground (`SetForegroundWindow`).
  - Send keyboard shortcuts (if FineReader supports them).
  - Or move/click the mouse at button coordinates.

This is more brittle but often works when controls are custom.

---

### 3. **Microsoft UI Automation (UIA)**
Modern Windows apps (including FineReader) often support **UI Automation**:
- UIA lets you find controls by their automation ID, name, or type.
- You can programmatically **invoke** a button’s action (much safer than coordinate-based clicks).
- Works even if the app uses custom rendering.

In C++, you can use the **UIAutomationClient API** (`#include <UIAutomation.h>`).  
Workflow:
1. Initialize `CoInitialize`.
2. Create an `IUIAutomation` instance.
3. Use `ElementFromHandle(hwnd)` to wrap FineReader’s window.
4. Traverse descendants (buttons, menus).
5. Call `IUIAutomationInvokePattern::Invoke` on the button.

---

### 4. **Third-party Automation Libraries**
If you don’t want to deal with raw UI Automation:
- **AutoIt / AutoHotkey** → external scripts you can call from MFC.
- **pywinauto** (Python, but can be triggered from your app).
- **UIAutomationCore wrappers** (C++/C# libraries).

---

✅ **Recommended path:**  
If you want something **robust and not brittle**, go with **UI Automation (UIA)** instead of raw PostMessage. PostMessage only works reliably with standard Win32 buttons, which ABBYY might not use.

---

User: what does this mean? what does custom-drawn mean? even if it is custom-drawn element, it should be  deployed to run on windows os so it should be win32 control, shouldn't it?

Assistant: Great question 👍 Let’s break it down clearly.  

---

### 🔹 Standard Win32 control
Things like:
- `BUTTON`
- `EDIT`
- `LISTBOX`
- `STATIC`

These are classes provided by the Windows API.  
When an app uses them, you can get the handle (HWND) of that control with `FindWindowEx` and then send it standard messages like `BM_CLICK` or `WM_SETTEXT`, and it just works.

Example:
```cpp
HWND hButton = FindWindowEx(hParent, NULL, L"Button", L"OK");
SendMessage(hButton, BM_CLICK, 0, 0);
```

---

### 🔹 Custom-drawn control
Many modern apps (including ABBYY FineReader, MS Office, Adobe Acrobat, etc.) **do not use standard Win32 controls** for buttons, toolbars, menus, etc. Instead, they:
- Create a plain `HWND` (just a window surface).
- Then **paint the button graphics themselves** (GDI, GDI+, Direct2D, etc.).
- Handle mouse clicks manually in their window procedure (`WM_LBUTTONDOWN`, `WM_PAINT`, etc.).

To Windows, such a “button” is **not a `BUTTON` class control**.  
It’s just part of a big custom canvas. So if you call `SendMessage(hwnd, BM_CLICK, …)` → there’s no handler for it, nothing happens.

---

### 🔹 Why apps do this?
- More flexible UI design (icons, skins, ribbons).
- Multi-platform toolkits (Qt, .NET WPF, etc.) that don’t map directly to Win32 controls.
- Greater control over rendering and behavior.

---

### 🔹 Key takeaway
> Even though FineReader runs on Windows, not all its UI pieces are exposed as standard Win32 controls.  
> Some are just custom-painted rectangles that look like buttons but don’t respond to `BM_CLICK`.

That’s why:
- **PostMessage/SendMessage works only if the control is a true Win32 control**.  
- **If it’s custom-drawn, you need UI Automation or input simulation (SendInput / mouse click at coordinates).**

---

⚡ Question for you:  
Do you want me to show you how to **check whether a FineReader button is a real Win32 control** (so you can use `BM_CLICK`), or should I jump straight to a **UI Automation example** that works regardless of whether it’s custom-drawn?

---

User: then if I use UI Automation (UIA), should i know about the automation id of the elements like buttons what is like custom-drawn? how can I get the automation id of the elements?

Assistant: Exactly 👌 — you’ve got it.  

When you use **UI Automation (UIA)**, you don’t care if the element is a standard Win32 control or a custom-drawn one.  
UIA exposes a logical tree of elements (buttons, menus, lists, etc.), even if under the hood they’re just painted pixels.  

---

## 🔹 Do you need the Automation ID?
Not necessarily.

With UIA, you can find elements by multiple properties:

- **AutomationId** → unique developer-assigned ID (if the app provides it).  
- **Name** → the text on the button (e.g., `"Open"`, `"Recognize"`, `"Save As"`).  
- **ControlType** → `Button`, `Edit`, `List`, etc.  
- **ClassName** → underlying window class (if relevant).  

👉 Many commercial apps (like FineReader) don’t bother setting AutomationIds, but you can usually grab the **Name** or **ControlType**.

---

## 🔹 How do you discover these properties?
You use a tool to **inspect the UI tree**:

- **Microsoft Inspect.exe** (part of Windows SDK)  
   - Lets you hover over UI elements and see their AutomationId, Name, ControlType, etc.
   - Location: `C:\Program Files (x86)\Windows Kits\10\bin\<version>\x64\inspect.exe`

- **Accessibility Insights for Windows** (modern UIA explorer by Microsoft).  
   - Easier to use, shows the same info as Inspect.exe.

With these tools, you point at the FineReader button and immediately see:
- Name: `"Recognize"`
- ControlType: `Button`
- AutomationId: (maybe blank)

---

## 🔹 Example UIA C++ workflow
1. Launch FineReader.  
2. Run **Inspect.exe**.  
3. Hover over the button you want.  
   - If it shows **Name = "Open"** and **ControlType = Button**, you can use those properties in your search.  

---

## 🔹 Code sketch (C++ UIA)
```cpp
#include <UIAutomation.h>
#pragma comment(lib, "uiautomationcore.lib")

void ClickFineReaderButton(HWND hFineReader, LPCWSTR buttonName)
{
    CComPtr<IUIAutomation> pAutomation;
    CoInitialize(NULL);
    CoCreateInstance(CLSID_CUIAutomation, NULL, CLSCTX_INPROC_SERVER, IID_PPV_ARGS(&pAutomation));

    // Wrap FineReader window
    CComPtr<IUIAutomationElement> pRoot;
    pAutomation->ElementFromHandle(hFineReader, &pRoot);

    // Condition: Name == buttonName AND ControlType == Button
    CComPtr<IUIAutomationCondition> pCondition;
    VARIANT var;
    var.vt = VT_BSTR;
    var.bstrVal = SysAllocString(buttonName);

    CComPtr<IUIAutomationCondition> pNameCond;
    pAutomation->CreatePropertyCondition(UIA_NamePropertyId, var, &pNameCond);
    SysFreeString(var.bstrVal);

    CComPtr<IUIAutomationCondition> pTypeCond;
    pAutomation->CreatePropertyCondition(UIA_ControlTypePropertyId,
        CComVariant(UIA_ButtonControlTypeId), &pTypeCond);

    pAutomation->CreateAndCondition(pNameCond, pTypeCond, &pCondition);

    // Search in subtree
    CComPtr<IUIAutomationElement> pButton;
    pRoot->FindFirst(TreeScope_Subtree, pCondition, &pButton);

    if (pButton) {
        // Invoke click
        CComPtr<IUIAutomationInvokePattern> pInvoke;
        if (SUCCEEDED(pButton->GetCurrentPatternAs(UIA_InvokePatternId, IID_PPV_ARGS(&pInvoke)))) {
            pInvoke->Invoke();
        }
    }

    CoUninitialize();
}
```

---

✅ So the workflow is:  
1. Use **Inspect.exe** or **Accessibility Insights** to discover the element’s Name / AutomationId.  
2. Write your MFC/Win32 code to find that element and call `Invoke()`.  

---

User: which plugins or libraries should I install among microsoft visual studio installer if I wanna use inspect.exe and UIAutomation.h?

Assistant: Good one 👍 — this depends on what you want to do:  

You need **two things** here:  

---

## 1. **Inspect.exe (UI Inspector Tool)**  
This comes with the **Windows SDK**.  
- In the **Visual Studio Installer**, check **“Individual components”** → then select:  
  - ✅ **Windows 10 SDK** (or **Windows 11 SDK** if you’re on Win11).  

That installs `Inspect.exe` here (example path):  
```
C:\Program Files (x86)\Windows Kits\10\bin\<version>\x64\inspect.exe
```

---

## 2. **UIAutomation.h and libraries**  
These are also part of the **Windows SDK**.  

The files you need:  
- `UIAutomation.h` → headers  
- `uiautomationcore.lib` → import library for linking  

Location after installing SDK:  
```
C:\Program Files (x86)\Windows Kits\10\Include\<version>\um\UIAutomation.h
C:\Program Files (x86)\Windows Kits\10\Lib\<version>\um\x64\uiautomationcore.lib
```

So again → install **Windows 10 SDK** (or 11).  

---

## 3. **In Visual Studio Installer**
At minimum, check:  
- ✅ **Desktop development with C++** workload  
- ✅ **Windows 10/11 SDK** (under Individual Components)  

Optional but useful:  
- ✅ **C++ ATL/MFC support** (since you’re using MFC).  

---

## 4. **How to link in your MFC project**
In your `.vcxproj` or project settings:  
- **Include directories** should already point to the Windows SDK include path.  
- Add `uiautomationcore.lib` to **Project Properties → Linker → Input → Additional Dependencies**.  

Example:
```cpp
#include <UIAutomation.h>
#pragma comment(lib, "uiautomationcore.lib")
```

That’s it. No extra NuGet packages, no external DLLs — UIA is part of Windows itself.

---

✅ So:  
Just install **Desktop development with C++** + **Windows 10/11 SDK** in Visual Studio Installer.  
That gives you both `Inspect.exe` and `UIAutomation.h`.  
