# Gameplay Message Subsystem

**Works with Unreal Engine 5.8.**

Lets gameplay objects send messages to each other without knowing about each other. One object **broadcasts** a
message (any struct) on a **channel** (a Gameplay Tag), and every object **listening** on that channel receives it.
For example: a pickup broadcasts on `Message.Score.Changed`, and the HUD, an achievement tracker and a sound manager
all react, while the pickup has no reference to any of them.

This is Epic's Gameplay Message Router plugin from the Lyra sample, packaged on its own. The plugin's file is named
`GameplayMessageRouter.uplugin`, and it shows as **Gameplay Message Subsystem** in the editor's Plugins window. Both
names are correct; don't rename the files.

---

## 1. Install

You need a **C++ project** (if yours is Blueprint-only, add any C++ class once: *Tools > New C++ Class*) and
Visual Studio with the "Game development with C++" workload.

1. Close the editor.
2. Copy this folder into your project's `Plugins` folder as `GameplayMessageRouter` (make the `Plugins` folder if it
   doesn't exist):
   ```
   MyProject/
     MyProject.uproject
     Plugins/
       GameplayMessageRouter/
         GameplayMessageRouter.uplugin
   ```
3. Right-click `MyProject.uproject` > **Generate Visual Studio project files**, then open the project. Click **Yes**
   when it asks to rebuild the missing modules.

The plugin is enabled by default. It also turns on the engine's **GameplayTagsEditor** plugin.

---

## 2. Set up your channels and messages

1. **Channels** are Gameplay Tags. Add them in *Edit > Project Settings > Project > Gameplay Tags*, for example
   `Message.Score.Changed`.
2. **Messages** are structs. In Blueprint: *Add > Blueprint > Structure*. In C++: a `USTRUCT(BlueprintType)`.
   Everyone on one channel must use the **same struct type**; a listener expecting a different type doesn't receive
   the message, and an error is logged.

---

## 3. Blueprint

### Send

**Get Game Instance Subsystem > Gameplay Message Subsystem**, then **Broadcast Message**:

- **Channel**: the tag to send on.
- **Message**: your struct.

### Receive

**Listen for Gameplay Messages** (async node):

- **Channel**: the tag to listen on.
- **Payload Type**: your struct type.
- **Match Type**: *Exact Match* (only `A.B`) or *Partial Match* (`A.B` and anything under it, such as `A.B.C`).
- **On Message Received** fires for every message; **Payload** gives you the struct and **Actual Channel** the exact
  tag it was sent on.

Keep the node's **Async Action** output and call **Cancel** on it when you stop listening (for example in
*Event End Play*). If you don't, it only cleans itself up at the next message after the Blueprint that started it
is destroyed.

---

## 4. C++

Add `"GameplayMessageRuntime"` and `"GameplayTags"` to your module's `PublicDependencyModuleNames` in its `.Build.cs`.

```cpp
#include "GameFramework/GameplayMessageSubsystem.h"

// Send
FMyScoreMessage Message;
Message.NewScore = 42;
UGameplayMessageSubsystem::Get(this).BroadcastMessage(ScoreChangedTag, Message);

// Receive (keep the handle)
ListenerHandle = UGameplayMessageSubsystem::Get(this).RegisterListener(
    ScoreChangedTag, this, &UMyWidget::OnScoreChanged);

void UMyWidget::OnScoreChanged(FGameplayTag Channel, const FMyScoreMessage& Message) { /* ... */ }

// Stop listening (e.g. in EndPlay / NativeDestruct)
ListenerHandle.Unregister();
```

`RegisterListener` also takes a lambda, and an optional `EGameplayMessageMatch::PartialMatch` to receive child tags.

---

## 5. Good to know

- Messages are delivered **immediately**, on the same frame, to listeners in the same game instance. They aren't
  replicated: on a multiplayer game each client and the server only see their own broadcasts.
- Listeners are called in no particular order; don't rely on it.
- Always unregister listeners you no longer need; a listener whose object is gone is skipped, but the handle stays
  registered until you remove it.
- Log category: `LogGameplayMessageSubsystem`.

---

## 6. Troubleshooting

| Problem | Fix |
|---|---|
| Nothing is received | The broadcast and the listener must use the same tag (or *Partial Match* on a parent tag), and the listener must be registered **before** the broadcast. |
| "Struct type mismatch" error in the log | The sender and a listener use different struct types on the same channel. Use one struct per channel. |
| "Modules are missing or built with a different engine version" | Click Yes to rebuild (needs Visual Studio). |
| Nodes don't show in Blueprint | Check the plugin is enabled in *Edit > Plugins* and the project was rebuilt. |
