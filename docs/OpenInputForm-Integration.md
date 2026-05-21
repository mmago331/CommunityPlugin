# Hosting an Input Form Builder Form Inside Encompass

This document explains exactly how the **CommunityPlugin** loads an Input
Form designed in Encompass's **Input Form Builder (IFB)** and renders it
inside Encompass at runtime in its own floating window, and how to
rebuild the feature from scratch inside any other Encompass SDK plugin.

The end-user experience: a field on the loan (e.g. `CX.OPENFORM`) is set
to the name of an IFB form. The plugin sees the field change, opens a
new window, and the form renders in that window using the same
`LoanScreen` control Encompass itself uses on its native tabs — so all
fields are live-bound to the current loan, business rules fire, and
formulas evaluate exactly as they do on a built-in tab.

---

## 1. How it works in this repo

There are exactly two source files that implement the feature:

| File | Role |
|------|------|
| `CommunityPlugin/Standard Plugins/OpenInputForm.cs` | Event-handler plugin. Watches the field change and triggers the popup. |
| `CommunityPlugin/Objects/QuickEntryPopupDialog2.cs` | The host `Form` containing the `LoanScreen` that renders the IFB form. |

Two supporting custom fields are required (created in IFB, see §3):

| Field ID | Purpose |
|----------|---------|
| `CX.OPENFORM` | Set this to the name of the IFB form to open. |
| `CX.OPENFORM.SIZE` | Optional. Comma-separated `width,height` (e.g. `800,600`). |

### 1.1 The flow, step by step

1. Encompass loads the plugin assembly (it is decorated with
   `[Plugin]`) and `PluginEntry` calls `Plugins.Start()`, which
   instantiates every `Plugin` subclass and calls `Run()`.
2. `Plugin.Run()` (in `Objects/Plugin.cs`) inspects the interfaces the
   subclass implements and wires up handlers. `OpenInputForm`
   implements `ILoanOpened` and `IFieldChange`, so it gets
   `EncompassApplication.LoanOpened` and `Loan.FieldChange`
   subscriptions.
3. `LoanOpened` runs once per loan. It checks the loan's
   `FieldDescriptors.CustomFields` to confirm `CX.OPENFORM` and
   `CX.OPENFORM.SIZE` exist. If not, the plugin no-ops for that loan.
4. `FieldChanged` fires on every loan field change. When
   `e.FieldID == "CX.OPENFORM"` and the new value is non-empty:
   - `Session.FormManager.GetFormInfoByName(e.NewValue)` resolves the
     IFB form by name and returns an `InputFormInfo`.
   - The optional `CX.OPENFORM.SIZE` is parsed into a `Size`
     (default `600x600`).
   - A new `QuickEntryPopupDialog2` is constructed with the loan data,
     the resolved `InputFormInfo`, the size, the field source
     (`FieldSource.CurrentLoan`) and the current Encompass session.
   - `.Show()` displays it as a modeless window.
   - `CX.OPENFORM` is blanked so the next assignment (even the same
     value) triggers another open.
5. Inside `QuickEntryPopupDialog2`'s constructor:
   - `LoanScreen Screen = new LoanScreen(Session.DefaultInstance);`
   - `this.Controls.Add(Screen);`
   - `Screen.LoadForm(formInfo);` — this is what makes the IFB form
     actually render and bind to the current loan.
6. On subsequent field changes, if any popup is still open, its
   `LoanScreen` is refreshed (or the main loan editor is refreshed)
   so the popup stays in sync with the loan.
7. On `LoanClosing`, every open popup whose `Name` starts with `pop`
   is closed.

### 1.2 The minimal API surface this depends on

These types come from internal Encompass assemblies (not the public
`EncompassObjects.dll` API). You will reference them by adding the
DLLs listed in §2:

| Type | Assembly | Namespace |
|------|----------|-----------|
| `Session` | `ClientSession.dll` | `EllieMae.EMLite.RemotingServices` |
| `Session.DefaultInstance` (static) | `ClientSession.dll` | `EllieMae.EMLite.RemotingServices` |
| `Session.FormManager.GetFormInfoByName(name)` | `ClientSession.dll` | `EllieMae.EMLite.RemotingServices` |
| `InputFormInfo` | `EMInput.dll` | `EllieMae.EMLite.InputEngine` |
| `LoanScreen` | `Client.dll` (UI) | `EllieMae.EMLite.ClientServer` |
| `IHtmlInput` | `ClientServer.dll` | `EllieMae.EMLite.ClientServer` |
| `FieldSource` | `EncompassObjects.dll` | `EllieMae.Encompass.Forms` |
| `EncompassApplication`, `Session` (automation wrapper) | `EncompassAutomation.dll` | `EllieMae.Encompass.Automation` |
| `FieldChangeEventArgs`, `Loan`, `FieldDescriptor` | `EncompassObjects.dll` | `EllieMae.Encompass.BusinessObjects.Loans` |
| `[Plugin]` attribute | `EncompassObjects.dll` | `EllieMae.Encompass.ComponentModel` |

