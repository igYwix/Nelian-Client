# Nelian Launcher Working Principle

This file explains the entire background of Nelian!

You can view the code from the **/NelianSource** repository.

Please don't forget to give credit if you use it in your project.

---

## Launcher

Nelian Launcher is developed with **C# WinForms**.

### MainThread.cs

`MainThread.cs` is the section where pages and the Titlebar are drawn. This section is like the heart of Nelian Launcher.

This file is a **Form** file and draws pages within itself.

The main screen (aka `Main.cs`) is a **UserControl** file. This UserControl is drawn inside the `MainThread.cs` Form.

`MainThread` determines which page will be shown when, and pages listen to this system.

### Splash.cs

`Splash.cs` is the **Loading** screen you see when the launcher first starts.

When the launcher starts, it first performs update checks. It downloads missing files or runs `Updater.exe` if a major update is required.

When the launcher is ready to switch to a new page, it sends a notification to `MainThread`.

Upon receiving this notification, `MainThread` first calls the new page and loads it over the current page. After the new page is successfully loaded, it `Dispose`s the `Splash` UserControl running in the background.

This way, while the old page is being removed during the transition, the loading of the new page is shown to the user more smoothly.

---

## AccountManager.cs

`AccountManager.cs` is the section where Minecraft accounts are managed within the launcher.

Users can add their Microsoft account to the launcher through this section. The login process is carried out through Microsoft's authentication system, and after a successful login, the necessary account and session information is stored by the launcher.

When launching Minecraft, the launcher uses this session information to log in with the player's account.

AccountManager also manages operations such as displaying registered accounts in the launcher and allowing the user to select them.

Since multiple accounts can be added, the user can select which account they want to use before launching Minecraft. The selected account is then passed to `MainThread` and the launcher's relevant launch systems during the Minecraft launch process.

In short, `AccountManager.cs` controls **the account management process from adding a Microsoft account to the launcher to using it when launching Minecraft**.

---

## AccountControl.cs

`AccountControl.cs` is the **UserControl** file that creates the interface cards for accounts displayed in `AccountManager`.

A separate `AccountControl` is created for each account. This control displays the account's username, account identifier, avatar, and available actions.

`AccountControl` supports two different account types:

- Microsoft account
- Offline account

Microsoft accounts use the `JEGameAccount` object. Offline accounts operate only using the username.

### Microsoft Account

When creating a Microsoft account, the `JEGameAccount` object is assigned to the `Account` property.

The username is retrieved from:

```csharp
account.Profile?.Username
```

while the account identifier is retrieved from:

```csharp
account.Identifier
```

This information is passed to the `lbUsername` and `lbIdentifier` Labels on the card.

The account's Minecraft UUID is also retrieved from:

```csharp
account.Profile?.UUID
```

and is used to load the avatar.

### Offline Account

A separate constructor exists for offline accounts:

```csharp
public AccountControl(string username)
```

When this constructor is called, the control marks itself as an offline account:

```csharp
_isOffline = true;
_offlineUsername = username;
```

The username is written directly to the `lbUsername` Label. The account type is displayed using the `AccountManager.OfflineMode` text retrieved from `LanguageManager`.

Offline accounts don't need a `JEGameAccount` object, so unlike Microsoft accounts, operations are performed using the username.

### Avatar System

The Minecraft avatar on the account card is loaded by the `LoadAvatarAsync()` method.

Using the provided UUID, the player's Minecraft head is downloaded from `mc-heads.net`:

```text
https://mc-heads.net/head/{uuid}
```

The downloaded data is converted to an `Image` object and assigned to the `pbAvatar` PictureBox.

The avatar loading process is performed asynchronously, so the launcher's interface doesn't have to wait during this operation.

If an error occurs while loading the avatar, the control catches it and clears the avatar:

```csharp
catch
{
    pbAvatar.Image = null;
}
```

For offline accounts, a default avatar is loaded using a fixed UUID.

### Interface Alignment

When `AccountControl_Load()` runs, the Labels and avatar on the account card are centered.

`CenterLabels()`:

- Username
- Account identifier

are centered horizontally.

The actual width of the text is calculated using `TextRenderer.MeasureText()` and the Label's position is adjusted accordingly.

