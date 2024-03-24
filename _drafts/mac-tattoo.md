---
layout: post
title: Mac Tattoo
date: 2024-03-23 20:24 -0600
---
# Mac Tattoo

This is my take on "tattooing the registry" from back in my SCCM/ConfigMgr days but for macOS.

If you manage any number of Macs in a corporate/enterprise environment, you 100% should have an MDM to manage Apple devices. Within those MDMs are varieties of information: Applications installed, hardware inventory, records of policies or actions run against the device, and most have a way for you to gather custom information.

While all that is useful, it's completely dependent on the device's inventory info being up-to-date when you're working with an End User to resolve an issue. And if any of you have ever been working with someone who is partially or fully disconnected, getting reliable information from them is challenging: Neither Microsoft or Apple have a built-in single-pane view showing said information.

All of this brings us to Part 1 of at least 2: Tattooing basic information locally onto your macOS fleet for use among a variety of ways:

* An End User launch-able app to display system and custom info
* Custom Extension Attributes for your MDM
* Leveraging tattooed values in other scripts
* Enrollment resilience

First, we need to explore not just the info we (you) want, but where to put it and how to get it.

# Which Info To Gather?

Everyone and every organization will have different needs, but in general, I work with the base philosophy that gather static (immutable) and semi-static data is the best place to start for your base script.

With macOS, especially for those of you coming from or used to Windows, it's important to understand that command outputs are pure string formats and nearly always need manipulation to isolate only the data you desire. An example is with `ioreg -l | grep IOPlatformSerialNumber` which gives an output like `"IOPlatformSerialNumber" = "XYZPDQ1234"`. In PowerShell, you would be able to use dot-notation or a Select operator or cmdlet to get just the serial number. In shell scripts, you have to parse the string using `awk` or `sed` and often remove the `"`, too. This makes it more challenging to get specific values, especially in multi-line outputs.

## Data Types

### Static Data

The way I define static data is fairly literal: Data that will not change or may only change if the Mac computer is erased and re-enrolled.

#### Examples

* Serial Number
* Model Number
* Processor Model
* Processor Architecture
* RAM Capacity
* Wi-Fi MAC address
* Enrollment user
* Enrollment date
* Management Account
* Warranty End Date

### Semi-Static Data

Semi-static data is subjective. For my use, the values can change, but it's very infrequent, if ever, and nearly always requires a restart before a the value is updated.

#### Examples

* ComputerName (The user-friendly name for the system.)
* HostName (The name associated with hostname(1) and gethostname(3).)
* LocalHostName (The local (Bonjour) host name.)
* NetBIOSName (Name seen by SMB shares.)
* FileVault status
* SecureToken users
* Volume Owners
* Admin users

### Dynamic Data

As the name suggests, Dynamic Data changes regularly or frequently, such as free disk space, IP address, logged in user account, connected SSID, VPN state, or AV status.

#### Examples

* Free disk space
* IP address
* Logged in user account
* Connected SSID
* VPN state
* VPN IP address
* AV status

# How To Gather the Data?

Without being too dismissive, Google is your friend for finding commands to gather the data you want to tattoo. But like I mentioned above with shell scripting, it's important to validate that the data value you want and the output of your command match. There's a big difference between expecting `10.10.10.1` yet your captured data is `" 10.10.10.1"` where the unneeded spaces and double-quotes can throw off regex checks, sorting, or date/time values.

To get you started, I'll go over a few of the examples mentioned earlier.

## Samples

Note: All samples are done using zsh (not bash or shell)

### Serial Number

For this first one, I'll break down each part of the command. First, let's go Jeopardy! style and start with the answer:

```sh
ioreg -c IOPlatformExpertDevice -d 2 | awk -F\" '/IOPlatformSerialNumber/{print $(NF-1)}'
XYZPDQ1234
```

