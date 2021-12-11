## What is this?

These Linux Pluggable Authentication Modules (Linux-PAM) configuration
rules support using both "Universal 2nd Factor" (U2F) and "Time-based
One-time Password" (TOTP) or "HMAC-based One-time Password" (HOTP) for
"Two-Factor Authentication" (2FA) by using Google's
`pam_google_authenticator.so` library and Yubico's `pam_u2f.so`
library.

## Why did you write this?

Because I wanted to have a similar log-in workflow in my computers to
the 2FA workflow that some online services provide:

  - Validate the user's password (part of `common-auth`).
  - Prompt for a U2F key if the user registered at least one.
  - If the user doesn't have their key or hasn't registered one,
    fallback to a TOTP/HTOP code from a TOTP/HTOP authenticator app.
  - If the user doesn't have both their key, nor the TOTP/HTOP app
    fallback to a rescue code (from the authenticator app).
  - Succeed if the user hasn't registered any 2FA methods.

This allows one to introduce other family members or coworkers to a
2FA workflow, and it helps system administrators to reduce the risk of
account takeover by malicious actors because of weak passwords.

## Usage

``` bash
user@computer:~$ sudo echo "Hello, World!"
```

And the expected workflow in the terminal is:

``` text
[sudo] password for user: 
insert key and/or press ENTER: 
touch key: 
Hello, World!
```

This can also be set as a prompt when logging in (see Installation).

## Installation