The avatar is placed in the center of the parent control by `CenterPictureBox()`.

### Login Button

When the Login button is pressed, it first checks whether the account is offline.

If it's an offline account:

```csharp
OfflineLoginClicked?.Invoke(_offlineUsername);
```

is executed and the offline username is sent to the relevant system.

For Microsoft accounts:

```csharp
LoginClicked?.Invoke(this, Account);
```

the event is triggered.

Thus, instead of performing the login operation directly, `AccountControl` notifies the `AccountManager` or other systems that manage the relevant operation.

This structure ensures that `AccountControl` is only responsible for the interface and user interaction.

### Remove Button

The Remove button works similarly to the Login button.

For offline accounts:

```csharp
OfflineRemoveClicked?.Invoke();
```

the event is triggered.

For Microsoft accounts:

```csharp
RemoveClicked?.Invoke(this, Account);
```

is executed.

Thus, the account deletion process is not performed directly within `AccountControl`. The control only notifies the relevant system that the account wants to be deleted.

### Event System

`AccountControl` contains four different events:

```csharp
public event EventHandler<JEGameAccount>? LoginClicked;
public event EventHandler<JEGameAccount>? RemoveClicked;
public event Action<string>? OfflineLoginClicked;
public event Action? OfflineRemoveClicked;
```

These allow communication between `AccountControl` and `AccountManager` without creating a direct dependency.

For example, when the user presses the Login button, `AccountControl` doesn't launch Minecraft itself. Instead, it triggers the relevant event. If `AccountManager` is listening to this event, it performs the necessary operation itself.

This structure makes the launcher's account cards more modular and allows `AccountControl` to work independently from the account management system.

### LanguageManager

Text shown to the user, such as button labels and account types, is retrieved through `LanguageManager` instead of being hardcoded.

For example:

```csharp
LanguageManager.Get("AccountControl.Login")
```

retrieves the Login button text.

The same system is used for the Remove button and Offline Mode text.

This makes it easier to translate the launcher into different languages.

### Double Buffering

Double Buffering is enabled in `AccountControl` to reduce flickering that may occur during WinForms drawing:

```csharp
this.DoubleBuffered = true;
```

Additionally:

```csharp
ControlStyles.OptimizedDoubleBuffer |
ControlStyles.AllPaintingInWmPaint |
ControlStyles.UserPaint
```

settings are also activated.

This provides smoother visuals, especially when the launcher contains animations and frequently redrawn interface elements.

In short, `AccountControl.cs` **manages the visual representation of the account and user interactions within the launcher, rather than managing the account itself.** It communicates with `AccountManager` through the event system for operations like login or removal.

---

## UpdateManager.cs

`UpdateManager.cs` is responsible for updating launcher, client, and guard files.

### Update URLs

```csharp
private const string MANIFEST_URL = "https://raw.githubusercontent.com/FixzyXqw/Nelian-Client/main/manifest.json";
private const string HASH_CHECK_URL = "https://raw.githubusercontent.com/FixzyXqw/Nelian-Client/main/FAX.Update";
private const string CLIENT_DOWNLOAD_URL = "https://github.com/FixzyXqw/Nelian-Client/releases/download/NelianClientUpdate/Nelian.jar";
private const string GUARD_DOWNLOAD_URL = "https://github.com/FixzyXqw/Nelian-Client/releases/download/NelianClientUpdate/NelianGuard.dll";
private const string GUARD_HASH_CHECK_URL = "https://raw.githubusercontent.com/FixzyXqw/Nelian-Client/main/NelianGuard.hash";
```

### File Locations

```csharp
private static string ProgramFilesRoot => Path.Combine(
    Environment.GetFolderPath(Environment.SpecialFolder.ProgramFilesX86),
    "Nelian");

private static string LocalManifestPath => Path.Combine(ProgramFilesRoot, "manifest.json");
private static string LocalClientPath => Path.Combine(ProgramFilesRoot, "Nelian.runtime");
private static string LocalGuardPath => Path.Combine(ProgramFilesRoot, "NelianGuard.dll");
private static string UpdaterPath => Path.Combine(ProgramFilesRoot, "Updater.exe");
```

### Progress Callback

```csharp
public static event Action<int>? ProgressChanged;
```

