# How Nelian Launcher Works



Nelian Launcher is built with C# WinForms and WebView2, combining a native backend with a modern HTML, CSS, and JavaScript interface. The idea is simple: keep things fast, clean, and easy to use without making users deal with unnecessary setup.



Nelian is completely free to use and contains no ads. No annoying promotions, no paid features hiding behind a button, and no unnecessary bloat. Just a launcher that does what it's supposed to do.



There's a lot going on behind that simple interface. Not that it needs to show off or anything.



If you use any of these ideas or parts of this architecture in your own project, please give credit.



## UI and Rendering



Nelian uses WebView2 for its interface instead of relying entirely on traditional WinForms controls. This allows the launcher to use modern web technologies while keeping the main logic in C#.



It doesn't bundle its own browser engine, either. Nelian uses the WebView2 Runtime available on Windows, avoiding the need to ship a separate browser with the launcher. There's no reason to bring an entire browser along just to display a UI. Right, Electron?



The frontend and backend communicate through a JavaScript bridge, allowing them to exchange data and handle user interactions. Some screens are also prepared in the background before they're opened, making navigation feel faster and smoother.



## .NET Runtime



Nelian comes with its own .NET 8 runtime, so users don't need to install it separately.



This makes installation much easier and avoids unnecessary setup steps. The trade-off is a larger download, but at least everything is ready to go from the start.



## Resource Management



There's no point in keeping the entire interface active while you're busy playing Minecraft. That's right, isn't it, Feather Client?



Nelian monitors window focus and reduces background activity when the launcher hasn't been used for a while. After three seconds without focus, it can switch to a suspended state.



Before hiding the live interface, the launcher captures a preview of the current screen. This keeps the window looking the same without having to render everything in the background.



WebView2's resource usage is also reduced where possible, and the launcher can request memory cleanup to lower its working set.



The actual savings depend on the system and current workload, but the goal is straightforward: don't waste resources when there's nothing to do.



## Screen Transitions



Switching between screens should feel smooth, not like the launcher is struggling to keep up.



Nelian prepares new screens before displaying them whenever possible and uses double buffering to reduce flickering during resizing and other UI updates.



It's a small detail, but it helps the whole thing feel more polished.



## Startup and Updates



When Nelian starts, it checks that the required components are available and up to date before continuing.



This includes the expected Java 21 runtime, launcher updates, client files, and NelianGuard components. If required files are missing or fail validation, the launcher can download the necessary files again.



These checks help prevent outdated or incomplete files from causing problems later.



## Accounts and Authentication



Nelian supports Microsoft authentication as well as offline profiles.



When a valid Microsoft session is already available, the launcher can reuse it instead of asking the user to log in again. If there's no usable premium session but exactly one offline profile is available, it can select that profile automatically.



Account management is kept separate from the interface, making it easier to maintain and expand without turning the entire launcher into one giant mess.



Because nobody wants that.



## Launching Minecraft



When you click Play, Nelian prepares the game and performs a verification step to ensure the launcher and client are working with the expected session.



A temporary token and some verification data are generated as part of this process. The launcher and client then check the expected values before proceeding.



This helps connect the client to the intended launcher session without exposing unnecessary implementation details.



## NelianGuard



NelianGuard is Nelian's anti-cheat and integrity protection system. Its job is to detect suspicious changes to the client while it's running.



It checks relevant memory regions, looks for signs of unauthorized code injection and execution hooks, and verifies the integrity of loaded DLLs against expected signatures.



If a serious integrity violation is detected, the client can terminate to prevent further execution in an unexpected state.



Of course, no anti-cheat is perfect. The goal is to make unauthorized modifications harder to pull off and detect common forms of tampering.



## Skin Manager



Nelian includes a dedicated skin management system that communicates with Minecraft's skin services when needed.



It supports authenticated requests, skin uploads, and different player model variants. Saved skin information is also stored locally, allowing the launcher to keep track of custom profiles.



Everything stays organized instead of having skin-related code scattered throughout the entire project.



## Modrinth Integration



Nelian connects to Modrinth's official API to display mods and modpacks directly in the launcher.



It also supports Fabric metadata, allowing the launcher to prepare the required profiles and launch configurations for modded Minecraft.



The goal is to make finding and setting up content easier without forcing users to handle every step manually.



## I don't have the time read all that bracacho

 I got you :D

Nelian Launcher focuses on a few simple things and its a clean interface, easy setup, efficient resource usage, and a reliable connection between the launcher and the client.



It's completely free, has no ads, and isn't built around pushing unnecessary extras onto its users. The goal is to keep the experience simple and let people enjoy Minecraft without getting in their way.



The frontend handles the visuals, C# takes care of the underlying logic, and NelianGuard adds another layer of integrity protection.



There's still room to improve, obviously. But that's part of building software.



And yes, there's a surprising amount of work behind that little Play button. Don't get the wrong idea, though. I'm not bragging.
