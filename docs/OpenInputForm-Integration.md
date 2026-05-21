# Hosting an Input Form Builder Form Inside Encompass — Exhaustive Rebuild Guide

This document is a **build-from-scratch specification** for recreating the
`OpenInputForm` feature of CommunityPlugin in a brand-new Encompass SDK
plugin. It is intentionally thorough: every type, every assembly, every
constructor parameter, every event registration, every gotcha that the
reference implementation in this repository encodes is written out
explicitly.

If you follow this document end-to-end and a step does not work, the
discrepancy is the bug — there is no implicit knowledge required beyond
what is written here.

---

## Table of Contents

1. [What the feature is and what the user sees](#1-what-the-feature-is-and-what-the-user-sees)
2. [Reference files in this repository](#2-reference-files-in-this-repository)
3. [Architecture overview](#3-architecture-overview)
4. [End-to-end runtime trace](#4-end-to-end-runtime-trace)
5. [Complete API surface — every Encompass type used](#5-complete-api-surface--every-encompass-type-used)
6. [Project scaffolding from zero](#6-project-scaffolding-from-zero)
7. [Encompass setup — custom fields, CDO, IFB form](#7-encompass-setup--custom-fields-cdo-ifb-form)
8. [The Plugin framework (optional but recommended)](#8-the-plugin-framework-optional-but-recommended)
9. [The host dialog — `QuickEntryPopupDialog2`, line-by-line](#9-the-host-dialog--quickentrypopupdialog2-line-by-line)
10. [The event-handler plugin — `OpenInputForm`, line-by-line](#10-the-event-handler-plugin--openinputform-line-by-line)
11. [Minimum-viable rebuild (no framework, ~120 lines)](#11-minimum-viable-rebuild-no-framework-120-lines)
12. [Permissions / `PluginAccess` (optional)](#12-permissions--pluginaccess-optional)
13. [Build, sign, deploy](#13-build-sign-deploy)
14. [Verification checklist](#14-verification-checklist)
15. [Gotchas — every one of them](#15-gotchas--every-one-of-them)
16. [Extending the pattern](#16-extending-the-pattern)

---

## 1. What the feature is and what the user sees

### 1.1 The user-facing behavior

1. A loan is opened in Encompass.
2. Anything — an advanced-code button on an IFB form, a business rule,
   a side-menu link, another plugin, or a manual edit in a field viewer
   — writes a form name into the custom field **`CX.OPENFORM`**.
3. Within ~one field-change cycle, a free-floating window opens. Its
   title is the form name (without the `pop` prefix). It contains the
   IFB form exactly as it would appear if the user had navigated to it
   via the loan's Forms tab list.
4. Edits made in the window write through to the live `CurrentLoan` —
   formulas recompute, business rules fire, calculations propagate to
   the main loan workspace, and switching back to the main tab shows
   the updated values.
5. `CX.OPENFORM` is blanked the moment the popup opens, so setting the
   same value again later re-opens the popup.
6. If the optional custom field **`CX.OPENFORM.SIZE`** holds a
   `width,height` string (e.g. `"800,600"`), the window opens at that
   size; otherwise the default is `600 × 600`.
7. When the user changes loans, the popup is closed automatically by
   the plugin's `LoanClosing` handler.
8. Multiple popups can be open at once. Whenever a field elsewhere
   changes, every open popup's `LoanScreen` is told to refresh its
   contents (unless it currently has focus, in which case refreshing
   would steal the user's caret).

### 1.2 Why use this at all?

Encompass already has `Macro.Popup(formName, title, w, h)`. It works,
but you do not own the window — you cannot subclass it, intercept
close, change the prefix used to identify it, control its
`StartPosition`, add buttons, or stop it from being modal. The
LoanScreen-based host gives you full WinForms control while still
using the same screen-rendering infrastructure Encompass uses
natively, so binding/formulas/rules behave identically to the
built-in tab.

---

## 2. Reference files in this repository

The feature exists in exactly two files. Everything else cited here
is shared infrastructure used by multiple plugins.

| Role | Path | Lines |
|------|------|-------|
| Event-handler plugin | `CommunityPlugin/Standard Plugins/OpenInputForm.cs` | 78 |
| Host dialog | `CommunityPlugin/Objects/QuickEntryPopupDialog2.cs` | 183 |

Supporting infrastructure used by `OpenInputForm`:

| Role | Path |
|------|------|
| Assembly entry point | `CommunityPlugin/PluginEntry.cs` |
| Plugin discovery / boot | `CommunityPlugin/Objects/Plugins.cs` |
| Plugin base class (event dispatch) | `CommunityPlugin/Objects/Plugin.cs` |
| Marker interface | `CommunityPlugin/Objects/Interface/ILoanOpened.cs` |
| Marker interface | `CommunityPlugin/Objects/Interface/IFieldChange.cs` |
| Marker interface | `CommunityPlugin/Objects/Interface/ILoanClosing.cs` |
| Reflection helper | `CommunityPlugin/Objects/Interface/InterfaceHelper.cs` |
| Permission gate | `CommunityPlugin/Objects/PluginAccess.cs` |
| Permission DTO | `CommunityPlugin/Objects/Models/PluginAccessRight.cs` |
| Field accessors | `CommunityPlugin/Objects/Helpers/EncompassHelper.cs` |
| Open-form tracker | `CommunityPlugin/Non Native Modifications/FormWrapper.cs` |
| Error logging | `CommunityPlugin/Objects/Logger.cs` |
| Settings DTO | `CommunityPlugin/Objects/Models/CDO.cs` |
| Settings (Encompass CDO loader) | `CommunityPlugin/Objects/Helpers/CDOHelper.cs` |
| Settings JSON (shipped to CDO) | `CommunitySettings.json` |
| Project file (refs & target) | `CommunityPlugin/CommunityPlugin.csproj` |
| App config (binding redirects) | `CommunityPlugin/app.config` |
| NuGet | `CommunityPlugin/packages.config` |
| Assembly info | `CommunityPlugin/Properties/AssemblyInfo.cs` |

---

## 3. Architecture overview

```
                  ┌─────────────────────────────┐
                  │   Encompass client process  │
                  └─────────────┬───────────────┘
                                │ loads plugins folder
                                ▼
                  ┌─────────────────────────────┐
                  │ MyPlugin.dll                │
                  │   [Plugin] PluginEntry()    │   ◄── EncompassObjects.dll
                  │       │                     │       attribute discovery
                  │       ▼                     │
                  │   Plugins.Start()           │
                  │       │  reflection         │
                  │       ▼                     │
                  │   for each Plugin sub-class │
                  │       Activator.CreateInst  │
                  │       p.Run()               │
                  └─────────┬───────────────────┘
                            │ Plugin.Run() inspects marker interfaces
                            │ and subscribes to EncompassApplication events
                            ▼
        ┌────────────────────────────────────────────────────┐
        │ OpenInputForm : Plugin, ILoanOpened, IFieldChange  │
        │                                                    │
        │   LoanOpened   ─► verify CX.OPENFORM exists        │
        │   FieldChanged ─► if CX.OPENFORM set:              │
        │                     resolve InputFormInfo          │
        │                     open QuickEntryPopupDialog2    │
        │                     blank CX.OPENFORM              │
        │                  else if Open:                     │
        │                     refresh open popups            │
        │   LoanClosing  ─► close every "pop*" form          │
        └─────────┬──────────────────────────────────────────┘
                  │
                  ▼
        ┌────────────────────────────────────────────────────┐
        │ QuickEntryPopupDialog2 : System.Windows.Forms.Form │
        │                                                    │
        │   ctor:                                            │
        │     this.Name = "pop" + form.Name                  │
        │     this.Size = sizeWidth × sizeHeight             │
        │     var screen = new LoanScreen(                   │
        │                    Session.DefaultInstance);       │
        │     this.Controls.Add(screen);                     │
        │     screen.LoadForm(formInfo);   ◄─── EMInput.dll  │
        └────────────────────────────────────────────────────┘
```

There is no MEF, no DI container, no XAML, no plug-in manifest beyond
the `[Plugin]` attribute. Discovery is `Assembly.GetTypes()` with a
`type.IsSubclassOf(typeof(Plugin))` filter.

---

## 4. End-to-end runtime trace

This is what happens, in order, the first time a user triggers the
feature after Encompass starts. Read this if anything is unclear after
sections 9 and 10.

1. **Encompass start.** Encompass discovers `MyPlugin.dll` in its
   plugin path (`%AppData%\Encompass\Plugins` or the shared deployment
   folder configured by Admin Tools). It scans for any class marked
   with the `[Plugin]` attribute from
   `EllieMae.Encompass.ComponentModel`.
2. **`PluginEntry` instantiated.** Encompass calls its parameterless
   ctor. We use the ctor to run our own bootstrap:

   ```csharp
   public PluginEntry() { Plugins.Start(); }
   ```

3. **`Plugins.Start()` discovers plugin classes.** It reflects on the
   currently-executing assembly:

   ```csharp
   foreach (Type t in assembly.GetTypes()
                              .Where(t => t.IsSubclassOf(typeof(Plugin))))
       (Activator.CreateInstance(t) as Plugin).Run();
   ```

   This means every subclass of `Plugin` is instantiated and run once.
   `OpenInputForm` is one of them.
4. **`OpenInputForm.Run()` (inherited from `Plugin`).** Inside
   `Plugin.Run()`, the base class checks which marker interfaces the
   subclass implements and wires Encompass events for each:
   - `ILoanOpened`     → `EncompassApplication.LoanOpened += Base_LoanOpened;`
   - `ILoanClosing`    → `EncompassApplication.LoanClosing += Base_LoanClosing;`
   - `IFieldChange`    → subscribed *inside* `Base_LoanOpened` (because
     the `Loan.FieldChange` event lives on the per-loan object, not on
     `EncompassApplication`).
5. **User opens a loan.** Encompass raises
   `EncompassApplication.LoanOpened`. `Base_LoanOpened` runs:
   - Captures `EncompassApplication.CurrentLoan` as `loan`.
   - Subscribes `loan.FieldChange += Base_FieldChange` (because
     `OpenInputForm` implements `IFieldChange`).
   - Calls the override: `OpenInputForm.LoanOpened(sender, e)`.
   - `LoanOpened` walks
     `EncompassApplication.Session.Loans.FieldDescriptors.CustomFields`
     and sets `HasFields = true` only if both `CX.OPENFORM` and
     `CX.OPENFORM.SIZE` exist in the schema.
6. **Some component sets `CX.OPENFORM = "Borrower Information"`.**
   The Encompass DataEngine raises `Loan.FieldChange`. Our
   `Base_FieldChange` calls the override
   `OpenInputForm.FieldChanged(sender, e)`.
7. **`FieldChanged` body.**
   - Returns immediately if `HasFields == false`.
   - If `e.FieldID == "CX.OPENFORM"` and `e.NewValue` is non-empty:
     1. `Session.FormManager.GetFormInfoByName(e.NewValue)` →
        `InputFormInfo form`. If `null`, return.
     2. Read `CX.OPENFORM.SIZE` via `EncompassHelper.Val(...)`. Split
        on `,`. If two integers, construct `Size`. Otherwise default
        `Size(600, 600)`.
     3. `new QuickEntryPopupDialog2(Session.LoanData, $"pop{form.Name}",
        form, width, height, FieldSource.CurrentLoan, "",
        Session.DefaultInstance).Show();`
     4. Set instance flag `Open = true;`
     5. `EncompassHelper.SetBlank("CX.OPENFORM");` — sets the field to
        empty so the next assignment fires `FieldChange` again.
   - Else if `Open == true` (we have at least one popup open),
     iterate every `Form` in `FormWrapper.OpenForms` whose `Name`
     starts with `pop`. For each:
     - Get its first child (`f.Controls[0]`) as `LoanScreen screen`.
     - If `screen == null`, bail.
     - If `screen.ContainsFocus == false` → `screen.RefreshLoanContents();`
     - Else → fall back to refreshing the main loan editor:
       `Session.Application.GetService<ILoanEditor>()?.RefreshContents();`
8. **Inside `QuickEntryPopupDialog2`'s ctor.**
   - `InitializeComponent()` builds an unused tab strip and OK/Close
     buttons left over from the original Encompass `QuickEntryPopupDialog`
     this class was forked from. **None of these controls participate
     in the form-rendering pipeline.** The only controls that matter
     are the ones added *after* `InitializeComponent()`:
     ```csharp
     this.Name = formTitle;                               // "popBorrower Information"
     this.Text = formTitle.Replace("pop","");             // "Borrower Information"
     this.Size = new Size(sizeWidth, sizeHeight);
     LoanScreen Screen = new LoanScreen(Session.DefaultInstance);
     this.Controls.Add(Screen);
     Screen.LoadForm(formInfo);
     ```
   - `LoanScreen` is Encompass's own loan-screen control (the same one
     each native tab uses).
9. **`Show()`.** Window appears modeless. The user can interact with
   both the popup and the main Encompass workspace freely.
10. **User edits the loan in the main tab.** The Encompass DataEngine
    fires `Loan.FieldChange` for every field write. Each event drops
    into `OpenInputForm.FieldChanged`. Because `Open == true` and the
    field is not `CX.OPENFORM`, the second branch runs — every open
    popup's `LoanScreen` is refreshed.
11. **User closes the loan or opens a different one.**
    `EncompassApplication.LoanClosing` fires. `Base_LoanClosing` calls
    `OpenInputForm.LoanClosing`, which iterates
    `FormWrapper.OpenForms` for `Name.StartsWith("pop")` and closes
    each.

---

## 5. Complete API surface — every Encompass type used

Each row is a type the rebuild requires, the namespace it lives in,
and the DLL it ships in. Every DLL listed lives in the Encompass
SmartClient cache (see §6.3).

### 5.1 Plugin discovery / lifecycle

| Member | Namespace | Assembly | Purpose |
|--------|-----------|----------|---------|
| `[Plugin]` attribute | `EllieMae.Encompass.ComponentModel` | `EncompassObjects.dll` | Marks `PluginEntry` so Encompass loads the assembly. |
| `EncompassApplication` (static) | `EllieMae.Encompass.Automation` | `EncompassAutomation.dll` | Top-level automation hub. |
| `EncompassApplication.Session` | `EllieMae.Encompass.Automation` | `EncompassAutomation.dll` | The automation `Session`. |
| `EncompassApplication.CurrentLoan` | `EllieMae.Encompass.Automation` | `EncompassAutomation.dll` | The open loan, or `null`. |
| `EncompassApplication.CurrentUser` | `EllieMae.Encompass.Automation` | `EncompassAutomation.dll` | The logged-in user. |
| `EncompassApplication.LoanOpened` (event) | `EllieMae.Encompass.Automation` | `EncompassAutomation.dll` | `EventHandler` — fires after loan open. |
| `EncompassApplication.LoanClosing` (event) | `EllieMae.Encompass.Automation` | `EncompassAutomation.dll` | `EventHandler` — fires before loan close. |
| `EncompassApplication.Login` (event) | `EllieMae.Encompass.Automation` | `EncompassAutomation.dll` | Fires after user login. Used for TabControl wire-up. |

### 5.2 Loan/field types

| Member | Namespace | Assembly | Purpose |
|--------|-----------|----------|---------|
| `Loan` | `EllieMae.Encompass.BusinessObjects.Loans` | `EncompassObjects.dll` | The open loan. |
| `Loan.FieldChange` (event) | `EllieMae.Encompass.BusinessObjects.Loans` | `EncompassObjects.dll` | Fires for every field write. `FieldChangeEventArgs e` has `e.FieldID`, `e.OldValue`, `e.NewValue`. |
| `FieldChangeEventArgs` | `EllieMae.Encompass.BusinessObjects.Loans` | `EncompassObjects.dll` | The event args. |
| `FieldDescriptor` | `EllieMae.Encompass.BusinessObjects.Loans` | `EncompassObjects.dll` | Schema entry per field. |
| `FieldDescriptors` (collection) | `EllieMae.Encompass.BusinessObjects.Loans` | `EncompassObjects.dll` | Indexed by FieldID. |
| `Session.Loans.FieldDescriptors.CustomFields` | `EllieMae.Encompass.BusinessObjects.Loans` | `EncompassObjects.dll` | Enumerable of `FieldDescriptor` for custom fields. |

### 5.3 The form-rendering chain

| Member | Namespace | Assembly | Purpose |
|--------|-----------|----------|---------|
| `Sessions.Session` | `EllieMae.EMLite.RemotingServices.Sessions` | `ClientSession.dll` | Instance type for a live session. |
| `Session.DefaultInstance` (static accessor) | `EllieMae.EMLite.RemotingServices` | `ClientSession.dll` | Returns the live `Sessions.Session` for the logged-in client. |
| `Session.LoanData` (static accessor) | `EllieMae.EMLite.RemotingServices` | `ClientSession.dll` | Returns the current loan as `IHtmlInput`. |
| `Session.FormManager` (static accessor) | `EllieMae.EMLite.RemotingServices` | `ClientSession.dll` | Form-catalog manager. |
| `IFormManager.GetFormInfoByName(string)` | `EllieMae.EMLite.RemotingServices` | `ClientSession.dll` | Returns the `InputFormInfo` for an IFB form by its name, or `null`. |
| `InputFormInfo` | `EllieMae.EMLite.InputEngine` | `EMInput.dll` | Metadata + payload for an IFB form. |
| `InputFormType.Custom` | `EllieMae.EMLite.InputEngine` | `EMInput.dll` | Used elsewhere to enumerate custom forms. |
| `LoanScreen` | `EllieMae.EMLite.ClientServer` | `Client.dll` | The screen-rendering control. Same one Encompass uses on built-in tabs. |
| `LoanScreen(Sessions.Session)` (ctor) | `EllieMae.EMLite.ClientServer` | `Client.dll` | Takes the session. |
| `LoanScreen.LoadForm(InputFormInfo)` | `EllieMae.EMLite.ClientServer` | `Client.dll` | Paints the form and binds it to the current loan. |
| `LoanScreen.RefreshLoanContents()` | `EllieMae.EMLite.ClientServer` | `Client.dll` | Re-pulls all field values from the loan and repaints. |
| `IHtmlInput` | `EllieMae.EMLite.ClientServer` | `ClientServer.dll` | Interface passed to the popup; the current loan satisfies it. |
| `FieldSource` (enum) | `EllieMae.Encompass.Forms` | `EncompassObjects.dll` | `FieldSource.CurrentLoan` is the standard value. |
| `ILoanEditor` | `EllieMae.Encompass.Client` | `EncompassObjects.dll` | Service interface for the main loan editor; `Session.Application.GetService<ILoanEditor>()`. Used for the focused-popup fallback refresh. |
| `EllieMae.EMLite.Common.UI.Controls.EMHelpLink` | `EllieMae.EMLite.Common.UI.Controls` | `ClientCommon.dll` (UI helpers) | Decorative control on the dialog from the original fork — not strictly needed for rebuild. |

### 5.4 Class-name collisions to watch for

There are **two** classes named `Session` in this namespace tree:

- `EllieMae.EMLite.RemotingServices.Session` — static class with
  `DefaultInstance`, `LoanData`, `FormManager`, `Application`, etc.
- `EllieMae.EMLite.RemotingServices.Sessions.Session` — the
  *instance* type, the actual session object the static accessor
  returns.

In files that do `using EllieMae.EMLite.RemotingServices;`, the
unqualified word `Session` resolves to the **static** class. The
instance type is referenced as `Sessions.Session` (the nested
namespace + class). Both `OpenInputForm.cs` and
`QuickEntryPopupDialog2.cs` use both.

Similarly there is `EllieMae.Encompass.Automation.EncompassApplication.Session`
(the **automation** session, an `EncompassObjects.dll` type). That is
a different type altogether — when you see code use both
`EncompassApplication.Session.Loans...` and `Session.DefaultInstance...`
in the same method, those are two distinct session objects. The
automation `Session` is the public-API wrapper; the static
`RemotingServices.Session` reaches into the unwrapped client
internals.

---

## 6. Project scaffolding from zero

### 6.1 Target framework

- **.NET Framework 4.7.2** for Encompass 19.x and later.
- Older Encompass versions (≤19.3) used 4.6.1; pick the framework
  matching the Encompass DLLs on the dev machine (see step 6.3 below).
- `OutputType` must be `Library` (DLL).
- `ProjectGuid` and `AssemblyName` can be anything — Encompass
  identifies plugins by the `[Plugin]` attribute, not by name.

### 6.2 Minimal `.csproj` shape

```xml
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Import Project="$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props" Condition="Exists(...)" />
  <PropertyGroup>
    <Configuration Condition=" '$(Configuration)' == '' ">Debug</Configuration>
    <Platform Condition=" '$(Platform)' == '' ">AnyCPU</Platform>
    <ProjectGuid>{NEW-GUID-HERE}</ProjectGuid>
    <OutputType>Library</OutputType>
    <RootNamespace>MyPlugin</RootNamespace>
    <AssemblyName>MyPlugin</AssemblyName>
    <TargetFrameworkVersion>v4.7.2</TargetFrameworkVersion>
    <Deterministic>false</Deterministic>
  </PropertyGroup>
  <!-- references in §6.3 -->
</Project>
```

This repo's `CommunityPlugin.csproj` uses the legacy
`Microsoft.NET.Framework` MSBuild XML schema (`ToolsVersion="15.0"`,
non-SDK style). You can use either the legacy form or the
SDK-style `<Project Sdk="Microsoft.NET.Sdk">` form — both build
identically.

### 6.3 Where to find the Encompass DLLs

On a developer machine that has run Encompass at least once, the
DLLs are deployed by ClickOnce to:

```
%LocalAppData%\Apps\2.0\<hash1>\<hash2>\<random>\<hash3>\SmartClientCache\Apps\UAC\Ellie Mae\<hash4>\Encompass360\
```

Easiest way to locate it: open `CommunityPlugin.csproj` from this
repo and look at any `<HintPath>` — it shows the relative path used
on the original author's box:

```
..\..\..\..\..\..\SmartClientCache\Apps\UAC\Ellie Mae\xIHR5EqGa7zPnRG0YpD5z4TPAB0=\Encompass360\<dll>
```

Adjust the relative path or copy the DLLs into a local `lib/` folder
in your repo. Do **not** check the DLLs into source control unless
your license allows it.

### 6.4 References required for *this feature only*

Only these references are necessary for `OpenInputForm` +
`QuickEntryPopupDialog2`:

```xml
<ItemGroup>
  <Reference Include="EncompassObjects"><HintPath>...\EncompassObjects.dll</HintPath><Private>False</Private></Reference>
  <Reference Include="EncompassAutomation"><HintPath>...\EncompassAutomation.dll</HintPath><Private>False</Private></Reference>
  <Reference Include="ClientSession"><HintPath>...\ClientSession.dll</HintPath><Private>False</Private></Reference>
  <Reference Include="ClientServer"><HintPath>...\ClientServer.dll</HintPath><Private>False</Private></Reference>
  <Reference Include="ClientCommon"><HintPath>...\ClientCommon.dll</HintPath><Private>False</Private></Reference>
  <Reference Include="Client"><HintPath>...\Client.dll</HintPath><Private>False</Private></Reference>
  <Reference Include="EMInput"><HintPath>...\EMInput.dll</HintPath><Private>False</Private></Reference>

  <Reference Include="System" />
  <Reference Include="System.Core" />
  <Reference Include="System.Drawing" />
  <Reference Include="System.Windows.Forms" />
</ItemGroup>
```

**`<Private>False</Private>` (i.e. `Copy Local = False`) is
mandatory** on every Encompass reference. If the built plugin folder
contains a second copy of `EMInput.dll`, `Client.dll`, or
`ClientSession.dll`, then `LoanScreen` will silently load the wrong
`InputFormInfo` type and `LoadForm` will either no-op or throw
`InvalidCastException`. The original repo's `csproj` enforces this.

### 6.5 `AssemblyInfo.cs`

The reference uses a stock `AssemblyInfo.cs` with:

```csharp
[assembly: AssemblyTitle("MyPlugin")]
[assembly: AssemblyProduct("MyPlugin")]
[assembly: ComVisible(false)]
[assembly: Guid("PUT-A-FRESH-GUID-HERE")]
[assembly: AssemblyVersion("1.0.*")]
```

Encompass does not require any specific assembly metadata. The
`Guid` is for COM interop and can be any fresh GUID.

### 6.6 `app.config`

If you reference `Newtonsoft.Json` (for settings loading), add the
binding redirect from this repo:

```xml
<configuration>
  <runtime>
    <assemblyBinding xmlns="urn:schemas-microsoft-com:asm.v1">
      <dependentAssembly>
        <assemblyIdentity name="Newtonsoft.Json" publicKeyToken="30ad4fe6b2a6aeed" culture="neutral" />
        <bindingRedirect oldVersion="0.0.0.0-12.0.0.0" newVersion="12.0.0.0" />
      </dependentAssembly>
    </assemblyBinding>
  </runtime>
</configuration>
```

Encompass ships its own `Newtonsoft.Json.dll` in the SmartClient
cache; the binding redirect ensures the runtime resolves to the
version Encompass loads, not the version the build copied next to
your DLL. If you do not use `Newtonsoft.Json`, omit this entirely.

---

## 7. Encompass setup — custom fields, CDO, IFB form

### 7.1 Custom fields

In Encompass: **Settings → Loan Setup → Custom Fields → New**.

| Field ID | Type | Format | Description |
|----------|------|--------|-------------|
| `CX.OPENFORM` | String | Any | Holds the name of the IFB form to open. |
| `CX.OPENFORM.SIZE` | String | Any | Optional. `width,height` (e.g. `800,600`). |

You can leave both fields off every form — the plugin reads/writes
them through the field collection, not through a visible control.

### 7.2 The IFB form(s) you want to open

Any input form **that exists in the Encompass form catalog** is
addressable. To find a form's internal name (which is what
`GetFormInfoByName` expects):

1. **Settings → Loan Setup → Input Form Builder**.
2. Right-click any form → **Properties** → the **Form Name** field
   is the value you pass to `CX.OPENFORM`.

A **form name is not the same as a form title**. Many forms have
identical names and titles, but custom IFB forms are usually
named with internal codes that differ from their display name.

### 7.3 Triggering the open

Anything that writes to a custom field works. The most common
triggers:

- **Advanced-code button** on another IFB form:
  ```vb
  Loan.Fields("CX.OPENFORM") = "Borrower Information"
  Loan.Fields("CX.OPENFORM.SIZE") = "800,600"
  ```
- **Business rule** — Field Trigger with action "Set Field" on
  `CX.OPENFORM`.
- **Another plugin** — `EncompassApplication.CurrentLoan.Fields["CX.OPENFORM"].Value = "Borrower Information";`

---

## 8. The Plugin framework (optional but recommended)

If you want the dispatch base class from this repo, here is what to
copy. These files are not strictly needed — you can wire
`EncompassApplication.LoanOpened` etc. by hand (see §11) — but the
base class is what lets `OpenInputForm` stay tiny.

### 8.1 `PluginEntry.cs`

```csharp
using EllieMae.Encompass.ComponentModel;

namespace MyPlugin
{
    [Plugin]
    public class PluginEntry
    {
        public PluginEntry()
        {
            Plugins.Start();
        }
    }
}
```

Encompass scans every loaded assembly for any type marked
`[Plugin]`. It instantiates each one (parameterless ctor) once per
client session. Doing your bootstrap in the ctor is the standard
pattern.

### 8.2 `Plugins.cs` — reflection-driven discovery

```csharp
using System;
using MyPlugin.Objects.Interface;

namespace MyPlugin
{
    public static class Plugins
    {
        public static void Start()
        {
            var i = new InterfaceHelper();
            foreach (Type type in i.GetAll(typeof(Plugin)))
                (Activator.CreateInstance(type) as Plugin).Run();
        }
    }
}
```

### 8.3 `InterfaceHelper.cs`

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace MyPlugin.Objects.Interface
{
    public class InterfaceHelper
    {
        public List<Type> GetAll(Type baseType)
        {
            try
            {
                return this.GetType().Assembly
                           .GetTypes()
                           .Where(t => t.IsSubclassOf(baseType))
                           .ToList();
            }
            catch (Exception ex)
            {
                Logger.HandleError(ex, nameof(InterfaceHelper));
                return null;
            }
        }
    }
}
```

### 8.4 Marker interfaces (only the ones `OpenInputForm` needs)

```csharp
namespace MyPlugin.Objects.Interface
{
    public interface IPlugin     { void Run(); }
    public interface ILoanOpened { void LoanOpened (object s, System.EventArgs e); }
    public interface ILoanClosing{ void LoanClosing(object s, System.EventArgs e); }
    public interface IFieldChange{ void FieldChanged(object s,
        EllieMae.Encompass.BusinessObjects.Loans.FieldChangeEventArgs e); }
}
```

### 8.5 `Plugin.cs` — the dispatcher

Only the slices needed for the OpenInputForm feature are shown here.
The full repo version handles ~15 interfaces in the same shape.

```csharp
using EllieMae.Encompass.Automation;
using EllieMae.Encompass.BusinessObjects.Loans;
using MyPlugin.Objects.Interface;
using System;

namespace MyPlugin.Objects
{
    public abstract class Plugin : IPlugin
    {
        public abstract bool Authorized();

        public virtual void Run()
        {
            if (!Authorized()) return;

            if (typeof(ILoanOpened).IsAssignableFrom(GetType()))
            {
                EncompassApplication.LoanOpened -= Base_LoanOpened;
                EncompassApplication.LoanOpened += Base_LoanOpened;
            }

            if (typeof(ILoanClosing).IsAssignableFrom(GetType()))
            {
                EncompassApplication.LoanClosing -= Base_LoanClosing;
                EncompassApplication.LoanClosing += Base_LoanClosing;
            }
        }

        public virtual void LoanOpened (object s, EventArgs e) { }
        public virtual void LoanClosing(object s, EventArgs e) { }
        public virtual void FieldChanged(object s, FieldChangeEventArgs e) { }

        private void Base_LoanOpened(object sender, EventArgs e)
        {
            Loan loan = EncompassApplication.CurrentLoan;
            if (loan == null) return;

            if (typeof(IFieldChange).IsAssignableFrom(GetType()))
            {
                loan.FieldChange -= Base_FieldChange;
                loan.FieldChange += Base_FieldChange;
            }

            LoanOpened(sender, e);
        }

        private void Base_LoanClosing(object sender, EventArgs e)
        {
            LoanClosing(sender, e);
        }

        private void Base_FieldChange(object sender, FieldChangeEventArgs e)
        {
            try { FieldChanged(sender, e); }
            catch (Exception ex) { Logger.HandleError(ex, "FieldChange"); }
        }
    }
}
```

The two important design choices encoded above (which match the
reference repo exactly):

- **`Loan.FieldChange` is subscribed inside `LoanOpened`, not in
  `Run()`.** The `FieldChange` event belongs to the per-loan object,
  not to `EncompassApplication`. You only get a `Loan` reference
  after `LoanOpened` fires. You don't need to unsubscribe explicitly
  — when the loan is released the event source is collected.
- **`-=` always precedes `+=`.** Encompass may raise `LoanOpened`
  multiple times in some flows (e.g. when an action re-opens the
  same loan after a re-import). Unsubscribing first prevents double
  handlers.

### 8.6 `Logger.cs` — stays out of the way

```csharp
using EllieMae.Encompass.Client;
using System;

namespace MyPlugin.Objects
{
    public static class Logger
    {
        public static void HandleError(Exception ex, string name, object data = null)
        {
            try
            {
                if (string.IsNullOrEmpty(name)) return;
                ApplicationLog.WriteError("MyPlugin",
                    $"{name}{Environment.NewLine}{ex}");
            }
            catch { /* swallow */ }
        }
    }
}
```

`ApplicationLog` is in `EncompassClient.dll` (referenced as
`Client` in the repo's csproj; the type lives in
`EllieMae.Encompass.Client`). Errors written here show in the
Encompass `ApplicationLog.xml` and surface to Encompass's normal
error logging UI.

---

## 9. The host dialog — `QuickEntryPopupDialog2`, line-by-line

`QuickEntryPopupDialog2.cs` is **183 lines but only ~10 do work.**
The rest is leftover scaffolding from the Encompass-internal
`QuickEntryPopupDialog` class that this one was originally forked
from. The unused leftovers (tabs `tabVOL`, `tabVOM`, `tabAdditional`,
buttons `btnOK`/`btnClose`, panels) are constructed inside
`InitializeComponent` but never added to the form's visible layout
in a useful way. You can delete all of it in your rebuild.

### 9.1 The lines that matter

From `Objects/QuickEntryPopupDialog2.cs`:

```csharp
public QuickEntryPopupDialog2(
    IHtmlInput        inputData,
    string            formTitle,
    InputFormInfo     formInfo,
    int               sizeWidth,
    int               sizeHeight,
    FieldSource       fieldSource,
    string            helpTag,
    Sessions.Session  session,
    object            property = null)
{
    InitializeComponent();                                       // ← unused scaffolding
    this.Name = formTitle;                                       // "popBorrower Information"
    this.Text = formTitle.Replace("pop", "");                    // "Borrower Information"
    this.Size = new Size(sizeWidth, sizeHeight);
    LoanScreen Screen = new LoanScreen(Session.DefaultInstance); // pulls live session
    this.Controls.Add(Screen);
    Screen.LoadForm(formInfo);                                   // ← THE rendering call
}
```

### 9.2 What each parameter means

| Param | Type | Used by | Notes |
|-------|------|---------|-------|
| `inputData` | `IHtmlInput` | (passed in but ignored in this body) | The repo passes `Session.LoanData`. If you mirror the original Encompass `QuickEntryPopupDialog` signature, keep it; if you write a minimal version, drop it. |
| `formTitle` | `string` | `this.Name`, `this.Text` | Name **must** include a prefix (`"pop"` in the reference) so the closer can find it. Title shown to the user is `formTitle.Replace("pop","")`. |
| `formInfo` | `InputFormInfo` | `Screen.LoadForm(formInfo)` | Result of `Session.FormManager.GetFormInfoByName(...)`. |
| `sizeWidth`/`sizeHeight` | `int` | `this.Size` | Window outer size. |
| `fieldSource` | `FieldSource` | (unused) | The repo passes `FieldSource.CurrentLoan`. Kept for signature compatibility with the Encompass dialog this forks. |
| `helpTag` | `string` | (unused) | Originally for `EMHelpLink.HelpTag`. Pass `""`. |
| `session` | `Sessions.Session` | (unused inside body) | Pass `Session.DefaultInstance`. |
| `property` | `object` | (unused) | Optional ctor overload kept for compatibility. |

### 9.3 Why `LoanScreen` needs `Session.DefaultInstance`

`LoanScreen`'s ctor stores the session reference and uses it later
when `LoadForm` resolves field bindings, fires rule contexts, and
attaches the rendered HTML controls to the field watch-list. Passing
a different session (e.g. one you build manually) will throw the
moment any rule or formula tries to read the live `LoanData`. Always
pass the live `Session.DefaultInstance` from
`EllieMae.EMLite.RemotingServices`.

### 9.4 Why the dialog ignores `InputForm.FormID`

`InputFormInfo` has both `FormID` (the unique identifier) and
`Name`. `LoadForm` works off the `InputFormInfo` object itself, so
you never extract or pass the `FormID` separately.

### 9.5 The dialog's `Name` prefix is load-bearing

`OpenInputForm` later does:

```csharp
FormWrapper.OpenForms.Where(x => x.Name.StartsWith("pop"))
```

…both to refresh open popups and to close them on `LoanClosing`. If
you change the prefix, change both call sites. If you remove the
prefix entirely you cannot tell *your* popups apart from Encompass's
own dialogs and you will close (or refresh) the wrong window.

---

## 10. The event-handler plugin — `OpenInputForm`, line-by-line

From `Standard Plugins/OpenInputForm.cs`:

```csharp
public class OpenInputForm : Plugin, ILoanOpened, IFieldChange
{
    private bool Open      = false;   // any popup currently shown?
    private bool HasFields = false;   // do CX.OPENFORM / CX.OPENFORM.SIZE exist on this loan?

    public override bool Authorized() =>
        PluginAccess.CheckAccess(nameof(OpenInputForm));

    public override void LoanOpened(object sender, EventArgs e)
    {
        HasFields =
            EncompassApplication.Session.Loans.FieldDescriptors.CustomFields
                .Cast<FieldDescriptor>()
                .Any(x => x.FieldID.Equals("CX.OPENFORM"))
         && EncompassApplication.Session.Loans.FieldDescriptors.CustomFields
                .Cast<FieldDescriptor>()
                .Any(x => x.FieldID.Equals("CX.OPENFORM.SIZE"));
    }

    public override void FieldChanged(object sender, FieldChangeEventArgs e)
    {
        if (!HasFields) return;

        if (e.FieldID.Equals("CX.OPENFORM") && !string.IsNullOrEmpty(e.NewValue))
        {
            InputFormInfo form = Session.FormManager.GetFormInfoByName(e.NewValue);
            if (form == null) return;

            string size = EncompassHelper.Val("CX.OPENFORM.SIZE").ToString();
            string[] setSize = size.Contains(',') ? size.Split(',') : new string[0];
            Size controlSize = setSize.Count() > 0
                ? new System.Drawing.Size(Convert.ToInt32(setSize[0]),
                                          Convert.ToInt32(setSize[1]))
                : new System.Drawing.Size(600, 600);

            QuickEntryPopupDialog2 q = new QuickEntryPopupDialog2(
                Session.LoanData,
                $"pop{form.Name}",
                form,
                controlSize.Width,
                controlSize.Height,
                EllieMae.Encompass.Forms.FieldSource.CurrentLoan,
                "",
                Session.DefaultInstance);
            q.Show();
            Open = true;

            EncompassHelper.SetBlank("CX.OPENFORM");
        }
        else if (Open)
        {
            foreach (Form f in FormWrapper.OpenForms
                                .Where(x => x.Name.StartsWith("pop"))
                                .Select(x => x))
            {
                LoanScreen screen = f?.Controls[0] as LoanScreen;
                if (screen == null) return;

                if (screen != null && !screen.ContainsFocus)
                    screen.RefreshLoanContents();
                else
                    Session.Application.GetService<ILoanEditor>()?.RefreshContents();
            }
        }
    }

    public override void LoanClosing(object sender, EventArgs e)
    {
        List<Form> close = FormWrapper.OpenForms
                            .Where(x => x.Name.StartsWith("pop"))
                            .ToList();
        foreach (Form f in close) f.Close();
    }
}
```

### 10.1 Field-existence guard

The `HasFields` check is the only safety net against running on
loans whose schema lacks the trigger fields. Without it,
`Fields["CX.OPENFORM"]` would throw on every field change. Run the
check **once per loan open** rather than every field change — the
schema is fixed for the duration of a single loan.

### 10.2 Why `SetBlank` is mandatory

After the popup opens, the trigger field is cleared. If you skip
this:

- Setting the same form name again (`CX.OPENFORM = "Form A"` →
  `"Form A"`) will **not** raise `FieldChange` because the value
  did not change.
- The popup will not reopen until the user sets a *different* value
  first.

`EncompassHelper.SetBlank` simply writes `string.Empty`:

```csharp
public static void SetBlank(string FieldID, string Index = null)
{
    if (string.IsNullOrEmpty(Index))
        EncompassApplication.CurrentLoan.Fields[FieldID].Value = string.Empty;
    else
        EncompassApplication.CurrentLoan.Fields[
            Loan.Fields.GetFieldAt(FieldID, Convert.ToInt32(Index)).ID].Value = string.Empty;
}
```

### 10.3 Why the second branch (`else if (Open)`) exists

Every other field change in the loan should refresh open popups so
they reflect calculation/rule cascades from the main form. But:

- If the popup itself has focus, **calling `RefreshLoanContents()`
  rebuilds the screen and the user loses the caret mid-edit.** The
  branch falls back to refreshing the main loan editor instead in
  that case, so the main UI sees the popup's typed-in values
  without disrupting the user.
- The `Open` flag prevents iterating `OpenForms` after every field
  change for the (common) case where no popup is open.

### 10.4 Why `LoanClosing` exists

The popup is bound to the prior `Loan` instance's data. If the user
opens a different loan and the popup is still showing, every read
or write through `LoanScreen` is dereferencing a disposed loan
object. Encompass throws `ObjectDisposedException`s and the popup
becomes a UI corpse. Always close on `LoanClosing`.

---

## 11. Minimum-viable rebuild (no framework, ~120 lines)

If you do not want `Plugin.cs` and friends, here is the entire
feature as four files. Drop them into a new project with the
references from §6.4 and the custom fields from §7.1.

### 11.1 `PluginEntry.cs`

```csharp
using EllieMae.Encompass.ComponentModel;

namespace MyPlugin
{
    [Plugin]
    public class PluginEntry
    {
        public PluginEntry() => new OpenInputForm().Wire();
    }
}
```

### 11.2 `OpenInputForm.cs`

```csharp
using EllieMae.EMLite.ClientServer;       // LoanScreen
using EllieMae.EMLite.InputEngine;        // InputFormInfo
using EllieMae.EMLite.RemotingServices;   // Session (static) + Sessions.Session
using EllieMae.Encompass.Automation;      // EncompassApplication
using EllieMae.Encompass.BusinessObjects.Loans;
using EllieMae.Encompass.Forms;           // FieldSource
using System;
using System.Drawing;
using System.Linq;
using System.Windows.Forms;

namespace MyPlugin
{
    public class OpenInputForm
    {
        private const string TRIGGER_FIELD = "CX.OPENFORM";
        private const string SIZE_FIELD    = "CX.OPENFORM.SIZE";
        private const string POPUP_PREFIX  = "pop";

        private bool hasFields;
        private bool open;

        public void Wire()
        {
            EncompassApplication.LoanOpened  += OnLoanOpened;
            EncompassApplication.LoanClosing += OnLoanClosing;
        }

        private void OnLoanOpened(object sender, EventArgs e)
        {
            Loan loan = EncompassApplication.CurrentLoan;
            if (loan == null) return;

            // Schema check once per loan open
            var custom = EncompassApplication.Session.Loans
                          .FieldDescriptors.CustomFields.Cast<FieldDescriptor>();
            hasFields = custom.Any(f => f.FieldID == TRIGGER_FIELD)
                     && custom.Any(f => f.FieldID == SIZE_FIELD);

            loan.FieldChange -= OnFieldChange;
            loan.FieldChange += OnFieldChange;
        }

        private void OnFieldChange(object sender, FieldChangeEventArgs e)
        {
            if (!hasFields) return;

            if (e.FieldID == TRIGGER_FIELD && !string.IsNullOrEmpty(e.NewValue))
            {
                OpenForm(e.NewValue);
                EncompassApplication.CurrentLoan.Fields[TRIGGER_FIELD].Value = string.Empty;
            }
            else if (open)
            {
                RefreshOpenPopups();
            }
        }

        private void OpenForm(string formName)
        {
            InputFormInfo form = Session.FormManager.GetFormInfoByName(formName);
            if (form == null) return;

            string raw = EncompassApplication.CurrentLoan
                          .Fields[SIZE_FIELD].Value?.ToString();
            Size size = ParseSize(raw);

            var host = new InputFormHost(form, size.Width, size.Height);
            host.Show();
            open = true;
        }

        private static Size ParseSize(string raw)
        {
            if (string.IsNullOrEmpty(raw) || !raw.Contains(","))
                return new Size(600, 600);
            var parts = raw.Split(',');
            return new Size(int.Parse(parts[0]), int.Parse(parts[1]));
        }

        private void RefreshOpenPopups()
        {
            foreach (Form f in Application.OpenForms.Cast<Form>()
                       .Where(x => x.Name != null && x.Name.StartsWith(POPUP_PREFIX))
                       .ToList())
            {
                LoanScreen screen = f.Controls.Count > 0
                    ? f.Controls[0] as LoanScreen : null;
                if (screen == null) continue;

                if (!screen.ContainsFocus)
                    screen.RefreshLoanContents();
                // else: skip — refresh would steal caret.
            }
        }

        private void OnLoanClosing(object sender, EventArgs e)
        {
            foreach (Form f in Application.OpenForms.Cast<Form>()
                       .Where(x => x.Name != null && x.Name.StartsWith(POPUP_PREFIX))
                       .ToList())
            {
                f.Close();
            }
        }
    }
}
```

### 11.3 `InputFormHost.cs`

```csharp
using EllieMae.EMLite.ClientServer;       // LoanScreen
using EllieMae.EMLite.InputEngine;        // InputFormInfo
using EllieMae.EMLite.RemotingServices;   // Session.DefaultInstance
using System.Drawing;
using System.Windows.Forms;

namespace MyPlugin
{
    public class InputFormHost : Form
    {
        public InputFormHost(InputFormInfo form, int width, int height)
        {
            this.Name             = "pop" + form.Name;   // matches POPUP_PREFIX
            this.Text             = form.Name;
            this.ClientSize       = new Size(width, height);
            this.StartPosition    = FormStartPosition.CenterScreen;
            this.ShowInTaskbar    = false;
            this.MinimizeBox      = false;
            this.FormBorderStyle  = FormBorderStyle.Sizable;

            var screen = new LoanScreen(Session.DefaultInstance);
            screen.Dock = DockStyle.Fill;
            this.Controls.Add(screen);
            screen.LoadForm(form);
        }
    }
}
```

That is the entire feature: a 4-file plugin that hosts any IFB form
in a free-floating window driven by a field change.

---

## 12. Permissions / `PluginAccess` (optional)

The reference repo gates every plugin behind
`PluginAccess.CheckAccess("OpenInputForm")`, which reads
`CommunitySettings.json` from an Encompass Custom Data Object (CDO).
You can omit this entirely if your plugin should always run.

### 12.1 What `CheckAccess` does

```csharp
public static bool CheckAccess(string pluginName,
                               bool menu = false,
                               bool loan = false)
{
    if (EncompassHelper.IsTest() || CDOHelper.CDO.CommunitySettings.SuperAdminRun)
        return true;

    PluginAccessRight right = Rights
        .FirstOrDefault(x => x.PluginName.Equals(pluginName));
    if (right == null) return false;

    bool allowed = loan ? false : right.AllAccess;
    if (!allowed && right.Personas != null)
        allowed = EncompassHelper.ContainsPersona(right.Personas);
    if (!allowed && right.UserIDs != null)
        allowed = right.UserIDs.Contains(EncompassHelper.User.ID);
    return allowed;
}
```

`Rights` is built from `CDO.CommunitySettings.Permission["AllAccess"]`
which is a list of plugin-name strings.

### 12.2 `CommunitySettings.json` shape (just the permission slice)

```json
{
  "CommunitySettings": {
    "TestServer": "TEBE11177289",
    "SuperAdminRun": false,
    "Permission": {
      "AllAccess": [
        "OpenInputForm"
      ]
    }
  }
}
```

### 12.3 How the file is loaded

`CDOHelper.CDO` lazy-loads via:

```csharp
File = JsonConvert.DeserializeObject<CDO>(
    Encoding.UTF8.GetString(
        EncompassApplication.Session.DataExchange
            .GetCustomDataObject("CommunitySettings.json").Data));
```

So the file lives in Encompass's CDO store, not on disk. Upload it
once via the same `DataExchange.SaveCustomDataObject` (see
`CDOHelper.UploadCDO`).

For a fresh plugin without CDO infrastructure, just hard-code
`Authorized() => true;` and skip every line in this section.

---

## 13. Build, sign, deploy

1. **Build.** Release configuration, `bin\Release\MyPlugin.dll`.
2. **Sign?** Encompass does not require strong-naming. If your
   org's policy requires it, sign the assembly normally.
3. **Deploy.** Two options:
   - **Per-user.** Copy `MyPlugin.dll` to
     `%AppData%\Encompass\Plugins\`. Encompass picks it up the next
     time it starts.
   - **Centralized (recommended).** Upload via **Encompass Admin
     Tools → Custom Code → Upload**. Encompass deploys it to every
     user on next login via ClickOnce. This is the production path.
4. **Verify load.** **Help → About → Plug-ins.** Your DLL must
   appear with a green status. If it's red, double-click it to see
   the load exception (almost always a missing/duplicated reference
   from §6.4).
5. **Restart Encompass.** Plugin discovery happens during startup;
   you can't hot-reload.

---

## 14. Verification checklist

In order, on a test server only:

- [ ] DLL appears under **Help → About → Plug-ins** with green status.
- [ ] Open a loan; no exceptions in `ApplicationLog.xml`.
- [ ] In a debugger or via an IFB button, set
      `Loan.Fields("CX.OPENFORM") = "Borrower Information"`.
- [ ] A floating window titled `Borrower Information` appears within
      ~1 second at 600×600.
- [ ] The window shows the IFB form with the live loan's data
      pre-populated.
- [ ] Edit a field in the popup. Switch to the main loan tab — the
      change is visible.
- [ ] Edit a field in the main loan tab. Without giving the popup
      focus, watch the popup refresh.
- [ ] Now set `CX.OPENFORM.SIZE = "1000,800"` first, then
      `CX.OPENFORM = "..."` — popup opens at 1000×800.
- [ ] Open a second form; both popups coexist.
- [ ] Close the loan (open a different one). Both popups close.
- [ ] Set the same form name twice in a row — popup re-opens both
      times (proves `SetBlank` is firing).

---

## 15. Gotchas — every one of them

### 15.1 Assembly resolution

- **`Copy Local = False` on every Encompass DLL.** Mandatory.
  Symptom of getting this wrong: `LoanScreen` silently fails to
  bind the form, or you get an `InvalidCastException` on a type
  that obviously is the right type.
- **`Newtonsoft.Json` binding redirect.** Encompass ships its own
  v12 in the SmartClient cache. If you reference a different
  version, add the binding redirect from §6.6.
- **Don't reference `mscorlib` or `System` manually** — the
  framework reference brings them.

### 15.2 Timing

- **Don't instantiate `LoanScreen` before user login.**
  `Session.DefaultInstance` is null pre-login. Gate construction
  behind `LoanOpened` or `EncompassApplication.Login`.
- **Don't construct `LoanScreen` from a background thread.**
  WinForms thread-affinity rules apply. `FieldChange` fires on the
  UI thread inside Encompass; trust it. If you receive a trigger
  from a worker thread (HTTP callback, polled queue, etc.) marshal
  back: `mainForm.Invoke(...)`.

### 15.3 Form-name resolution

- **`GetFormInfoByName` matches internal form name, not display
  title.** Check via **IFB → form Properties → Form Name**.
- **Names are case-sensitive in some Encompass versions.** If the
  lookup returns `null` unexpectedly, try the exact casing shown in
  IFB.
- **Standard (built-in) forms are still findable by name** — they
  just live in `InputFormType.Standard` instead of `Custom`. The
  `GetFormInfoByName` lookup is type-agnostic.

### 15.4 Field interactions

- **Always blank the trigger field after opening** or the same
  value setting won't refire.
- **`HasFields` check uses CustomFields, not StandardFields.**
  If you ever rename the trigger to a standard field, change the
  schema-check source.
- **Don't read `Fields["CX.OPENFORM"]` if `HasFields` is false** —
  the indexer throws.

### 15.5 Popup lifecycle

- **Prefix `Form.Name` with something unique** (`"pop"` in the
  reference). It's how the `LoanClosing` cleanup and the
  field-change refresh both find your popups. Without a prefix
  you cannot distinguish your forms from any other dialog
  Encompass opens.
- **Close on `LoanClosing`.** Encompass does not do this for you.
  Failing this leaks bindings and `LoanScreen` errors fire on
  every subsequent field write.
- **Skip `RefreshLoanContents` when `screen.ContainsFocus`.**
  Refreshing the focused screen drops the user's caret and
  collapses dropdowns mid-selection. The reference falls back to
  `ILoanEditor.RefreshContents()` on the main editor in that case.

### 15.6 `Session` namespace ambiguity

- `EllieMae.EMLite.RemotingServices.Session` (static) ≠
  `EllieMae.EMLite.RemotingServices.Sessions.Session` (instance) ≠
  `EllieMae.Encompass.Automation.EncompassApplication.Session`
  (automation wrapper). All three are referenced in the same file
  in this repo. Read the using directives carefully when porting.

### 15.7 Threading on `FormWrapper.OpenForms`

The reference uses a custom `FormWrapper.OpenForms` set populated
by a 1-second `Timer` (`Non Native Modifications/FormWrapper.cs`).
If you skip `FormWrapper` and iterate `Application.OpenForms`
directly (as the minimal rebuild does), you avoid the timer
entirely. The trade-off: `Application.OpenForms` is a snapshot at
call time, which is fine for our use case — we only iterate it
inside `FieldChange` / `LoanClosing`, both on the UI thread.

### 15.8 `LoanScreen` quirks

- `LoanScreen.LoadForm` is synchronous and blocks until layout
  completes.
- It does not paint until the host `Form` is shown — `Show()`
  triggers the first paint.
- It re-binds the entire field set on every `RefreshLoanContents`.
  If you call this in a tight loop the UI freezes. The reference
  guards it behind a per-`FieldChange` call, which Encompass
  naturally rate-limits.

### 15.9 Multiple popups for the same form

The reference does not de-dupe. Setting `CX.OPENFORM` to the same
value twice opens the form twice. If you want at-most-one-per-name
behavior, check `Application.OpenForms` for `Name == "pop" + form.Name`
before constructing a new host.

### 15.10 Plugin discovery

- `[Plugin]` class must have a **public, parameterless** ctor.
- The `[Plugin]` class is instantiated **once per client process**.
  Don't do per-loan work in its ctor.

---

## 16. Extending the pattern

### 16.1 Open by `FormID` instead of name

Form names can collide; `FormID` is unique. Replace
`GetFormInfoByName(name)` with `GetFormInfoById(id)` (also on
`Session.FormManager`) and feed the ID through `CX.OPENFORM`
instead.

### 16.2 Open without a trigger field

Skip `OpenInputForm` entirely. Anywhere in your code (a menu item,
a button, a `LoanOpened` hook), construct `InputFormHost` directly:

```csharp
var info = Session.FormManager.GetFormInfoByName("My Form");
new InputFormHost(info, 800, 600).Show();
```

### 16.3 Embed the form inside another tab/control

`LoanScreen` is a normal `Control`. Drop it into any container:

```csharp
var screen = new LoanScreen(Session.DefaultInstance);
screen.Dock = DockStyle.Fill;
myUserControl.Controls.Add(screen);
screen.LoadForm(Session.FormManager.GetFormInfoByName("My Form"));
```

That is how a custom side-panel or floating ribbon could host an
IFB form without a `Form` at all.

### 16.4 Read-only mode

There is no `ReadOnly` toggle on `LoanScreen`. To make the embedded
form read-only, gate via the user's persona (Encompass's normal
form-permission system on the IFB form itself).

### 16.5 Multi-instance fields

The reference's `EncompassHelper.Val` returns `string.Empty` for
multi-instance fields:

```csharp
if (!EncompassApplication.Session.Loans.FieldDescriptors[FieldID].MultiInstance)
    return EncompassApplication.CurrentLoan.Fields[FieldID].Value?.ToString() ?? "";
else
    return string.Empty;
```

If you want to drive form-open via a multi-instance field
(borrower-pair specific, for example), pass an index using
`Loan.Fields.GetFieldAt(FieldID, index).ID` instead.

---

## 17. Quick reference card

```
Trigger     → custom field CX.OPENFORM := "<IFB form name>"
Lookup      → Session.FormManager.GetFormInfoByName(name)  // EllieMae.EMLite.RemotingServices
Render      → new LoanScreen(Session.DefaultInstance) + LoadForm(InputFormInfo)
Host        → System.Windows.Forms.Form with Name="pop<name>"
Refresh     → screen.RefreshLoanContents()  (skip when screen.ContainsFocus)
Cleanup     → on EncompassApplication.LoanClosing, close all forms with Name.StartsWith("pop")
Sizing      → custom field CX.OPENFORM.SIZE := "<w>,<h>" (default 600,600)
Re-trigger  → blank CX.OPENFORM after opening, otherwise same-value setter won't fire
```

That is the entire feature, rebuildable from first principles with
no external knowledge beyond what is in this document.
