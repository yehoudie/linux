# Gnome

## Install

```
$ pacman -S gnome
$ pacmn -S gnome-tweaks
```

### Components
https://github.com/GNOME/...
- [Baobab](https://github.com/GNOME/baobab): 
    Gnome disk usage analyzer
- [epiphany](https://github.com/GNOME/epiphany)
    A GNOME web browser based on the WebKit rendering engine.
- [Evince](https://github.com/GNOME/evince): 
    Evince is a document viewer capable of displaying multiple and single page document formats like PDF and Postscript.
- [GDM](https://github.com/GNOME/gdm):
    The GNOME Display Manager is a system service that is responsible for providing graphical log-ins and managing local and remote displays.
- [Gnome Connections](https://apps.gnome.org/Connections/)
    Connections allows you to connect to and use other desktops. 
- [gnome-menus](https://github.com/GNOME/gnome-menus)
    gnome-menus contains the libgnome-menu library, the layout configuration files for the GNOME menu, as well as a simple menu editor. 
    The layout files control the menu layout for the GNOME Classic desktop. 
    They are unused in normal GNOME sessions and have no effect outside GNOME Classic.
- [gnome-session](https://github.com/GNOME/gnome-session):
    gnome-session contains the GNOME session manager, as well as a configuration program to choose applications starting on login.
- [gnome-software](https://github.com/GNOME/gnome-software):
    Software allows users to easily find, discover and install apps. It also keeps their OS, apps and devices up to date without them having to think about it, and gives them confidence that their system is up to date. 
- [gnome-user-share](https://github.com/GNOME/gnome-user-share):
    gnome-user-share is a small package that binds together various free software projects to bring easy to use user-level file sharing to the masses.
- [Grilo Plugins](https://github.com/GNOME/grilo-plugins):
    Grilo is a framework for browsing and searching media content from various sources using a single API.
- [gvfs](https://github.com/GNOME/gvfs):
    GVfs is a userspace virtual filesystem implementation for GIO (a library available in GLib).
- [loupe](https://github.com/GNOME/loupe):
    GNOME's default image viewer.
- [malcontent](https://github.com/endlessm/malcontent):
    malcontent implements support for restricting the type of content accessible to non-administrator accounts on a Linux system. 
- [mutter](https://github.com/GNOME/mutter):
    Mutter is a Wayland display server and X11 window manager and compositor library.
- [nautilus](https://github.com/GNOME/nautilus):
    This is the project of the Files app, a file browser for GNOME, internally known by its historical name nautilus.
- [orca](https://github.com/GNOME/orca):
    Screen reader that provides access to the graphical desktop
- [rygel](https://github.com/GNOME/rygel):
    Rygel is a home media solution that allows you to easily share audio, video and pictures, and control of media player on your home network.
- [simple-scan](https://github.com/GNOME/simple-scan):
    Document Scanner is a document scanning application for GNOME. 
    It allows you to capture images using image scanners (e.g. flatbed scanners) that have suitable SANE drivers installed.
- [snapshot](https://github.com/GNOME/snapshop):
    Screenshot, etc
- [sushi](https://github.com/GNOME/sushi):
    A quick previewer for Files (nautilus), the GNOME desktop file manager.
- [tecla](https://github.com/GNOME/tecla):
    A keyboard layout viewer.
- [totem](https://github.com/GNOME/totem):
    A movie player for the GNOME desktop based on GStreamer
- [xdg-desktop-portal-gnome](https://github.com/GNOME/xdg-desktop-portal-gnome):
    A backend implementation for xdg-desktop-portal that is using GTK and various pieces of GNOME infrastructure, such as the org.gnome.Shell.Screenshot or org.gnome.SessionManager D-Bus interfaces.
- [xdg-user-dirs-gtk](https://github.com/GNOME/xdg-user-dirs-gtk):
    xdg-user-dirs-gtk is a companion to xdg-user-dirs that integrates it into the Gnome desktop and Gtk+ applications.
- [yelp](https://github.com/GNOME/yelp):
    The default help viewer for the GNOME desktop. Yelp provides a simple graphical interface for viewing Mallard, DocBook, HTML, man, and info formatted documentation.


## Start
To start automatically, enable gdm.service
```
$ systemctl enable gdm.service
$ reboot
# or switch to another tty and 
$ systemctl start gdm.service
```


## Network
```
$ pacman -S networkmanager
$ systemctl enable NetworkManager # beware of the cases
$ systemctl start NetworkManager
```

## Fonts
```
$ vim /etc/vconsole.conf
1> KEYMAP=de
2> FONT=lat9w-16
3> FONT_MAP=8859-1_to_uni
4> XKBLAYOUT=de
```

In case of utf-8 icons are not displayed propperly, 
  one of the following fonts could help
```
$pacman -S ttf-dejavu 
$pacman -S gnu-free-fonts
$pacman -S ttf-font-awesome
```

## fractional scaling
$ gsettings set org.gnome.mutter experimental-features "['scale-monitor-framebuffer']"
$ reboot

## open new windows in front
gsettings set org.gnome.desktop.wm.preferences auto-raise 'true'
gsettings set org.gnome.desktop.wm.preferences focus-new-windows 'smart'


## folder view
gsettings set org.gnome.nautilus.preferences default-folder-viewer 'list-view'