### Update Checks

#### CheckLauncherUpdateAsync()
Checks for launcher updates via manifest.json. Compares remote and local manifests.

#### CheckClientUpdateAsync()
Checks for client (Nelian.runtime) updates using the hash in FAX.Update.

#### CheckGuardUpdateAsync()
Checks for NelianGuard.dll updates using the hash in NelianGuard.hash.

#### CheckAllUpdatesAsync()
Checks all updates in a single method.

```csharp
public static async Task<UpdateCheckResult> CheckAllUpdatesAsync()
{
    bool launcher = await CheckLauncherUpdateAsync();
    if (launcher) return UpdateCheckResult.LauncherUpdate;

    bool client = await CheckClientUpdateAsync();
    if (client) return UpdateCheckResult.ClientUpdate;

    bool guard = await CheckGuardUpdateAsync();
    if (guard) return UpdateCheckResult.GuardUpdate;

    return UpdateCheckResult.NoUpdate;
}
```

### Applying Updates

#### ApplyLauncherUpdate()
Updates the launcher by running Updater.exe.

#### ApplyClientUpdateAsync()
Downloads the client file, performs hash verification, and updates.

#### ApplyGuardUpdateAsync()
Downloads NelianGuard.dll, performs hash verification, and updates.

### Changelog

```csharp
public static string GetChangelog()
```
Fetches changelog information via GitHub API.

### UpdateCheckResult Enum

```csharp
public enum UpdateCheckResult
{
    NoUpdate,
    LauncherUpdate,
    ClientUpdate,
    GuardUpdate,
}
```

---

## Logger.cs

`Logger.cs` is the centralized logging system used throughout the launcher.

### Log Levels

| Level | Description |
|-------|-------------|
| INFO | General information |
| ERROR | Error messages |
| WARN | Warnings |
| DEBUG | Debug information |
| CML | CmlLib related logs |
| MC-OUT | Minecraft output |
| MC-ERR | Minecraft errors |

### Event System

```csharp
public static event EventHandler<string> LogEntryAdded;
```

### Methods

```csharp
public static void Log(string message, string level = "INFO")
public static void LogError(string message)
public static void LogWarning(string message)
public static void LogDebug(string message)
public static void LogInfo(string message)
public static void LogCml(string message)
public static void LogMcOut(string message)
public static void LogMcError(string message)
```

### Log Format

```csharp
string logLine = $"[{DateTime.Now:HH:mm:ss}] [{level}] {message}";
```

Example output:
```
[14:23:45] [INFO] Launcher started
[14:23:46] [MC-OUT] Starting Minecraft...
[14:23:47] [ERROR] Failed to load asset
```

---

## LogViewer.cs

`LogViewer.cs` is the form that displays logs with colors.

### Features

- Real-time log updates
- Color-coded log lines
- Copy to clipboard
- Export to file
- Open log folder
- Open latest.log file

### Log Colors

| Log Level | Color Code | Color |
|-----------|------------|-------|
| ERROR / MC-ERR | #FF6464 | Red |
| WARN | #FFC864 | Orange |
| MC-OUT | #B4D2F0 | Light Blue-Gray |
| CML | #64DCFF | Light Blue |
| DEBUG | #C8C8C8 | Gray |
| INFO | #FFFFFF | White |

### Methods

```csharp
private void AppendColoredLog(string log)
private void Logger_LogEntryAdded(object sender, string log)
private void BtnExport_Click(object sender, EventArgs e)
```

### Static Usage

```csharp
public static bool IsOpen => _instance != null && !_instance.IsDisposed;
public static void OpenLogger()
```

---

## Main.cs

`Main.cs` is the main screen of the launcher and manages game launch operations.

### Launch Types

#### Vanilla Launch

```csharp
private async void LaunchVanilla(string version)
```

Features:
- Uses CmlLib
- Supports version selection
- Auto-connect to server
- Fullscreen / Windowed mode

#### Nelian Client Launch

```csharp
private async void Launch()
```

Features:
- Custom Nelian client
- NelianGuard anti-cheat integration
- HWID-based session validation
- Temporary version creation
- File integrity verification

#### Modrinth Instance Launch

```csharp
private async Task LaunchModrinthInstance(Instance instance)
```

