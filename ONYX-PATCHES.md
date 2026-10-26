# ONYX patches to vendored JUCE 8.0.12

Local changes to this JUCE copy. Each one is marked `// ONYX patch:` in the source. Re-apply them after any JUCE upgrade.

## LV2 worker thread: semaphore wait instead of a 1 ms poll (2026-10-20)

- File: `modules/juce_audio_processors_headless/format_types/juce_LV2PluginFormatImpl.h`, `lv2_host::SharedThreadedWorker`.
- Problem: upstream's worker loop ran `sleep_for(1ms)` whenever its queue was empty. That gave about 965 wakeups/s in plugin-host on the Linux tablet as soon as an LV2 plugin with the worker extension was loaded (x42 dpl limiter).
- Fix: a POSIX counting semaphore.
  - `schedule()` calls `sem_post` once per successful push.
  - The destructor posts once after setting `shouldExit`.
  - The thread calls `sem_wait` when the queue is empty.
- RT safety: `schedule()` runs on the audio thread. `sem_post` is lock-free and async-signal-safe, with no mutex. `juce::WaitableEvent` is not used because it is a mutex plus condvar. The project builds with C++17, so `std::atomic::wait` is not available.
- Scope: only enabled when `JUCE_LINUX || JUCE_BSD`. Other platforms keep the upstream poll, because macOS has no unnamed `sem_init`. Override it with `-DJUCE_ONYX_LV2_WORKER_SEMAPHORE=0/1`. If `sem_wait` fails with anything other than EINTR, the loop falls back to a 1 ms sleep so it never spins.
- Test: `test/plugin-host/test_lv2_worker_wakeup.cpp`. It runs standalone, and its build recipe is in the file header.
- On-device check: `/proc/<plugin-host pid>/task/<worker tid>/status` should show `voluntary_ctxt_switches` going from about 965/s to about 0/s while idle.

## VST3 EditControllerParameterDispatcher: 10 Hz when idle instead of a fixed 60 Hz (2026-10-20)

- File: `modules/juce_audio_processors_headless/format_types/juce_VST3PluginFormatImpl.h`, `EditControllerParameterDispatcher`.
- Problem: every hosted VST3 instance ran a 60 Hz `Timer` for its whole life to forward processor→controller parameter values, even with nothing to forward. On the Linux tablet that was the main source of message-thread wakeups (~60/s) and JUCE TimerThread wakeups (~180/s) in plugin-host and in every instrument-host.
- Fix: `flush()` now returns whether it sent anything. After 30 empty ticks (~0.5 s) the timer drops to 10 Hz, and the first non-empty flush goes back to 60 Hz.
- Cost: after a quiet spell, the first change reaches the plugin's controller/editor up to 100 ms late. The host-side value (`cachedParamValues`, `getValue()`) is not delayed, and `getState`/`setState` still flush synchronously.
- Test: `test/plugin-host/test_juce_idle_timers.cpp` (standalone, recipe in its header).

## LV2 ProcessorToUi: timer only while a UI is attached (2026-10-20)

- File: `modules/juce_audio_processors_headless/format_types/juce_LV2PluginFormatImpl.h`, `ProcessorToUi`.
- Problem: this shared object (a `SharedResourcePointer`, created with the first LV2 instance) started a 60 Hz timer in its constructor and never stopped it, even though it only services attached UIs (calls `idle()` and delivers messages). With an LV2 plugin loaded and no LV2 editor open, plugin-host still woke 60 times a second.
- Fix: `addUi` starts the timer and `removeUi` stops it when the last UI goes, dropping whatever is still queued. `pushMessage` drops messages whose listener is null: an instance without an editor sends its output ports that way, and upstream's tick discarded them anyway. That keeps the buffer from growing while the timer is stopped.
- Test hook: `isUiTimerRunning()`. Test: `test/plugin-host/test_juce_idle_timers.cpp`.