After following installation procedures on [how to install pam<sub>googleauthenticator</sub>.so](https://github.com/google/google-authenticator-libpam/), and on [how to install pam<sub>u2f</sub>.so](https://github.com/Yubico/pam-u2f), just
include `@include custom-2fa` in one of your PAM rules after the line
that reads `@include common-auth`; these rules are usually located in
`/etc/pam.d/`. You may probably want to add them to the following
rules:

  - `/etc/pam.d/sudo`
  - `/etc/pam.d/gdm-password`
  - `/etc/pam.d/login`

You may also want to place some comments, so you remember how you
configured your system. I use the following format:

``` text
# custom-2fa
# - Prompts for FIDO2-capable U2F device
# - If present, request touch of device
# - Otherwise request a totp validation code
# - Allow for users without 2FA to authenticate with password
# note: see /etc/pam.d/custom-2fa for implementation details
@include custom-2fa
```

### WARNING\!

Make sure you have several terminal emulators with root privileges
open so that you can undo changes that would leave you without
access.

## About

### Linux-PAM configuration rules

The files are made of lists of rules. Each rule is a space separated
collection of tokens:

``` example
service type control module-path module-arguments
```

The files in `/etc/pam.d/` lack the "service" field. Some info
regarding each field:

  - service  
    usually carry the name of the service or name of a
    familiar application, like "login" and "sudo".
  - type  
    the management group that the rule corresponds to
  - control  
    indicates behaviour of PAM-API if module fails to
    authenticate. Two types of syntax are used. Simple key word, and
    square-bracketed value=action pairs.
  - module-path  
    full filename (begin with "/") of the PAM to be used
    or relative pathname from default location (which could be either
    `/lib/security/`, `/lib64/security/`, or
    `/lib/x86_64-linux-gnu/security/`)

### These custom-2fa rules

These Linux-PAM configuration rules support using both U2F and
TOTP/HOTP for 2FA by using `pam_google_authenticator.so` and
`pam_u2f.so`.

#### What is accomplished?

  - Prompt for a U2F device, and to then press ENTER.
      - In case the U2F device is known, prompt the user to press the
        tactile trigger.
      - In case the U2F device is not known or present, prompt for a
        verification code from Google Authenticator.
  - Allow users that are not configured to use U2F or TOTP/HOTP to log
    in.

#### rule 1: request U2F key; press ENTER and detect key

  - management group type
      - `auth`  
        module type that authenticates the user
  - control values
      - `default=ignore`  
        the module's return status will not contribute
        to the return code the application obtains
      - `ignore=ignore`  
        PAM module wants its result to be ignored
      - `new_authtok_reqd=ok`  
        new authentication token is required
      - `success=1`  
        Jump over the N modules in the stack on success
  - module path
      - `pam_u2f.so`  
        use Yubico's pam<sub>u2f</sub>
  - module arguments
      - `authfile=/etc/2fa/u2f/u2f_mappings`  
        Sets the location of the
        file that holds the mappings of user names to keyHandles and user
        keys; should have 0600 permissions
      - `interactive`  
        Set to prompt a message and wait before testing
        the presence of a FIDO device. Recommended if your device doesn't
        have a tactile trigger
      - `nouserok`  
        Set to enable authentication attempts to succeed
        even if the user trying to authenticate is not found inside
        authfile or if authfile is missing/malformed
      - `[prompt=insert key and/or press ENTER: ]`  
        Specify the prompt
        to insert a U2F key and press ENTER; hint at TOTP option
      - `userpresence=0`  
        If 1, request user presence during
        authentication. If 0, do not request user presence during
        authentication. Otherwise, fallback to the authenticator's
        default behaviour.

<!-- end list -->

``` text
auth \
    [success=1 new_authtok_reqd=ok ignore=ignore default=ignore] \
        pam_u2f.so \
            authfile=/etc/2fa/u2f/u2f_mappings \
            interactive \
            nouserok \
            [prompt=insert key and/or press ENTER: ] \
            userpresence=0
```

#### rule 2: if no key was inserted, ask for TOTP token

  - management group type
      - `auth`  
        module type that authenticates the user
  - control values
      - `default=bad`  
        should be thought of as indicative of the
        module failing
      - `ignore=ignore`  
        PAM module wants its result to be ignored
      - `new_authtok_reqd=ok`  
        new authentication token is required
      - `success=1`  
        Jump over the N modules in the stack on success
  - module path
      - `pam_google_authenticator.so`  
        google-authenticator-libpam,
        by Google, will be used
  - module arguments
      - `[authtok_prompt=Type in token: ]`  
        set token prompt
      - `nullok`  
        OK if user doesn't have TOTP/HOTP 2FA rolled out
      - `secret=/etc/2fa/totp/${USER}/.totp_secrets`  
        the non-standard
        location for the file holding the secrets; it should have 0600
        permissions

<!-- end list -->

``` text
auth \
    [success=1 new_authtok_reqd=ok ignore=ignore default=bad] \
        pam_google_authenticator.so \
            [authtok_prompt=type in token: ] \
            nullok \
            secret=/etc/2fa/totp/${USER}/.totp_secrets
```

#### rule 3: if U2F key was inserted, request touch

  - management group type
      - `auth`  
        module type that authenticates the user
  - control values
      - `required`  
        failure of such a PAM will ultimately lead to the
        PAM-API returning failure but only after the remaining stacked
        modules (for this service and type) have been invoked. This is a
        shorthand for the following values: \[success=ok
        new<sub>authtokreqd</sub>=ok ignore=ignore default=bad\]
  - module path
      - `pam_u2f.so`  
        use Yubico's pam<sub>u2f</sub>
  - module arguments
      - `authfile=/etc/2fa/u2f/u2f_mappings`  
        Sets the location of the
        file that holds the mappings of user names to keyHandles and user
        keys; should have 0600 permissions
      - `cue`  
        Set to prompt a message to remind to touch the device
      - `[cue_prompt=Touch key: ]`  
        Specify prompt to touch key
      - `nouserok`  
        Set to enable authentication attempts to succeed even
        if the user trying to authenticate is not found inside authfile or
        if authfile is missing/malformed.
      - `userpresence=1`  
        If 1, request user presence during
        authentication. If 0, do not request user presence during
        authentication. Otherwise, fallback to the authenticator's
        default behaviour.

<!-- end list -->

``` text
auth \
    required \
        pam_u2f.so \
            authfile=/etc/2fa/u2f/u2f_mappings \
            cue \
            [cue_prompt=touch key: ] \
            nouserok \
            userpresence=1
```

### Debugging & Troubleshooting

Find which files use custom-2fa:

  - `grep -irlE "^#?@include custom-2fa" /etc/pam.d/`

Enable debug logging:

  - if problems while logging in (rules 1 and 3):
      - `sudo touch /var/log/pam_u2f.log`
      - add "`debug_file=/var/log/pam_u2f.log`" as module arguments
  - inspect debug logs:
      - for pam<sub>u2f</sub> debugging (rules 1 and 3), output is on stderr:
          - `sudo -k && sudo echo "Hello World!"`
      - if debug<sub>file</sub> was specified:
          - inspect it, e.g.: `nano /var/log/pam_u2f.log`
          - remove it, e.g.: `sudo rm /var/log/pam_u2f.log`
      - for pam<sub>googleauthenticator</sub> (rule 2), output is on syslog:
          - open shell and monitor with: `tail -f /var/log/auth.log`
          - in another shell: `sudo -k && sudo echo "Hello World!"`
  - for all rules (1, 2, and 3):
      - add "`debug`" as an additional module argument
      - (remember to escape previous line breaks with " `\`")

## Why did I include so much info and not just a README?

Because, honestly, sometimes you don't want to be scouring through the
Internet to try to grok rules that you wrote several months ago.

## Copyright notice

custom-2fa  
Copyright 2021, Rolando Garza.  
License GPLv3+: GNU GPL version 3 or later,  
                <https://gnu.org/licenses/gpl.html>.  
This is free software: you are free to change and redistribute it.  
There is NO WARRANTY, to the extent permitted by law.  
  
Written by Rolando Garza.

## TODO:

  - \[-\] Ask Yubico's pam-u2f developers if they could expand ${USER}
    variable so that we could get something like:
    authfile=/etc/2fa/u2f/${USER}/u2f<sub>mappings</sub>
      - \[X\] see: <https://github.com/Yubico/pam-u2f/issues/218>
      - \[ \] contribute?

## References

  - <http://www.linux-pam.org/Linux-PAM-html/sag-configuration.html>
  - <https://github.com/google/google-authenticator-libpam/>
  - <https://github.com/Yubico/pam-u2f>
  - <https://refspecs.linuxfoundation.org/fhs.shtml>