Features:
- Fabric mod loader support
- Instance isolation
- Custom game directory

### Launch Options

```csharp
MLaunchOption launchOption = new MLaunchOption
{
    Session = _session,
    MaximumRamMb = SettingsHelper.GetMemory(),
    FullScreen = SettingsHelper.IsFullscreenEnabled(),
    ScreenWidth = SettingsHelper.GetWindowWidth(),
    ScreenHeight = SettingsHelper.GetWindowHeight(),
    GameLauncherName = "Nelian",
    GameLauncherVersion = "7.8",
    ServerIp = serverIp,
    ServerPort = serverPort,
    ExtraJvmArguments = jvmArgs
};
```

### JVM Arguments

```csharp
new MArgument("-XX:+AggressiveOpts")
new MArgument("-XX:+AlwaysPreTouch")
new MArgument("-XX:+UseCompressedOops")
new MArgument("-XX:+DisableExplicitGC")
new MArgument("-XX:+UseStringDeduplication")
new MArgument("-XX:+ParallelRefProcEnabled")
new MArgument("-XX:-OmitStackTraceInFastThrow")
new MArgument("-XX:+OptimizeStringConcat")
new MArgument("-XX:+UseLargePages")
new MArgument("-XX:LargePageSizeInBytes=2m")
```

### LaunchBehavior

```csharp
public enum LaunchBehavior
{
    KeepOpen,   // Launcher stays open
    Minimize,   // Launcher is minimized
    Close       // Launcher is closed
}
```

### Nelian Client Launch Flow

1. UI controls are disabled
2. NelianGuard.dll is checked / updated
3. Assets are copied to Minecraft folder
4. Random token is generated
5. Nelian.runtime is copied to versions folder
6. SHA256 file integrity verification
7. Version JSON is created
8. Temporary session data is written
9. JVM arguments are prepared
10. Launch options are created
11. Minecraft process is started
12. Process events are managed
13. UI is restored after game closes

### Session Management

```csharp
private void WriteTempSession()
{
    string path = Path.Combine(Path.GetTempPath(), "KFAX.dat");
    string hwid = GetHWID();
    string userId = _session.Username;
    string data = $"{userId}|{hwid}|{DateTimeOffset.UtcNow.ToUnixTimeSeconds()}";
    string encoded = Convert.ToBase64String(Encoding.UTF8.GetBytes(data));
    File.WriteAllText(path, encoded);
}
```

### Error Codes

| Code | Description |
|------|-------------|
| 0 | Normal exit |
| -1073740791 | Terminated by NelianGuard / AntiCheat |
| Other | Game crashed |

### Magic Particle System

```csharp
private class MagicParticle
{
    public float X { get; set; }
    public float Y { get; set; }
    public float TargetX { get; set; }
    public float TargetY { get; set; }
    public float Speed { get; set; }
    public int Size { get; set; }
    public string Symbol { get; set; }
    public int Alpha { get; set; }
    public Color Color { get; set; }
}
```

Features:
- Animated rune particles
- Target tracking (follows the launch button)
- Alpha fading when approaching
- Can be enabled/disabled from settings

### Discord RPC

```csharp
public void InitializeRPC()
{
    if (!SettingsHelper.IsDiscordRpcEnabled()) return;

    client = new DiscordRpcClient("sneakybeakylike!");
    client.Initialize();

    client.SetPresence(new RichPresence
    {
        Details = _session?.Username == "Ywix"
            ? "Logged in as a Developer: " + _session.Username
            : $"Logged in as {_session?.Username ?? "Guest"}",
        State = "On Main Menu",
        Assets = new Assets
        {
            LargeImageKey = "nelian",
            LargeImageText = "Nelian Client",
            SmallImageKey = "nelianico",
            SmallImageText = "Nelian Launcher"
        },
        Timestamps = Timestamps.Now
    });
}
```

### Modrinth Integration

```csharp
public void ShowModrinthPanel()
private async void ModrinthMain_InstanceLaunchRequested(object sender, Instance instance)
private async Task LaunchModrinthInstance(Instance instance)
```

Fabric meta URL:
```
https://meta.fabricmc.net/v2/versions/loader/{version}/{loaderVersion}/profile/json
```
