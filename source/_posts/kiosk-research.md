---
title: Kiosk Research
date: 2019-12-05 18:29:13
tags: 
- 工作
- Linux
- 研究
excerpt: share some kiosk solution
---

### 1. Requirements of Kiosk
- basic
    - limited usage
        - only ourself desktop application
    - system management
        - wifi config
        - file share
        - network config
    - platform
        - linux: best
        - windows: not bad
        - android/ios: none
- advance
    - remote control
    - report & alert
        - battery level alert: Get device battery percentage (%) alerts for the selected time range of a day.
        - location report: Location history reports for each device or group of devices.
        - data usage alert: Receive a notification when the device exceeds the data limit set by you.

### 2. Kiosk Solution
#### 2.1 commerce(windows)
- [scalefusion windows 10 kiosk mode](https://scalefusion.com/windows-10-kiosk-mode)
    - custom branding
    - configure wifi setting
    - app blacklisting
    - location tracking
    - restrict peripherals
    - mail configuration
    - enforce passcode policy
    - edge brower management
- [kioware for windows](https://www.kioware.com/windows.aspx)
    - keyboard filtering
    - clears user data
    - file download blocking
    - device orgnazition
    - content management
    - remote monitoring
    - device usage statistics
    - reporting
- [other windows solution](http://techgenix.com/windows-kiosk-mode-solutions/)

#### 2.2 [open source (linux)](http://tuxdiary.com/2014/11/05/linux-distros-for-kiosks/)
- [Porteus kiosk](http://porteus-kiosk.org/)

    The mini distro Porteus has a dedicated mod to support kiosk mode. Provides options to customize as need on first boot.Porteus Kiosk is a free to use, lightweight Linux operating system which has been restricted to allow only use of the web browser.
    
- [Ubuntu Guest login](http://www.ubuntu.com/)
    
    Normal Ubuntu installation allows guest access from the login prompt. A perfect solution for kiosks. Login into guest mode with restricted permissions but non-restricted browser. Works from USB too.

- Scientific Linux 6 and CentOS 6

    Both the distros can be configured into kiosk-mode automatically by running this [script](http://www.marcinwilk.eu/sl/make-kiosk.sh).
    
- Kiosk conversion script
    
    Redditor rawfan shared his script ([create_kiosk.sh](https://gist.github.com/5bba6c9b6425a42b4ea1)) to convert Ubuntu into a kiosk. It is untested and should not be run directly on your workstation as it modifies lightdm behaviour. Test first. It does the following:
    - create a kiosk user with limited rights
    - the user gets an empty password (mkpasswd ”) with a password-policy that it can’t be changed
    - mount a union file system on top of the kiosk users $HOME (aufs) so that the user can’t actually change anything
    - wipe the aufs on login, logout and boot
    - display a warning on login that users need to save their stuff to USB keys because everything gets wiped clean
    - log the user out after a couple of minutes of inactivity
    - lightdm only displays the kiosk user and the option to enter a custom username and a message to press enter to log in
    - set up unattended upgrades for all packages

    If you test it out, don’t forget to leave a feedback on how the script performs.
- [Ubuntu Kiosk](https://tutorials.ubuntu.com/tutorial/secure-ubuntu-kiosk#0)

#### 2.3 other article
- [other linux kiosk distros and browsers tools](https://www.how2shout.com/tools/free-open-source-linux-kiosk-distros-browsers.html)

### 3. Ubuntu kiosk
take a deep research for below targets:
- how to work
- how to setup
- how to deploy
- does it support basic system config
- does it support for multiple application

#### 3.1 basic knowledge
We use Wayland as the primary display interface. We will use Mir to manage the display and support connections from Wayland clients. Snapd will confine the applications and enable Wayland protocol interactions through Mir, securely.

Your graphics approach largely depends on the toolkit your application uses:

- GTK3/4 and Qt5 - have native support for Wayland
This is the simplest case, as the application can talk Wayland to Mir directly.

- GTK2, Qt4, Java - do not have Wayland support
This is a more complex case, as the toolkits require a legacy X11 server to function. To enable these applications we will embed a tiny X11 server into your application package, which translates X11 calls to Wayland ones.

- Electron, HTML5, Chromium - do not have Wayland support
We will need to use the embedded browser together with a tiny embedded X11 server to handle the translation.

![kiosk-structure](kiosk-structure.png)

If our descktop app base on electron, so the third option should be choosed.

#### 3.2 [make a HTML/Electron-based kiosk snap](https://tutorials.ubuntu.com/tutorial/electron-kiosk#0)
- prepare
    - install mir
    ```
    sudo snap install --devmode mir-kiosk -y
    sudo mir-kiosk
    ```
    - install snacraft
    ```
    sudo apt install snapcraft -y
    ```
    - install node
    ```
    # install nvm
    wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.1/install.sh | bash
    # enable nvm command
    command -v nvm
    # install nodejs(the version should support the async/await)
    nvm install v12.13.1
    ```
    - build & install & run
    ```
    # 1. create a test workspace
    mkdir demo && cd demo
    # 2. init a project
    snapcraft init
    # 3. edit the snapcraft.yaml use the demo application config: https://github.com/nnsay/blog/wiki/Ubuntu-Kiosk-Electron-Snap-Config
    gedit snap/snapcraft.yaml
    # 4. build
    snapcraft
    # 5. install
    sudo snap install --dangerous ./electron-hello-world-kiosk_0.1_amd64.snap
    # 6. connect to brower support
    sudo snap connect electron-hello-world-kiosk:wayland
    # 7. run(optional)
    sudo snap restart electron-hello-world-kiosk
    ```
    the app will show in the `Mir On X`:
    ![kiosk-result](kiosk-result.png)
    
#### 3.3 summary
- ubuntu kiosk use wayland and mir to guarantee the security
- ubuntu kiosk is not a good way for system config
- no found the information about how to implement multiple application within kiosk

### 4. QA
#### 4.1  how to set the version of snapcraft nodejs plugin?
check [here](https://github.com/search?q=filename%3Asnapcraft.yaml+%22plugin%3A+nodejs%22&type=Code), set the `node-engine`
```
parts:
  hmon:
    plugin: nodejs
    node-engine: 12.13.1
```