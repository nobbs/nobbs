---
draft: true
title: Use sudo with Touch ID in macOS Sonoma
summary: >
  For whatever reason, Touch ID approval for `sudo` commands is not enabled by default in macOS. Fortunately, it's easy to change that.
date: 2023-10-15T10:07:25+01:00
tags:
  - macos
  - sonoma
  - security
---

Enabling Touch ID for `sudo` commands hasn't changed much since at least Big Sur[^1] - with the introduction of macOS Sonoma, Apple made it even easier to make it a persistent setting.

Follow these steps, and you're done:

1. Switch into the `/etc/pam.d` directory - you will find a `sudo_local.template` file there.

   ```console
   $ cd /etc/pam.d
   $ ls -l sudo*
   .r--r--r-- 283 root 16 Sep 15:28 sudo
   .r--r--r-- 179 root 16 Sep 15:28 sudo_local.template
   ```

2. Copy the `sudo_local.template` file to a new file called `sudo_local` and uncomment the `auth sufficient pam_tid.so` line.

   ```console
   $ sudo cp sudo_local.template sudo_local
   $ sudo -e sudo_local
   ```

   It should look like this:

   ```shell
   # sudo_local: local config file which survives system update and is included for sudo
   # uncomment following line to enable Touch ID for sudo
   auth       sufficient     pam_tid.so
   ```

3. Save the file and exit the editor. **That's it.** From now on, you can use Touch ID to approve `sudo` commands. Create a new terminal session and give it a try.

[^1]: Based on this [StackExchange post](https://apple.stackexchange.com/questions/259093/can-touch-id-on-mac-authenticate-sudo-in-terminal).
