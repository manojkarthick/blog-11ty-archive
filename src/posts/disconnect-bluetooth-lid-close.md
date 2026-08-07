---
id: "post/13"
title: "Automatically disconnect bluetooth headphones when your MacBook goes to sleep"
date: 2022-01-30 18:00:00
tags:
- bluetooth
- hammerspoon
- automation
desc: "Use hammerspoon to watch for system sleep events and automatically disconnect configured bluetooth headphones using the bluetoothconnector package and a custom bash script"
url: https://manojkarthick.netlify.app
img: https://manojkarthick.netlify.app/img/sony-wh-1000xm4.jpg
img_alt: "Sony WH-1000XM4 Headphones"
type: summary_large_image
twitter: manojkarthick
name: Manoj Karthick
---

<!-- Excerpt Start -->
My current favorite headphones are the [Sony-WH1000XM4][1]s. They are light, have great noise cancellation and tremendous battery life. I use them on both my personal and work MacBook.

But I have two minor annoyances with how they behave. This is not specific to these but bluetooth headphones in general.

* When the headphones are turned off without disconnecting, they connect to the machine they were previously connected to when they are turned back on.
* If I close my MacBook without disconnecting, they will stay connected. If I want to switch the headphones to my other MacBook they will have to be manually disconnected first.

The solution I came up with is to disconnect my headphones when the MacBook goes to sleep. To avoid constant disconnects, you can adjust your Mac's [sleep/wake configuration][9] or use an app like [Amphetamine][2] to keep your Mac awake.

![Sony WH1000XM4](/img/sony-wh-1000xm4.jpg)

<!-- Excerpt End -->

Thankfully, [Hammerspoon][3] provides an easy way to hook into system's sleep events. If you're unfamiliar with Hammerspoon, it's a macOS based automation tool that allows you interact with various parts of the operating system using a Lua based scripting environment.

Let's look at my Hammerspoon configuration to see how it can be done:

```lua
-- wrapper function for shell commands
function shell(command)
    hs.execute(command .. ' &', true)
end

-- add bluetooth caffeine integration
function disconnectHeadphonesOnSleep(eventType)
    if eventType == hs.caffeinate.watcher.systemWillSleep then
        shell("~/bin/headphones disconnect sony-wh-1000-xm4")
    end
end
headphonesWatcher = hs.caffeinate.watcher.new(disconnectHeadphonesOnSleep)
headphonesWatcher:start()

```

First, we create a new function called `disconnectHeadphonesOnSleep` that accepts an event type. If the type of the event is `systemWillSleep`, then the `headphones disconnect <...>` command is executed.

macOS provides the [caffienate][4] system/command to prevent the system from going to sleep. We'll use this inbuilt framework to listen for relevant system events by registering and starting a watcher. The watcher is started when Hammerspoon is first launched.

`headphones` is simple shell script that disconnects the headphones with the given name. It uses the [BluetoothConnector][5] command line tool to get the status, connect, disconnect or toggle bluetooth devices using their [MAC address][6]. I have made the script available as a [Github Gist][7], so feel free to skip the rest of the blog post if you would rather look at the script directly.

Let's try to understand how the script works by looking at the relevant parts.

```shell
#!/usr/bin/env bash

SONY_ID="sony-wh-1000-xm4"
SONY_MAC="f8-4e-17-38-53-18"
PLT_ID="plantronics-bb-pro-2"
PLT_MAC="bc-f2-92-d4-11-25"

declare -A headphones
headphones["$SONY_ID"]="$SONY_MAC"
headphones["$PLT_ID"]="$PLT_MAC"
```

Here, we create associative array that maps a human readable headphone name to it's MAC address. You can find the MAC addresses you are interesting in by running the `bluetoothconnector --status` command. This command shows a list of all the bluetooth devices paired to your MacBook and their respective MAC address. You can install [BluetoothConnector][5] using [Homebrew][8] by running `brew install bluetoothconnector`. Remember to redefine the array based on your headphones' name and MAC address.

```shell
TARGET_MAC="${headphones[$2]}"
OUTPUT="$($CMD --status "$TARGET_MAC")"

case $1 in
status)
  if [[ "$OUTPUT" =~ "Disconnected" ]]; then
    echo "$2 is not connected"
  else
    echo "$2 is connected"
  fi
  ;;
toggle)
  if [[ "$OUTPUT" =~ "Disconnected" ]]; then
    echo "Connecting $2 with mac address $TARGET_MAC."
    $CMD --connect "$TARGET_MAC" --notify >/dev/null 2>&1 &

  else
    echo "Disconnecting $2 with mac address $TARGET_MAC."
    $CMD --disconnect "$TARGET_MAC" --notify >/dev/null 2>&1 &
  fi
  ;;
disconnect)
  if [[ "$OUTPUT" =~ "Disconnected" ]]; then
    echo "Nothing to do."
  else
    echo "Disconnecting $2 with mac address $TARGET_MAC."
    $CMD --disconnect "$TARGET_MAC" --notify >/dev/null 2>&1 &
  fi
  ;;
connect)
  if [[ "$OUTPUT" =~ "Disconnected" ]]; then
    echo "Connecting $2 with mac address $TARGET_MAC."
    $CMD --connect "$TARGET_MAC" --notify >/dev/null 2>&1 &
  else
    echo "Nothing to do."
  fi
  ;;
esac
```

In the above snippet of code, we define four subcommands, `status`, `toggle`, `disconnect` and `connect`. We identify the MAC address for the headphones from the previously defined array and pass them to the `bluetoothconnector` CLI based on the subcommand. If the `toggle` subcommand is chosen, we check whether the headphones are already connected and disconnect if so, or vice versa.

From now onwards, everytime the MacBook goes to sleep the headphones are disconnected automatically and you're free to turn them off or switch to a different MacBook seamlessly. One simple note if it wasn't already obvious: make sure the Hammerspoon app, the headphones script and the watcher are configured on all Macs you want this functionality on.

Cheers and have fun automating!

[1]: https://www.sony.com/lr/electronics/headband-headphones/wh-1000xm4
[2]: https://roaringapps.com/app/amphetamine
[3]: https://www.hammerspoon.org/
[4]: https://ss64.com/osx/caffeinate.html
[5]: https://github.com/lapfelix/BluetoothConnector
[6]: https://en.wikipedia.org/wiki/MAC_address
[7]: https://gist.github.com/manojkarthick/289ab431727fa367c4c04ba66e320fb0
[8]: https://brew.sh/
[9]: https://support.apple.com/en-ca/guide/mac-help/mchle41a6ccd/mac