Excerpt from the `man` page ([Great blog on man pages by Armin Briegel](https://scriptingosx.com/2017/04/on-viewing-man-pages/)):
>**ioreg** displays the I/O Kit registry.  It shows the hierarchical registry structure as an inverted tree.

First, we need to find possible values with the serial number.

```sh
ioreg -l | grep -i serialnumber
```

This command lists every possible value from `ioreg` and uses `grep -i` (`-i` is case a insensitive param) to filter for all lines containing the string "serialnumber". There are a lot of results:

![ioreg_grep-serialnumber](assets/img/ioreg_grep-serialnumber.png)

Note that the results are displayed in order but are not sequential, which means that Line 2 of this output is *not* Line 2 of the entire `ioreg -l` output.

Use the un-blurred values, you can see that Line 1 shows the desired value, so we want to adjust our command to look for `IOPlatformSerialNumber`

```sh
ioreg -l | grep IOPlatformSerialNumber
```

Many would pause here and just use this output to then parse out the serial number using `awk` and maybe `tr`. And you're not wrong, but we can make this more efficient and faster.

Let's add a parameter to `grep` so we can see the 16 (I cheated, obviously, by knowing the correct number of lines) lines preceding `IOPlatformSerialNumber`

```sh
ioreg -l | grep -B 16 IOPlatformSerialNumber
```

![ioreg_grep-b-16-ioplatformserialnumber](assets/img/ioreg_grep-b-16-ioplatformserialnumber.png)

Now we can see the C++ class that `IOPlatformSerialNumber` is in, which is `IOPlatformExpertDevice`. Being able to specify the class reduces the time needed to run the command.

Refer back to your man page on `ioreg` and modify the command:

```sh
ioreg -c IOPlatformExpertDevice
```

This output is way too long to use, so we'll add `-d 2` to control the depth of the output:

```sh
ioreg -c IOPlatformExpertDevice -d 2
```

![ioreg -c IOPlatformExpertDevice -d 2](assets/img/ioreg_-c-IOPlatformExpertDevice-d-2.png)
(Hey, this output looks familiar..)

Next, let's use a basic `awk` command to grab the line we want.

```sh
ioreg -c IOPlatformExpertDevice -d 2 | awk '/IOPlatformSerialNumber/{print}'
```

![alt text](assets/img/ioreg_-c-IOPlatformExpertDevice-d-2_awk_01.png)

Now we'll set the field separator to use `"` so it's easier to get the right column of output data and not have to use `tr` to remove the quotes. (This won't change the output from the previous command.)

```sh
ioreg -c IOPlatformExpertDevice -d 2 | awk -F\" '/IOPlatformSerialNumber/{print}'
```

Finally, we modify the `print` command within `awk` to select the last column. We can do this by using `{print $4}`, but a better option is to call the last column by using `{print $(NF-1)}` to be more dynamic in case preceding columns get changed in the future.

```sh
ioreg -c IOPlatformExpertDevice -d 2 | awk -F\" '/IOPlatformSerialNumber/{print $(NF-1)}'
```

![ioreg_-c-IOPlatformExpertDevice-d-2_awk_02-final](assets/img/ioreg_-c-IOPlatformExpertDevice-d-2_awk_02-final.png)

### Model Info

There are several model identifiers for Mac computers

Shout out to @PicoMitchell who wrote this great script where most of these were extracted from: [freegeek-pdx | get_specs_url_from_serial.sh](https://github.com/freegeek-pdx/macOS-Testing-and-Deployment-Scripts/blob/main/Other%20Scripts/get_specs_url_from_serial.sh)

#### Marketing Model Name

```sh
# This first option only works for Macs with Apple Silicon processors. Intel T2 and older models do not have the Marketing Model Name stored locally; you'll have to use a script
hkystar35 ~ % model_marketingName=$( /usr/libexec/PlistBuddy -c 'Print :0:product-name' /dev/stdin <<< "$(ioreg -arc IOPlatformDevice -k product-name)" 2> /dev/null | tr -d '[:cntrl:]' )
hkystar35 ~ % echo $model_marketingName                                                                      
MacBook Pro (14-inch, 2021)
```
For non-Apple Silicon Macs, check out this other script by Pico: [freegeek-pdx | get_specs_url_from_serial.sh](https://github.com/freegeek-pdx/macOS-Testing-and-Deployment-Scripts/blob/main/Other%20Scripts/get_specs_url_from_serial.sh)

#### Model ID

```sh
hkystar35 ~ % model_Id=$( sysctl -n hw.model 2> /dev/null )
hkystar35 ~ % echo $model_Id
MacBookPro18,3
```

#### Model ID Name

```sh
hkystar35 ~ % model_Id_name="${model_Id//[0-9,]/}"
hkystar35 ~ % echo $model_Id_name                          
MacBookPro
```

#### Model ID Number

```sh
hkystar35 ~ % model_Id_number="${model_Id//[^0-9,]/}"
hkystar35 ~ % echo $model_Id_number 
18,3
```

### Processor Model

```sh
hkystar35 ~ % processor_model=$( sysctl -n machdep.cpu.brand_string 2> /dev/null )
hkystar35 ~ % echo $processor_model 
Apple M1 Pro
```

### Processor Architecture

```sh
hkystar35 ~ % processor_arch=$( arch )
hkystar35 ~ % echo $processor_arch 
arm64
```

### RAM Capacity

```sh
hkystar35 ~ % memory_capacityBytes=$( sysctl -n hw.memsize )
hkystar35 ~ % echo $memory_capacityBytes 
17179869184
```

### Wi-Fi MAC address

```sh
hkystar35 ~ % network_WiFi_MacAddress=$( networksetup -getmacaddress Wi-Fi | awk '/Ethernet Address:/{print $3}' )

hkystar35 ~ % echo $network_WiFi_MacAddress 
a1:b2:c3:d4:e5:f6
```

## Scripting it

Once you've identified (*and tested*) the data you want to tattoo, it's time to put it all in a script.



```sh
#!/usr/bin/env zsh

# Placeholder


```

# How Often To Gather the Data?

# Where To Store the Data?

# How To Access the Stored Data?

