# Hackathon workspace: rules for Claude

This PC trains the models. The user controls Claude remotely from a laptop (Remote Control), so every command must run unattended and never block.

## Machine
- Windows 11, shell: PowerShell 5.1 (no `&&`; use `; if ($?) { ... }`)
- GPU: AMD Radeon RX 6600, 8 GB VRAM (gfx1032), ROCm PyTorch
- CPU: i5-10400F (6 cores). Never fall back to CPU training silently.
- **Python to use for everything ML:** `C:\Users\mithun\Desktop\v1\.tools\rocm-venv\Scripts\python.exe`
  (torch 2.15 + ROCm, torchvision, ultralytics). In PyTorch the AMD GPU is `device="cuda"` / `device=0`.
  - Check that `torch.cuda.is_available()` is True before training. If it's False, stop and tell the user. Do not train on the CPU.
  - Warnings like `MIOpen: Warning ... CK grouped conv library not found` / `File is unreadable ... .fdb.txt` are harmless. Ignore them.
  - With 8 GB VRAM: start with modest batch sizes, use AMP (fp16) where possible, and if you hit OOM, halve the batch size and resume.
- Do not `pip install` big packages (torch, CUDA/ROCm, TensorFlow, etc.) without asking the user first. The venv is managed by `uv`, so it has no pip.

## Before starting work
- Run `git pull` first, every session and before every new training run.

## Training: always in the background, always logged
- Never run training in the foreground. Start it as a background process and write all output (stdout + stderr) to `logs/<run-name>.log`:
  ```powershell
  New-Item -ItemType Directory -Force logs, checkpoints | Out-Null
  Start-Process -FilePath "C:\Users\mithun\Desktop\v1\.tools\rocm-venv\Scripts\python.exe" `
    -ArgumentList "-u", "train.py", "--run-name", "<run-name>" `
    -RedirectStandardOutput "logs\<run-name>.log" -RedirectStandardError "logs\<run-name>.err.log" `
    -WindowStyle Hidden -PassThru | Select-Object Id
  ```
  (`-u` = unbuffered so the log updates live.) Record the PID and tell the user the run name, PID and log path.
- `<run-name>` must be short and unique, e.g. `yolo-s-640-v1`. Never overwrite an existing log.
- Training scripts must print one parseable line per epoch, for example:
  `EPOCH 3/50 | loss 0.4213 | val_loss 0.4890 | acc 0.873 | epoch_time 92.4s`
- Ultralytics/YOLO: pass `project="checkpoints", name="<run-name>", save_period=1`. Its console output still goes to the log via the redirect above.

## Checkpoints: every epoch, resumable
- Save a checkpoint at the end of **every** epoch to `checkpoints/<run-name>/`: `last.pt` (overwrite each epoch) plus `best.pt`.
  Save model state, optimizer state, scheduler/scaler state, epoch number and best metric.
- Every training script must support `--resume`, which loads `checkpoints/<run-name>/last.pt` and continues from the next epoch.
  For YOLO, use `model = YOLO("checkpoints/<run-name>/weights/last.pt"); model.train(resume=True)`.
- If a run died (power loss, crash, reboot), resume it from `last.pt`. Do not restart from scratch.

## When the user asks for "status"
1. Read the end of `logs/<run-name>.log` (and the `.err.log`) for the current or most recent run. Use `Get-Content -Tail 50`.
2. Check whether the process is still alive: `Get-Process -Id <PID>`.
3. Reply briefly with:
   - **Epoch:** current / total
   - **Loss:** latest train loss (and val loss if present)
   - **Accuracy:** latest accuracy / mAP / main metric
   - **ETA:** (total − current epochs) × average epoch time, e.g. "~38 min left (≈ 21:40)"
   - Running / finished / crashed (if it crashed, quote the error and suggest how to resume)

## Git
- Never commit datasets, model weights, checkpoints, logs, or **any file larger than 50 MB**. These are listed in `.gitignore`.
  Before committing, check `git status`. If a large file shows up, add it to `.gitignore` instead of committing it.
  (A local pre-commit hook on this PC also blocks files over 50 MB.)
- Commit and push only when the user asks: `git add -A; git commit -m "<message>"; git push`.
  Include a short summary of results (metrics, run name) in the commit message when committing results.
