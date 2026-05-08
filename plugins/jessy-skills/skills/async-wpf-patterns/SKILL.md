---
name: async-wpf-patterns
description: "MUST use when fixing WPF UI freeze or 'Not Responding' on title bar, when user reports
             hanging screen on button click, when a blocking call (hardware, PKCS#11, SOAP, file I/O,
             HTTP) runs on the UI thread, when user says 'screen hang', 'not responding', 'UI freeze',
             'async', 'Task.Run', or when adding loading indicator / disable button during processing
             in WPF / Caliburn.Micro apps."
---

# Async WPF Patterns — UI Thread Safety & Responsive Design
*Fix UI freezes, offload blocking calls, and add loading feedback in WPF + Caliburn.Micro*

## Activation

When this skill activates:
`⚡ Async WPF Patterns skill loaded — UI thread safety and responsive design ready.`

Then apply the knowledge below to assist with the task.

---

## The Core Problem

WPF has one UI thread (STA). Any blocking call on this thread freezes the entire app:
- Title bar shows **(Not Responding)**
- Window goes white / unresponsive
- User cannot interact until the call returns

**Common blocking culprits:**
- PKCS#11 token detection (`new Pkcs11(...)`, `GetSlotList()`, `GetSlotInfo()`)
- SOAP/HTTP calls (CertVerifier, web services)
- File I/O (reading large files, XML parsing)
- Database queries
- Hardware enumeration (USB, smart card readers)

---

## Pattern 1 — Offload to Background Thread

```csharp
// BEFORE (blocks UI thread)
public void LoadTokenType()
{
    StatusInfo si = DetectToken.Token();  // slow PKCS#11 call — freezes UI
    // handle result...
}

// AFTER (UI stays responsive)
public async Task LoadTokenType()
{
    StatusInfo si = await Task.Run(() => DetectToken.Token());
    // handle result — automatically back on UI thread after await
}
```

**Key rules:**
- Change `void` → `async Task` (linter / Caliburn.Micro prefers `async Task` over `async void`)
- Add `using System.Threading.Tasks;`
- After `await`, WPF's `SynchronizationContext` automatically resumes on the UI thread
- No `Dispatcher.Invoke` needed for UI updates after the `await`

---

## Pattern 2 — Disable Buttons + Loading Cursor

Use `try/finally` to guarantee cleanup even if an exception is thrown.

```csharp
private bool _isProcessing;

// Caliburn.Micro convention: CanXxx property auto-controls IsEnabled on button named Xxx
public bool CanLoadTokenType => !_isProcessing;
public bool CanCancel => !_isProcessing;

public async Task LoadTokenType()
{
    _isProcessing = true;
    NotifyOfPropertyChange(() => CanLoadTokenType);
    NotifyOfPropertyChange(() => CanCancel);
    Mouse.OverrideCursor = Cursors.Wait;

    try
    {
        StatusInfo si = await Task.Run(() => DetectToken.Token());
        // handle result...
    }
    finally
    {
        _isProcessing = false;
        Mouse.OverrideCursor = null;        // null = restore default cursor
        NotifyOfPropertyChange(() => CanLoadTokenType);
        NotifyOfPropertyChange(() => CanCancel);
    }
}
```

**Required usings:**
```csharp
using System.Threading.Tasks;
using System.Windows.Input;   // Mouse, Cursors
```

---

## Pattern 3 — Caliburn.Micro CanXxx Convention

Caliburn.Micro auto-wires button `IsEnabled` to `Can<MethodName>` — **no XAML change needed**.

| XAML button name | CM looks for | Controls |
|-----------------|--------------|---------|
| `x:Name="LoadTokenType"` | `CanLoadTokenType` property | `IsEnabled` |
| `Name="Cancel"` | `CanCancel` property | `IsEnabled` |

```csharp
// Returning false disables the button automatically
public bool CanLoadTokenType => !_isProcessing;
public bool CanCancel => !_isProcessing;
```

After changing the value, notify CM to re-evaluate:
```csharp
NotifyOfPropertyChange(() => CanLoadTokenType);
```

---

## Pattern 4 — Background Thread Calling WPF (Reverse Problem)

When a **background thread** (e.g. WebSocket service) calls WPF UI operations, it throws:
```
InvalidOperationException: The calling thread must be STA
```

**Fix:** Either remove the WPF call from the background thread, or dispatch it:
```csharp
// Option A — remove WPF call (preferred if not needed)
// e.g. remove StackedCursorOverride from DetectToken.Token()

// Option B — dispatch to UI thread
Application.Current.Dispatcher.Invoke(() =>
{
    Mouse.OverrideCursor = Cursors.Wait;
});
```

**Real case from MyTrustID:** `DetectToken.Token()` had `StackedCursorOverride` (a WPF call).
When called from WebSocket background thread → STA exception. Fix: removed the WPF call entirely,
since cursor management belongs in the ViewModel, not the helper.

---

## Diagnosis Checklist

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| "Not Responding" on button click | Blocking call on UI thread | `await Task.Run(...)` |
| `InvalidOperationException: must be STA` | WPF call from background thread | Remove WPF call or use `Dispatcher.Invoke` |
| Button stays active during processing | No `CanXxx` property or not notified | Add `Can<Name>` + `NotifyOfPropertyChange` |
| Cursor not restored after error | Missing `finally` block | Wrap in `try/finally` |
| After `await`, `Dispatcher.Invoke` still needed | Task.Run used without `await` (fire-and-forget) | Always `await` the Task |

---

## What to Offload vs What to Keep on UI Thread

| Keep on UI thread | Offload with Task.Run |
|-------------------|-----------------------|
| `conductor.ActivateItem(...)` | PKCS#11 / token detection |
| `MessageBox.Show(...)` | SOAP/HTTP calls |
| `Mouse.OverrideCursor = ...` | File I/O |
| `NotifyOfPropertyChange(...)` | Database queries |
| Setting bound properties | Hardware enumeration |

---

## Real-World Example (MyTrustID Desktop)

**File:** `SelectStorageViewModel.cs`

**Problem:** Clicking "Continue" (Token selected) called `DetectToken.Token()` synchronously.
This loaded PKCS#11 DLLs and queried USB slots — 2–5 seconds of hardware I/O on the UI thread
→ "Not Responding" on title bar.

**Fix applied:**
1. Changed `void LoadTokenType()` → `async Task LoadTokenType()`
2. `DetectToken.Token()` → `await Task.Run(() => DetectToken.Token())`
3. Added `_isProcessing` flag + `CanLoadTokenType` / `CanCancel` properties
4. Added `Mouse.OverrideCursor = Cursors.Wait` in `try`, restored `null` in `finally`
5. Added `using System.Threading.Tasks` and `using System.Windows.Input`

---

## Mandatory Rules

1. Always use `try/finally` when setting `_isProcessing = true` or `Mouse.OverrideCursor`
2. Always `await` the `Task.Run` — never fire-and-forget on UI-affecting operations
3. After `await`, no `Dispatcher.Invoke` needed — WPF SynchronizationContext handles it
4. `CanXxx` properties must `NotifyOfPropertyChange` both when set true AND when restored
5. `Mouse.OverrideCursor = null` restores the cursor — do NOT restore with `Cursors.Arrow`

## Level History

- **Lv.1** — Base: Task.Run offload pattern, Caliburn.Micro CanXxx button disable, Mouse.OverrideCursor loading cursor, try/finally cleanup, STA thread reverse problem.
  (Origin: 2026-05-08, from MyTrustID Desktop — SelectStorageViewModel UI freeze on PKCS#11 token detection)
