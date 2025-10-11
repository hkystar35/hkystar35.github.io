---
layout: post
title: Extension Attribute Script Templates
tags: MDM Jamf Extension-Attributes Zsh Scripting
date: 2025-10-11 14:54 -0600
---

This will be a quick post: After years of writing various Jamf Extension Attributes (Or whatever your MDM flavor calls gathering custom info from your Mac fleet), I finally decided to make a few templates of the most common types of data I typically need to gather.

## Common Attributes

There are a ton of varying data points to inventory outside of your MDM's default data. Some apps have different versions (looking at you, Microsoft), some don't have an app and need info gathered from a CLI (Homebrew stats), and other times you're tracking another custom solution for your environment (perhaps a complex migration script to move Macs to a new VPN solution) in a home-made `.plist`.

In my experience, these are the 3 most common data types to inventory:

1. Property value of a `.plist`
2. If a file exists
3. App installation source (Mac App Store (user/VPP) or CDN)

## Creating Templates

Why create templates? As with most things, it's just easier. Creating and using templates ensure a consistent output, format, shell choice, and can be a great tool when you have a larger team so that everyone uses the same templates.

Here's what goes into mine:

1. Script Naming convention
   * I like to copy PowerShell's verb-noun scheme as it's easy to read
2. A consistent shebang and shell choice
   * Using `#!/usr/bin/env zsh` (as opposed to `#!/bin/zsh`) for portability
      * [Dave Eddy - ysap.sh - Bash style guide](https://style.ysap.sh/)
      * [nixCraft - Make Linux/Unix Script Portable...](https://www.cyberciti.biz/tips/finding-bash-perl-python-portably-using-env.html)
3. Variables
   * At the top
      * Easy to see and modify
   * Single quotes for static values
      * or double-quotes when using variables, like a user's home folder
   * A default value for the end result

## Example Templates

Let's go through the 3 I use most.

### Property value of a `.plist`

This is a simple plug-and-play to output the value of a plist property. I use `PlistBuddy` instead of `defaults` because it's easier to grab nested properties and because at some point `defaults` is supposed to be deprecated.

This Extension Attribute simply gets the installed version of Google Chrome

```sh
#!/usr/bin/env zsh

plist='/Applications/Google Chrome.app/Contents/Info.plist'
plistKey='CFBundleShortVersionString'
RESULT='Not Installed'

if [[ -n "${plistKey}" && -n "${plist}" ]]; then
    if value="$(/usr/libexec/PlistBuddy -c "Print :${plistKey}" "${plist}")"; then
        RESULT="${value}"
    fi
fi

/bin/echo "<result>${RESULT}</result>"

exit 0

```

And here is a sample within the logged in user's Home folder to output a boolean value if the user's Apple Account is Managed or not.

> Notice that we're not assuming the user's Home folder is in `/Users/`; instead, we're reading from `dscl` since often times MDM created admin accounts' Home folders are in `/private/var/`

```sh
#!/usr/bin/env zsh

loggedInUser="$( /bin/echo "show State:/Users/ConsoleUser" | /usr/sbin/scutil | /usr/bin/awk '/Name :/&&!/loginwindow/{print $3}' )"
loggedInUserHomeFolder="$(/usr/bin/dscl . -read /users/$loggedInUser NFSHomeDirectory | cut -d " " -f 2)"
plist="$loggedInUserHomeFolder/Library/Preferences/MobileMeAccounts.plist"
plistKey='Accounts:0:isManagedAppleID'
RESULT='false'

if [[ -n "${plistKey}" && -n "${plist}" ]]; then
    if value="$(/usr/libexec/PlistBuddy -c "Print :${plistKey}" "${plist}")"; then
        RESULT="${value}"
    fi
fi

/bin/echo "<result>${RESULT}</result>"

exit 0

```

### If a file exists

This sample is very simple and can be used for a variety of things, like checking for `/var/db/.AppleSetupDone` for post-enrollment actions (or in this example, `.JamfSetupStarted` for use with Jamf onboarding tools).

```sh
#!/usr/bin/env zsh

file='/private/var/db/.JamfSetupStarted'
RESULT='FALSE'

if [[ -f "${file}" ]]; then
    RESULT='TRUE'
fi

/bin/echo "<result>${RESULT}</result>"

exit 0

```

### App installation source (Mac App Store (user/VPP) or CDN)

Finally, this is one that I've found very useful, especially for Microsoft apps to ensure users are on our org's desired update channels. (I scope a conversion Policy to Macs with non-CDN apps installed.)

```sh
#!/usr/bin/env zsh

# EA to determine App installation source

appPath="/Applications/Microsoft Outlook.app"
RESULT='Not Installed'

if [[ -d "${appPath}" ]]; then
    case "$(/usr/bin/mdls -name kMDItemAppStoreReceiptType "${appPath}" | awk '{print $3}')" in
        '"Production"')
            RESULT='MAS user'
            ;;
        '"ProductionVPP"')
            RESULT='MAS VPP'
            ;;
        *)
            RESULT='CDN'
            ;;
    esac
fi

echo "<result>${RESULT}</result>"

exit 0

```

## Parting Thoughts

It's easy to sit behind my keyboard and shout about how 'eVErYOnE SHoulD DO tHIs' because I think my way is best (it's not). As I age, it's more and more common for me to start a complex scripting journey only to pause after a while and think to myself, "Does it really need to be this involved?" And too often, it does not. Save your sanity and follow [K.I.S.S.](https://en.wikipedia.org/wiki/KISS_principle) when you can. Cheers.
