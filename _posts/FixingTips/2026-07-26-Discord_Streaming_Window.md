---
title: How to fix Discord's streaming window not open on Hyprland # Tên bài viết sẽ hiện to đùng
date: 2026-07-26 12:35:00 +0700      # Thời gian đăng (Quan trọng: +0700 là giờ VN)
categories: [Fixing Tips, Hyprland]         # Danh mục lớn, danh mục con
tags: [blog, linux, hyprland, discord, fixing]     # Tag để tìm kiếm (viết thường)
author: "RobinVA"
---


# The problem

So I hopped in a Discord voicechat to see my friends' gaming streams. I would like to show them something so I tried to open streaming, but I couldn't. There was nothing
 that popped out off my screen after clicking this button:

![button](/assets/img/FixingTips/Discord_Streaming_Window/Icon.jpg)

At that moment, I was using Flatpak Discord. I also installed Vesktop but the problem was also not solved.

Then, I decided to run Vesktop via terminal to see if any logs appeared. Just like what I expected, this was the log I encountered:

```bash
[3555:0725/195200.630261:ERROR:third_party/webrtc/modules/desktop_capture/linux/wayland/screen_capture_portal_interface.cc:54] Failed to request session: GDBus.Error:org.freedesktop.DBus.Error.NameHasNoOwner: Could not activate remote peer 'org.freedesktop.portal.Desktop': startup job failed 
[3555:0725/195200.630284:ERROR:third_party/webrtc/modules/desktop_capture/linux/wayland/base_capturer_pipewire.cc:93] ScreenCastPortal failed: 3 Error during screenshare picker Failed to get sources. 
(node:3555) UnhandledPromiseRejectionWarning: TypeError: Video was requested, but no video stream was provided at VesktopMain:97:5759 
(node:3555) UnhandledPromiseRejectionWarning: Unhandled promise rejection. This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). To terminate the node process on unhandled promise rejection, use the CLI flag --unhandled-rejections=strict (see https://nodejs.org/api/cli.html#cli_unhandled_rejections_mode). (rejection id: 3
```

---

# Explaination

The story when you click the button

There are three layers involved in this process:

1. Vesktop: The app that you press the "Share" button;

2. `xdg-desktop-portal`: The "middleman" that pops the "Choose screen" window out for you;

> ["An XDG Desktop Portal is a program that lets other applications communicate with the compositor through D-Bus.](https://wiki.hypr.land/Hypr-Ecosystem/xdg-desktop-portal-hyprland/#:~:text=An%20XDG%20Desktop,or%20screen%20sharing.)

> [A portal implements certain functionalities, such as opening file pickers or **screen sharing**."](https://wiki.hypr.land/Hypr-Ecosystem/xdg-desktop-portal-hyprland/#:~:text=An%20XDG%20Desktop,or%20screen%20sharing.)

> ---*Hyprland Wiki*---

3. `graphical-session.target`: The sign signals systemd that graphical session is active or not.

**NOTE**: 

- `graphical-session.target` is useful since some graphical applications such as `waybar`, `polybar`, `dunst` will crash if compositor/window manager (Hyprland, KDE, GNOME) doesn't finish its startup yet. Compositor can only run after you log in successfully;

<a id="not_auto_open"></a>

-  However, only KDE and GNOME automatically activates `graphical-session.target`, but Hyprland does not.

To see the unit service config file of `xdg-desktop-portal`, run `systemctl --user cat xdg-desktop-portal.service`:

```bash
❯ systemctl --user cat xdg-desktop-portal.service
# /usr/lib/systemd/user/xdg-desktop-portal.service
[Unit]
Description=Portal service
PartOf=graphical-session.target
Requisite=graphical-session.target
After=graphical-session.target

[Service]
Type=dbus
BusName=org.freedesktop.portal.Desktop
ExecStart=/usr/libexec/xdg-desktop-portal
Slice=session.slice
```

Pay attention to `Requisite=graphical-session.target`. This implies that `xdg-desktop-portal.service` can run only if `graphical-session.target` is activated.

> ["Warning]
> [The systemd xdg-desktop-portal.service may require an active graphical-session.target, which Hyprland **doesn’t start by default**. See systemd to set that up."](https://wiki.hypr.land/Hypr-Ecosystem/xdg-desktop-portal-hyprland/#:~:text=Usage-,Warning,Hyprland%20doesn%E2%80%99t%20start%20by%20default.%20See%20systemd%20to%20set%20that%20up.,-XDPH%20is%20automatically)

> ---*Hyprland Wiki*---

Look at the log:

```bash
Failed to request session: 
GDBus.Error:org.freedesktop.DBus.Error.NameHasNoOwner: Could
not activate remote peer 'org.freedesktop.portal.Desktop': startup
job failed
```

`xdg-desktop-portal`, via the alias `org.freedesktop.portal.Desktop`, couldn't be activated. Therefore, the culprit should be `graphical-session.target`.

Use `journalctl -xe` to confirm the error:

``` bash
Jul 25 18:10:36 67canyouhelpmerev69 systemd[2050]: graphical-session.target - Current graphical user session is inactive.
...
```

Based on the result and [this](#not_auto_open), I can confidently conclude that `graphical-session.target`'s inactivation is the culprit.

---

# Solution

1. Setup `graphical-session.target` (if needed)

- Check your `graphical-session.target`'s state:

```bash
systemctl --user status hyprland-session.target
```

If it prints out `Unit hyprland-session.target could not be found.`, follow the procedure below. If not, skip to step 2.

- Create file:

```bash
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/hyprland-session.target
```

- Inside `hyprland-session.target`, paste:

```bash
[Unit]
Description=Hyprland session
BindsTo=graphical-session.target
Wants=graphical-session-pre.target
After=graphical-session-pre.target
PropagatesStopTo=graphical-session.target
```

2. Reload systemd user

```bash
systemctl --user daemon-reload
```

3. Activate target and portal

```bash
systemctl --user start hyprland-session.target
systemctl --user status graphical-session.target #If "Active: active", proceeds

systemctl --user start xdg-desktop-portal
systemctl --user status xdg-desktop-portal #If "Active: active", you are ready to stream.
```

4. Setup auto activation

Check if your `~/.config/hypr/conf/autostart.lua` has this line or not:

```lua
hl.on("hyprland.start", function()
	--Other cmds
	
    hl.exec_cmd("systemctl --user start hyprland-session.target") --This line

	--Other cmds
end)
```

If not, add it.

---

# Have fun streaming!
