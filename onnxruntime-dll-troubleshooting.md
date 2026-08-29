# onnxruntime DLL troubleshooting — step by step

For a managed Windows machine where you cannot install system software.
Everything here runs in your own venv unless marked otherwise.

**Before you start:** open a terminal, `cd` to the project, activate the venv.
Every command assumes you are inside it.

---

## Using a weaker LLM as your assistant

A smaller model is genuinely useful here, but only for certain jobs. Knowing
which is the difference between help and a wasted afternoon.

**It is good at:** reading output you paste in and telling you what it contains,
comparing two lists, extracting a version number, spotting which item is
missing, reformatting text.

**It is bad at:** remembering which onnxruntime version needs which CUDA
version, knowing what is current, deciding strategy. It will state wrong version
numbers with total confidence.

**So the rule is: put the facts in the prompt.** Every prompt below carries the
information the model needs. You are asking it to compare and parse, never to
recall. If a prompt seems to over-explain, that is deliberate.

**Ignore the model if it tells you to:**
- install or reinstall the CUDA Toolkit system-wide (you can't, and you don't
  need to)
- update your GPU driver
- run anything as administrator
- disable your antivirus
- "just reinstall onnxruntime" without specifying a version

Those are the five things a weak model reaches for by default, and none of them
apply to a managed machine.

---

## Step 0 — Capture the actual error

Get the error back first. Guessing without it wastes every step below.

```
python -c "import onnxruntime as ort; ort.InferenceSession('nonexistent.onnx', providers=['CUDAExecutionProvider'])"
```

**Prediction:** a traceback, in one of three shapes:

- `LoadLibrary failed with error 126 ... onnxruntime_providers_cuda.dll` →
  CUDA/cuDNN dependency missing. **Go to Step 1.**
- `DLL load failed while importing onnxruntime_pybind11_state` →
  onnxruntime itself won't load. MSVC runtime problem. **Go to Step 6.**
- `No such file or directory: nonexistent.onnx`, no DLL complaint →
  CUDA is fine. Your problem is elsewhere; stop and re-describe it.

