# Copyright notice

```text
custom-2fa
Copyright 2025, Rolando Garza.
License GPLv3+: GNU GPL version 3 or later,
                <https://gnu.org/licenses/gpl.html>.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Written by Rolando Garza.
```


# What is this?

These Linux Pluggable Authentication Modules (Linux-PAM) configuration rules support using both "Universal 2nd Factor" (U2F) and "Time-based One-time Password" (TOTP) or "HMAC-based One-time Password" (HOTP) for "Two-Factor Authentication" (2FA) by using Google's `pam_google_authenticator.so` library and Yubico's `pam_u2f.so` library.


# Why did you write this?

Because I wanted to have a similar log-in workflow in my computers to the 2FA workflow that some online services provide:

-   Validate the user's password (part of `common-auth`).
-   Prompt for a U2F key if the user registered at least one.
-   If the user doesn't have their key or hasn't registered one, fallback to a TOTP/HTOP code from a TOTP/HTOP authenticator app.
-   If the user doesn't have both their key, nor the TOTP/HTOP app fallback to a rescue code (from the authenticator app).
-   Succeed if the user hasn't registered any 2FA methods.

This allows one to introduce other family members or co-workers to a 2FA workflow, and it helps system administrators to reduce the risk of account takeover by malicious actors because of weak passwords.


# Installation

Please make sure you have already met these two prerequisites:

-   having installed and configured [`pam_google_authenticator.so`](https://github.com/google/google-authenticator-libpam/)
-   having installed and configured [`pam_u2f.so`](https://github.com/Yubico/pam-u2f)

Let's, for example, add the `custom-2fa` rules to `/etc/pam.d/sudo`:

-   copy `custom-2fa` file into `/etc/pam.d/`:
    -   `sudo cp custom-2fa /etc/pam.d/`
-   edit (and don't close until we verify it works) `/etc/pam.d/sudo`:
    -   `sudo nano /etc/pam.d/sudo`
-   include `@include custom-2fa` after `@include common-auth` line, and save file without exiting the editor or closing the terminal emulator (see example below with included comments)
-   in another terminal emulator window or tab, verify installation:
    -   `sudo -k && sudo echo "Hello World!"`
    -   if you were able to successfully authenticate, you may close both windows
    -   if there was an error, see the Debugging section
    -   worst case scenario, you can comment the `@include custom-2fa` line to disable `custom-2fa` in the editor that still has root privileges that you totally didn't close.

You may probably want to add `custom-2fa` to the following rule files:

-   `/etc/pam.d/sudo`
-   `/etc/pam.d/gdm-password`
-   `/etc/pam.d/login`

Additionally, you may also want to place some comments so you remember how you configured your system. I use the following format (without `#` before `@include custom-2fa`):

```text
# custom-2fa
# - Prompts for FIDO2-capable U2F device
# - If present, request touch of device
# - Otherwise request a TOTP/HOTP validation code
# - Allow for users without 2FA to authenticate with password
# note: see /etc/pam.d/custom-2fa for implementation details
#@include custom-2fa
```


## WARNING!

Make sure you have several terminal emulators with root privileges open so that you can undo changes that would leave you without superuser access.


# Usage

Here is an example expected workflow in the terminal:

```console
user@computer:~$ sudo echo 'Hello, World!'
[sudo] password for user:
insert key and/or press ENTER:
touch key:
Hello, World!
```

This can also be set as a prompt when logging in (see Installation):

![img](./gdm3-login-screenshot.png "Example of 2FA prompt after password verification.")


# About


## Linux-PAM configuration rules

The files are made of lists of rules. Each rule is a space separated collection of tokens:

`service type control module-path module-arguments`

The files in `/etc/pam.d/` lack the "service" field. Some info regarding each field:

-   **service:** usually carry the name of the service or name of a familiar application, like "login" and "sudo".
-   **type:** the management group that the rule corresponds to
-   **control:** indicates behavior of PAM-API if module fails to authenticate. Two types of syntax are used: simple key word, and square-bracketed value=action pairs.
-   **module-path:** full filename (begins with "`/`") of the PAM to be used or relative path-name from default location (which could be either `/lib/security/`, `/lib64/security/`, or `/lib/x86_64-linux-gnu/security/`)


## These custom-2fa rules

These Linux-PAM configuration rules support using both U2F and TOTP/HOTP for 2FA by using `pam_google_authenticator.so` and `pam_u2f.so`.


### What is accomplished?

-   Prompt for a U2F device, and to then press ENTER.
    -   In case the U2F device is known, prompt the user to press the tactile trigger.
    -   In case the U2F device is not known or present, prompt for code for verification from TOTP/HOTP app (like Google Authenticator).
-   Allow users that are not configured to use U2F or TOTP/HOTP to log in.


# The rules, explained


## rule 1: request U2F key; press ENTER and detect key

-   management group type
    -   **`auth`:** module type that authenticates the user
-   control values
    -   **`default=ignore`:** the module's return status will not contribute to the return code the application obtains
    -   **`ignore=ignore`:** PAM module wants its result to be ignored
    -   **`new_authtok_reqd=ok`:** new authentication token is required
    -   **`success=1`:** Jump over the N modules in the stack on success
-   module path
    -   **`pam_u2f.so`:** use Yubico's `pam_u2f`
-   module arguments
    -   **`authfile=/etc/2fa/u2f/u2f_mappings`:** Sets the location of the file that holds the mappings of user names to keyHandles and user keys; should have `0600` permissions
    -   **`expand`:** Enables variable expansion within the authfile path: `%u` is expanded to the local user name (`PAM_USER`) and `%%` is expanded to `%`
    -   **`interactive`:** Set to prompt a message and wait before testing the presence of a FIDO device. Recommended if your device doesn't have a tactile trigger
    -   **`nouserok`:** Set to enable authentication attempts to succeed even if the user trying to authenticate is not found inside authfile or if authfile is missing/malformed
    -   **`openasuser`:** Setuid to the authenticating user when opening the authfile. Useful when the user's home is stored on an NFS volume mounted with the `root_squash` option.
    -   **`origin=pam://HOSTNAME`:** Set the relying party ID for the FIDO authentication procedure. If no value is specified, the identifier `pam://$HOSTNAME` is used.
    -   **`[prompt=insert key and/or press ENTER: ]`:** Specify the prompt to insert a U2F key and press ENTER; hint at TOTP option
    -   **`userpresence=0`:** If `1`, request user presence during authentication. If `0`, do not request user presence during authentication. Otherwise, fallback to the authenticator's default behavior.

```text
auth \
    [success=1 new_authtok_reqd=ok ignore=ignore default=ignore] \
        pam_u2f.so \
            authfile=/etc/2fa/u2f/%u/u2f_mappings \
            expand \
            interactive \
            nouserok \
            openasuser \
            origin=pam://HOSTNAME \
            [prompt=insert key and/or press ENTER: ] \
            userpresence=0
```


## rule 2: if no key was inserted, ask for TOTP token

-   management group type
    -   **`auth`:** module type that authenticates the user
-   control values
    -   **`default=bad`:** should be thought of as indicative of the module failing
    -   **`ignore=ignore`:** PAM module wants its result to be ignored
    -   **`new_authtok_reqd=ok`:** new authentication token is required
    -   **`success=1`:** Jump over the N modules in the stack on success
-   module path
    -   **`pam_google_authenticator.so`:** google-authenticator-libpam, by Google, will be used
-   module arguments
    -   **`[authtok_prompt=Type in token: ]`:** set token prompt
    -   **`nullok`:** OK if user doesn't have TOTP/HOTP 2FA rolled out
    -   **`secret=/etc/2fa/totp/${USER}/.totp_secrets`:** the nonstandard location for the file holding the secrets; it should have `0600` permissions

```text
auth \
    [success=1 new_authtok_reqd=ok ignore=ignore default=bad] \
        pam_google_authenticator.so \
            [authtok_prompt=type in token: ] \
            nullok \
            secret=/etc/2fa/totp/${USER}/.totp_secrets
```


## rule 3: if U2F key was inserted, request touch

-   management group type
    -   **`auth`:** module type that authenticates the user
-   control values
    -   **`required`:** failure of such a PAM will ultimately lead to the PAM-API returning failure but only after the remaining stacked modules (for this service and type) have been invoked. This is a shorthand for the following values:
        -   `[success=ok new_authtok_reqd=ok ignore=ignore default=bad]`
-   module path
    -   **`pam_u2f.so`:** use Yubico's `pam_u2f`
-   module arguments
    -   **`authfile=/etc/2fa/u2f/u2f_mappings`:** Sets the location of the file that holds the mappings of user names to keyHandles and user keys; should have `0600` permissions
    -   **`expand`:** Enables variable expansion within the authfile path: `%u` is expanded to the local user name (`PAM_USER`) and `%%` is expanded to `%`.
    -   **`cue`:** Set to prompt a message to remind to touch the device
    -   **`[cue_prompt=Touch key: ]`:** Specify prompt to touch key
    -   **`nouserok`:** Set to enable authentication attempts to succeed even if the user trying to authenticate is not found inside authfile or if authfile is missing/malformed.
    -   **`openasuser`:** Setuid to the authenticating user when opening the authfile. Useful when the user's home is stored on an NFS volume mounted with the `root_squash` option.
    -   **`origin=pam://HOSTNAME`:** Set the relying party ID for the FIDO authentication procedure. If no value is specified, the identifier `pam://$HOSTNAME` is used.
    -   **`userpresence=1`:** If `1`, request user presence during authentication. If `0`, do not request user presence during authentication. Otherwise, fallback to the authenticator's default behavior.

```text
auth \
    required \
        pam_u2f.so \
            authfile=/etc/2fa/u2f/%u/u2f_mappings \
            expand \
            cue \
            [cue_prompt=touch key: ] \
            nouserok \
            openasuser \
            origin=pam://HOSTNAME \
            userpresence=1
```


# Debugging and Troubleshooting

First, it may be useful to identify which files use custom-2fa: `grep -irlE "^#?@include custom-2fa" /etc/pam.d/ --exclude=custom`

After that, try to pinpoint if the problem is with `pam_u2f` (rules 1 and 3), or with `pam_google_authenticator` (rule 2). Then enable debug logging, try authenticating again, and inspect the output. Remember to escape previous line breaks with "`\`" when adding module arguments to the PAM rules.

For `pam_u2f` (rules 1 and 3):

-   Enable debug logging:
    -   `sudo touch /var/log/pam_u2f.log`
    -   add "`debug`" as an additional module argument
    -   optionally, also add "`debug_file=/var/log/pam_u2f.log`"
-   Try authenticating again:
    -   `sudo -k && sudo echo "Hello World!"`
-   Inspect debug logs:
    -   if `debug_file` was not specified, output will be on stderr
    -   if `debug_file` was specified:
        -   inspect it: `nano /var/log/pam_u2f.log`
        -   remove it: `sudo rm /var/log/pam_u2f.log`

For `pam_google_authenticator` (rule 2):

-   Enable debug logging
    -   add "`debug`" as an additional module argument
-   Begin monitoring syslog:
    -   open shell and monitor with: `tail -f /var/log/auth.log`
-   Try authenticating again:
    -   in another shell: `sudo -k && sudo echo "Hello World!"`


# Why did I include so much info and not just a README?

Because, honestly, sometimes you don't want to be scouring through the Internet to try to grok rules that you wrote several months ago. So I decided to include most of the README in the actual source as comments.


# TODO:

-   [X] Ask Yubico's pam-u2f developers if they could expand `%u` variable so that we could get something like: `authfile=/etc/2fa/u2f/%u/u2f_mappings`
    -   [X] see: <https://github.com/Yubico/pam-u2f/issues/218>
    -   [X] ~~contribute?~~
    -   [X] test
-   [X] Update README:
    -   [X] Document using newer pam-u2f library (with `%u`)
    -   [X] Document using `origin=pam://HOSTNAME`
-   [ ] Remove TODO section


# References

-   <http://www.linux-pam.org/Linux-PAM-html/sag-configuration.html>
-   <https://github.com/google/google-authenticator-libpam/>
-   <https://github.com/Yubico/pam-u2f>
-   <https://refspecs.linuxfoundation.org/fhs.shtml>