Encompass's "Quick Entry" popup (the built-in `QuickEntryPopupDialog`)
is the model `QuickEntryPopupDialog2` is patterned after; it lives in
`Client.dll` under `EllieMae.EMLite.ClientServer`. Reflecting on it is
a good cross-check for the constructor signature.

---

## 2. Project setup for a new plugin

### 2.1 Create the project

1. Create a **.NET Framework 4.7.2** (or whatever matches your
   Encompass version) **Class Library** project.
2. Set the assembly to be signed if your environment requires it,
   and ensure the target output is a single DLL deployable to
   Encompass's plugins folder.

### 2.2 Reference Encompass assemblies

All Encompass DLLs live in the SmartClient cache, typically at:

```
%LocalAppData%\Apps\2.0\...\SmartClientCache\Apps\UAC\Ellie Mae\<hash>\Encompass360\
```

(Open Encompass once on the dev box to populate this folder, then
copy the path from any working plugin's `HintPath` — this repo's
`CommunityPlugin.csproj` shows the exact form.)

Required references for this feature:

- `EncompassObjects.dll` (public SDK — `EncompassApplication`,
  `Loan`, `FieldDescriptor`, `[Plugin]`, `FieldSource`)
- `EncompassAutomation.dll`
- `ClientSession.dll` (gives you `Sessions.Session` and
  `FormManager`)
- `ClientServer.dll` (gives you `IHtmlInput`)
- `Client.dll` (gives you `LoanScreen`; this is the same control
  Encompass uses internally)
- `EMInput.dll` (gives you `InputFormInfo`)
- `System.Windows.Forms`, `System.Drawing`

Set `Copy Local = False` on every Encompass DLL — Encompass loads
them itself at runtime; copying them into the plugin output causes
type-identity conflicts.

### 2.3 Mark the assembly as a plugin

Add a single `PluginEntry` class so Encompass discovers and loads
the plugin:

```csharp
using EllieMae.Encompass.ComponentModel;

namespace MyPlugin
{
    [Plugin]
    public class PluginEntry
    {
        public PluginEntry()
        {
            // bootstrap your plugins here
            Bootstrapper.Start();
        }
    }
}
```

(In this repo: `CommunityPlugin/PluginEntry.cs` and
`CommunityPlugin/Objects/Plugins.cs`.)

---

## 3. Create the two custom fields in Input Form Builder

In Encompass go to **Settings → Loan Setup → Custom Fields**, then
create:

| Field ID | Type | Description |
|----------|------|-------------|
| `CX.OPENFORM` | String | Holds the name of the IFB form to open. |
| `CX.OPENFORM.SIZE` | String | Optional. `width,height` (e.g. `800,600`). |

You do **not** need to place these fields on any form for the
feature to work — the plugin just reads/writes them through the
field collection. They only need to exist.

You can drive these fields from anything: another IFB form, an
advanced-code button, a business rule, or another plugin.

---

## 4. The host dialog (rendering control)

The host is a plain `System.Windows.Forms.Form` that contains a
single `LoanScreen` control. `LoanScreen` is Encompass's own form
host — it's the same control that backs every IFB tab inside the
loan workspace, so once it has loaded the form everything (data
binding, formulas, business rules, eFolder hyperlinks, advanced-code
buttons) just works.

Minimum viable host:

```csharp
using EllieMae.EMLite.ClientServer;       // LoanScreen, IHtmlInput
using EllieMae.EMLite.InputEngine;        // InputFormInfo
using EllieMae.EMLite.RemotingServices;   // Sessions.Session
using EllieMae.Encompass.Forms;           // FieldSource
using System.Drawing;
using System.Windows.Forms;

namespace MyPlugin.UI
{
    public class InputFormHost : Form
    {
        public InputFormHost(
            IHtmlInput loanData,
            string title,
            InputFormInfo formInfo,
            int width,
            int height,
            Sessions.Session session)
        {
            // Name MUST start with a known prefix so you can find/close
            // all of these later (this repo uses "pop").
            this.Name = "pop" + formInfo.Name;
            this.Text = title;
            this.ClientSize = new Size(width, height);
            this.StartPosition = FormStartPosition.CenterScreen;
            this.ShowInTaskbar = false;
            this.MinimizeBox = false;

            var screen = new LoanScreen(Session.DefaultInstance);
            screen.Dock = DockStyle.Fill;
            this.Controls.Add(screen);
            screen.LoadForm(formInfo);
        }
    }
}
```

Notes about `LoanScreen`:

- **You must pass an active `Sessions.Session`.** The static
  `Session.DefaultInstance` works inside a logged-in Encompass
  client. If you instantiate `LoanScreen` before login it will
  throw — gate the open behind `EncompassApplication.LoanOpened` or
  a `Login` handler.
- `LoadForm(InputFormInfo)` is what actually paints the form and
  binds it to `EncompassApplication.CurrentLoan`.
- `RefreshLoanContents()` causes the LoanScreen to re-pull current
  loan values — call it after the user changes fields elsewhere so
  the popup doesn't go stale.

---

## 5. The event-handler plugin

This class is the trigger. It subscribes to the loan field-change
event and opens the host when the watched field changes.

```csharp
using EllieMae.EMLite.ClientServer;       // LoanScreen
using EllieMae.EMLite.InputEngine;        // InputFormInfo
using EllieMae.EMLite.RemotingServices;   // Sessions.Session
using EllieMae.Encompass.Automation;      // EncompassApplication
using EllieMae.Encompass.BusinessObjects.Loans;
using EllieMae.Encompass.Forms;           // FieldSource
using System;
using System.Drawing;
using System.Linq;
using System.Windows.Forms;
using MyPlugin.UI;

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
            EncompassApplication.LoanOpened  += LoanOpened;
            EncompassApplication.LoanClosing += LoanClosing;
        }

        private void LoanOpened(object sender, EventArgs e)
        {
            var loan = EncompassApplication.CurrentLoan;
            if (loan == null) return;

            // Confirm the trigger fields actually exist on the schema.
            var custom = EncompassApplication.Session.Loans
                .FieldDescriptors.CustomFields.Cast<FieldDescriptor>();
            hasFields =
                custom.Any(f => f.FieldID == TRIGGER_FIELD) &&
                custom.Any(f => f.FieldID == SIZE_FIELD);

            // Subscribe per-loan; unsubscribe will happen automatically
            // when the loan object is released.
            loan.FieldChange -= FieldChanged;
            loan.FieldChange += FieldChanged;
        }

        private void FieldChanged(object sender, FieldChangeEventArgs e)
        {
            if (!hasFields) return;

            if (e.FieldID == TRIGGER_FIELD && !string.IsNullOrEmpty(e.NewValue))
            {
                OpenForm(e.NewValue);
                // Clear so the same form name can be opened again later.
                EncompassApplication.CurrentLoan.Fields[TRIGGER_FIELD].Value = string.Empty;
            }
            else if (open)
            {
                RefreshOpenPopups();
            }
        }

        private void OpenForm(string formName)
        {
            // Look up the IFB form by name.
            InputFormInfo form =
                Session.DefaultInstance.FormManager.GetFormInfoByName(formName);
            if (form == null) return;

            var size = ParseSize(
                EncompassApplication.CurrentLoan.Fields[SIZE_FIELD].Value?.ToString());

            var host = new InputFormHost(
                loanData : Session.DefaultInstance.LoanData,
                title    : form.Name,
                formInfo : form,
                width    : size.Width,
                height   : size.Height,
                session  : Session.DefaultInstance);

            host.Show();
            open = true;
        }

        private static Size ParseSize(string raw)
        {
            if (string.IsNullOrEmpty(raw) || !raw.Contains(","))
                return new Size(600, 600);

            var parts = raw.Split(',');
            return new Size(
                Convert.ToInt32(parts[0]),
                Convert.ToInt32(parts[1]));
        }

        private void RefreshOpenPopups()
        {
            foreach (Form f in Application.OpenForms
                                .Cast<Form>()
                                .Where(x => x.Name != null
                                         && x.Name.StartsWith(POPUP_PREFIX))
                                .ToList())
            {
                var screen = f.Controls.Count > 0
                    ? f.Controls[0] as LoanScreen
                    : null;
                if (screen == null) continue;

                if (!screen.ContainsFocus)
                    screen.RefreshLoanContents();
            }
        }

        private void LoanClosing(object sender, EventArgs e)
        {
            // Close every popup we opened.
            foreach (Form f in Application.OpenForms
                                .Cast<Form>()
                                .Where(x => x.Name != null
                                         && x.Name.StartsWith(POPUP_PREFIX))
                                .ToList())
            {
                f.Close();
            }
        }
    }
}
```

Bootstrap it from `PluginEntry`:

```csharp
public static class Bootstrapper
{
    public static void Start()
    {
        new OpenInputForm().Wire();
    }
}
```

That is the entire feature. The above 4 files (`PluginEntry.cs`,
`Bootstrapper.cs`, `OpenInputForm.cs`, `InputFormHost.cs`) plus the
two custom fields in IFB are all that is required.

---

## 6. How the CommunityPlugin version differs

The reference implementation in this repo is split across more
files because of its plugin framework:

- `Objects/Plugin.cs` is an abstract base that wires up *every*
  Encompass event (`LoanOpened`, `LoanClosing`, `FieldChange`,
  `LogEntry…`, `Milestone…`, etc.) based on which marker interfaces
  the subclass implements. `OpenInputForm` therefore just
  declares `: Plugin, ILoanOpened, IFieldChange` and overrides
  `LoanOpened` and `FieldChanged`.
- `Objects/Plugins.cs` uses reflection to find every `Plugin`
  subclass in the assembly and call `.Run()` on each.
- `Objects/PluginAccess.cs` reads permissions from
  `CommunitySettings.json` so each plugin can be enabled per persona.
- `Non Native Modifications/FormWrapper.cs` polls
  `Application.OpenForms` on a 1-second timer and raises a
  `FormOpened` event whenever a new `Form` appears. It also tracks
  the set of open forms — that's what `FormWrapper.OpenForms` returns
  in `OpenInputForm.LoanClosing`.

If you want a one-feature plugin, the minimal version in §5 is
enough. If you want to grow it into a multi-plugin framework, copy
`Plugin.cs`, `Plugins.cs`, `PluginEntry.cs` and the
marker-interfaces folder (`Objects/Interface/`) as a starting point.

---

## 7. Picking the right trigger

Field-change triggering (`CX.OPENFORM`) is the simplest and most
flexible — anything can write to a field. Other options the same
pattern supports:

| Trigger | How |
|---------|-----|
| Button on an IFB form | Advanced-code button that sets `CX.OPENFORM = "MyForm"`. |
| Business rule | Field-trigger rule that sets `CX.OPENFORM` based on conditions. |
| Side menu link | Call `OpenForm("MyForm")` directly from a menu click. See `Non Native Modifications/SideMenu/Controls/CustomLinkLabel.cs` in this repo for the pattern. |
| Built-in Encompass macro | `Macro.Popup(formName, title, w, h)` from `EllieMae.Encompass.Automation.Macro` does its own popup but you lose control over the host window. |

The plugin pattern (custom host + `LoanScreen.LoadForm`) gives you
control of size, position, lifecycle, prefix-based discovery and
auto-refresh that `Macro.Popup` does not.

---

## 8. Gotchas / things that will burn you

1. **Form name vs. form title.** `GetFormInfoByName` matches the
   form's internal **name** (the value you see in IFB's form
   properties), not its display title. They are often different.
2. **`LoanScreen` before login.** Constructing `LoanScreen` requires
   `Session.DefaultInstance` to be initialized. Gate every open
   call behind `LoanOpened` or `EncompassApplication.Login`.
3. **Re-opening the same form.** After opening, the trigger field
   is blanked. If you do not blank it, setting the same value
   again will not raise `FieldChange` and the popup won't reopen.
4. **Stale popup content.** When the user edits fields in the main
   loan workspace, the popup's `LoanScreen` does not auto-refresh.
   Subscribe to `FieldChange` and call `RefreshLoanContents()` (as
   shown above), but skip it when the popup has focus — refreshing
   the active screen mid-edit drops the user's caret.
5. **Closing popups on loan close.** Encompass will not close your
   popups when the user switches loans or closes the loan. You must
   do it yourself in `LoanClosing`, otherwise the popup is bound to
   the *previous* loan's data and will throw on the next field
   write.
6. **`Copy Local = False`.** Every Encompass DLL reference must have
   `Copy Local = False`. If Encompass loads your plugin and finds
   a second copy of `EMInput.dll` in the plugin folder, `LoanScreen`
   will silently fail to bind because the `InputFormInfo` it sees
   is a different CLR type.
7. **Threading.** `FieldChange` fires on the UI thread inside the
   Encompass client — you can safely manipulate WinForms from it.
   Do **not** open popups from a background thread; marshal back
   with `Form.Invoke` if you ever trigger this from a worker.
8. **Multiple popups.** Use a unique prefix (`"pop"` in this repo)
   in `Form.Name` so you can find and close all of them later
   without affecting unrelated Encompass dialogs.

---

## 9. Verifying it works

After installing the built DLL into Encompass's plugin folder and
restarting Encompass:

1. Open any loan.
2. In the Encompass field-lookup or any IFB form that exposes
   `CX.OPENFORM`, set it to the exact internal name of an existing
   IFB form (e.g. `"Borrower Information"`).
3. The popup should open within a second, sized either from
   `CX.OPENFORM.SIZE` or the 600×600 default.
4. Edits inside the popup should write through to the live loan and
   be visible on the main tab immediately when you switch back.
5. Closing the loan should close the popup.

If nothing happens, check:

- `Session.DefaultInstance.FormManager.GetFormInfoByName(name)`
  returns non-null (use a debug log line) — most failures are a
  form-name typo.
- The custom fields actually exist in the loan schema.
- The plugin is loaded by Encompass (look in **Help → About → Plug-ins**).