**Ask the LLM** (if the error doesn't clearly match one of the three):

> Below is a Python traceback. Answer only these three questions, in order,
> with no other commentary:
> 1. Does the text contain the string "error 126"? Yes or no.
> 2. Does the text contain "pybind11_state"? Yes or no.
> 3. List every filename ending in .dll that appears, one per line.
>
> [paste the traceback]

**Backout:** none, this changes nothing.

---

## Step 1 — Find out what you actually have

```
nvidia-smi
nvcc --version
python -c "import onnxruntime; print(onnxruntime.__version__)"
```

**Prediction:** `nvidia-smi` prints a table with a CUDA version top-right (that
is the *driver's* maximum, not what is installed). `nvcc --version` prints the
*toolkit* version — probably 12.9. onnxruntime prints something like `1.27.0`.

If `nvcc` is not recognised, no toolkit is installed system-wide. That is fine —
Step 3 handles it.

**Ask the LLM:**

> I will give you facts and then some terminal output. Use only the facts I
> give you. Do not use anything you remember about these version numbers.
>
> FACT 1: onnxruntime-gpu version 1.27 and above requires CUDA major version 13.
> FACT 2: onnxruntime-gpu versions below 1.27 require CUDA major version 12.
> FACT 3: CUDA major versions are not interchangeable. A build for 13 will not
> work with 12.x, and vice versa. Minor versions within a major version are
> fine — a build for 12.8 works with 12.9.
>
> From the output below, extract:
> (a) the onnxruntime version number
> (b) the CUDA toolkit version reported by `nvcc --version`
>
> Then state whether they are compatible, using FACT 1-3 only.
> Answer in three short lines. Do not suggest any fixes.
>
> [paste the output]

**Prediction:** it reports a mismatch — onnxruntime 1.27+ against CUDA 12.9.
That is your problem, and Step 2 is the fix.

**Backout:** none, read-only.

---

## Step 2 — Pin onnxruntime to a CUDA 12 build

```
pip uninstall -y onnxruntime onnxruntime-gpu
pip install "onnxruntime-gpu<1.27"
```

Both packages in the uninstall — having `onnxruntime` and `onnxruntime-gpu`
installed together is its own source of confusion.

**Prediction:** installs something like 1.26.x. Now re-run the Step 0 command.

- Error gone → **done.** That was it.
- Same error 126 → the CUDA 12 runtime DLLs are missing too. **Go to Step 3.**
- New error naming a *different* DLL → progress. **Go to Step 3.**

**Backout:**
```
pip install --force-reinstall onnxruntime-gpu
```

---

## Step 3 — Put CUDA and cuDNN inside the venv

The important one for a managed machine. You do not need the system CUDA
install — pull the runtime in as ordinary Python packages.

```
pip install nvidia-cuda-runtime-cu12 nvidia-cudnn-cu12 nvidia-cublas-cu12 nvidia-cufft-cu12 nvidia-curand-cu12
```

**Prediction:** downloads several hundred MB into site-packages. No admin
prompt, no system change. Then re-run the Step 0 command.

- Error gone → **done.**
- Still error 126 → DLLs present but not being found. **Go to Step 4.**

**Ask the LLM** (if a new error names a DLL you don't recognise):

> Here is a DLL filename from a Windows error message. Tell me only which
> library family it belongs to, based on its name prefix. Use this mapping and
> nothing else:
>   cudnn*    -> cuDNN
>   cublas*   -> cuBLAS
>   cufft*    -> cuFFT
>   curand*   -> cuRAND
>   cudart*   -> CUDA runtime
>   nvrtc*    -> CUDA runtime compiler
>   vcruntime*, msvcp*  -> Microsoft Visual C++ runtime (NOT NVIDIA)
> If the prefix is not in the list, say "unknown".
> Answer in one line, no commentary.
>
> DLL name: [paste it]

That tells you which pip package from the list above is the missing one.

**Backout:**
```
pip uninstall -y nvidia-cuda-runtime-cu12 nvidia-cudnn-cu12 nvidia-cublas-cu12 nvidia-cufft-cu12 nvidia-curand-cu12
```
Safe unless PyTorch is installed in the same venv — if it is, leave them.

---

## Step 4 — Make onnxruntime look in the venv

```
python -c "import onnxruntime as ort; ort.preload_dlls(verbose=True); print(ort.get_available_providers())"
```

**Prediction:** prints a list of DLL paths it loaded, then a provider list.

- Paths shown and `CUDAExecutionProvider` present → works when preloaded.
  **Go to Step 5.**
- Paths shown, no `CUDAExecutionProvider` → something still missing. Use the
  prompt below.
- `preload_dlls` doesn't exist → your onnxruntime predates 1.21. Reinstall:
  `pip install "onnxruntime-gpu>=1.21,<1.27"`

**Ask the LLM:**

> Below is a list of DLL file paths that a program loaded, followed by a list
> of available providers.
>
> Answer only these questions:
> 1. Does the provider list contain "CUDAExecutionProvider"? Yes or no.
> 2. Group the DLL paths by their parent directory. List each unique directory
>    once, with a count of how many DLLs came from it.
> 3. Does any path contain "site-packages"? Yes or no.
>
> Do not suggest fixes. Do not comment on versions.
>
> [paste the output]

**Prediction:** if question 3 is "no", onnxruntime is not finding what Step 3
installed — that is the actual fault, and Step 5 fixes it.

**Backout:** none, read-only.

---

## Step 5 — Make the preload permanent

If Step 4 worked only *because* of the preload, your app needs it too. Add these
two lines at the very top of your entry point, before anything else imports
onnxruntime (for insightface, above the insightface import):

```python
import onnxruntime
onnxruntime.preload_dlls()
```

**Prediction:** the app runs on GPU. If it runs but feels slow, check
`get_available_providers()` — it may have silently fallen back to CPU.

**Backout:** delete the two lines.

---

## Step 6 — MSVC runtime check

Only if Step 0 gave you `pybind11_state`, or Steps 1-5 all failed.

```
python -c "import ctypes; ctypes.CDLL('vcruntime140.dll'); ctypes.CDLL('vcruntime140_1.dll'); ctypes.CDLL('msvcp140.dll'); print('MSVC runtime OK')"
```

**Prediction:**

- `MSVC runtime OK` → not your problem. McAfee is a red herring here.
- `FileNotFoundError` on any of them → MSVC runtime missing or quarantined.
  **This is the one you need IT for.** Note which filename failed.

**Backout:** none, read-only.

---

## Step 7 — Check what McAfee took

```powershell
Get-ChildItem "C:\ProgramData\McAfee\Endpoint Security\Quarantine" -Recurse |
  Select-Object FullName, LastWriteTime, Length | Format-Table -Auto
```

**Prediction:** a list of quarantined items with timestamps.

**Ask the LLM:**

> Below is a directory listing with filenames and timestamps.
>
> Answer only:
> 1. List every entry whose filename contains "dll", "vcruntime", "msvcp",
>    "vc_redist", or "cuda". One per line.
> 2. List the three most recent entries by timestamp.
>
> No commentary, no recommendations.
>
> [paste the listing]

**Backout:** none, read-only. Do not restore files yourself — on a managed
machine that either fails silently or gets re-quarantined within minutes.

---

## If you reach here: the IT ticket

Escalate with specifics. A bounded request gets done; a debugging session
doesn't.

> Two requests, both scoped to my development environment:
>
> 1. Add a McAfee Endpoint Security **exclusion** for `D:\AI_Data\projects\`
>    including subdirectories. Python packages ship compiled `.dll` files that
>    trigger heuristic detection.
>
> 2. Restore the quarantined files listed below, or reinstall Visual Studio
>    Build Tools with the exclusion above already in place. The specific files
>    my tooling needs are `vcruntime140.dll`, `vcruntime140_1.dll`, and
>    `msvcp140.dll`.
>
> These are Microsoft-signed development tools. No system-wide policy change is
> needed — a single directory exclusion is sufficient.

Attach the Step 6 and Step 7 output.

**Ask the LLM** to tidy the ticket, not to write it:

> Rewrite the message below to be more concise and polite for an IT support
> ticket. Keep every file path and filename exactly as written. Do not add any
> technical claims that are not already present. Do not add suggestions.
>
> [paste the ticket text]

---

## Quick reference

| Error text | Cause | Step |
|---|---|---|
| `error 126 ... providers_cuda.dll` | CUDA version mismatch or missing DLLs | 2, 3 |
| `DLL load failed ... pybind11_state` | MSVC runtime missing | 6 |
| Runs but no GPU | Provider silently fell back to CPU | 4 |
| `nvcc` not recognised | No system CUDA — use venv packages | 3 |

---

## If you want to ask the LLM something freely

Paste this framing first, or it will send you after system installs you cannot
do:

> Context you must respect:
> - Windows laptop, managed by corporate IT. I have NO administrator rights.
> - I cannot install system software, change the system PATH, modify antivirus
>   settings, or update drivers.
> - I CAN install Python packages into a virtual environment I own.
> - CUDA toolkit 12.9 is installed system-wide. I cannot change that.
> - Any suggestion requiring admin rights is useless to me. Do not offer one.
>
> Given only those constraints: [your question]

That framing is worth more than any individual prompt — it removes the whole
category of answer that wastes your time.
