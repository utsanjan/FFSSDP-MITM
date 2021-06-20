# Firefox for Android LAN-Based Intent Triggering
 
*Exploit research and development by Chris Moberly (Twitter: [@init_string](https://twitter.com/init_string))*
<br>*Recreated by Utsanjan Maity - aka DopeSatan (Twitter: [@utsanjan](https://twitter.com/utsanjan))*

## 👁️‍🗨️ Overview 

The SSDP engine in Firefox for Android (68.11.0 and below) can be tricked into triggering Android intent URIs with zero user interaction.  This attack can be leveraged by attackers on the same WiFi network and manifests as applications on the target device suddenly launching, without the users' permission, and conducting activities allowed by the intent.

The target simply has to have the Firefox application running on their phone. They do not need to access any malicious websites or click any malicious links. No attacker-in-the-middle or malicious app installation is required. They can simply be sipping coffee while on a cafe's WiFi, and their device will start launching application URIs under the attacker's control.

I discovered this bug while the newest version of Firefox Mobile v79 was being rolled out globally. Google Play Store was still serving a vulnerable version at this time, but only for a short period. I reported the issue [directly to Mozilla](https://bugzilla.mozilla.org/show_bug.cgi?id=1659381), just to be safe. They responded right away and were quite pleasant to work with, providing some good info on where exactly this bug came from. They were able to confirm that the vulnerable functionality was not included in the newest version and opened some issues to ensure that the offending code was not re-introduced at a later time.

If you find a Firefox bug, I definitely recommend sending it straight to them. The process is very easy, the team members are smart and friendly, and it's a good way to support a project that has helped shape the way we use the web.

As long as you have app updates enabled and have recently connected to WiFi, you should have received the new version and are safe from exploitation. You can verify this yourself by opening Firefox on your device, clicking the three dots next to the address bar, and navigating to "Settings -> About Firefox". If your version is 79 or above, you are safe.

This write-up is specifically about the Android mobile application - the desktop application does not have this vulnerability.

🔗 [Click here to download the Video Demo of this Script](https://raw.githubusercontent.com/utsanjan/FFSSDP-MITM/main/poc.mp4)
<br>🔗 [Click here for Alternate Video Demo Link (If the above link doesn't work)](https://raw.githubusercontent.com/utsanjan/FFSSDP-MITM/main/poc2.mp4)

## ⚙ Technical Details

The vulnerable Firefox version periodically sends out SSDP discovery messages, looking for second-screen devices to cast to (such as the Roku). These messages are sent via UDP multicast to 239.255.255.250, meaning any device on the same network can see them. If you run Wireshark on your LAN, you will probably see something on your network doing the same. A discovery message from Firefox looks like this:

```
M-SEARCH * HTTP/1.1
Host: 239.255.255.250:1900
ST: roku:ecp
Man: "ssdp:discover"
MX: 3
```

Any device on the local network can respond to these broadcasts and provide a location to obtain detailed information on a UPnP device. Firefox will then attempt to access that location, expecting to find an XML file conforming to the UPnP specifications.

This is where the vulnerability comes in. Instead of providing the location of an XML file describing a UPnP device, an attacker can run a malicious SSDP server that responds with a specially crafted message pointing to an [Android intent URI](https://developer.android.com/training/basics/intents/sending). Then, that intent will be invoked by the Firefox application itself.

For example, responding with a message like the following would force any Android phones on the local network with Firefox running to suddenly launch a browser to `http://example.com`:

```
HTTP/1.1 200 OK
CACHE-CONTROL: max-age=1800
DATE: Tue, 16 Oct 2018 20:17:12 GMT
EXT:
LOCATION: intent://example.com/#Intent;scheme=http;package=org.mozilla.firefox;end
OPT: "http://schemas.upnp.org/upnp/1/0/"; ns=01
01-NLS: uuid:7f7cc7e1-b631-86f0-ebb2-3f4504b58f5c
SERVER: UPnP/1.0
ST: roku:ecp
USN: uuid:7f7cc7e1-b631-86f0-ebb2-3f4504b58f5c::upnp:rootdevice
BOOTID.UPNP.ORG: 0
CONFIGID.UPNP.ORG: 1
```

## 📝 Proof of Concept

If you'd like to play around with the bug yourself, you can grab an older version of Firefox for Android [here](https://archive.mozilla.org/pub/mobile/releases/68.11.0/).

I've spent a bit of time developing attack POCs for SSDP vulnerabilities in other applications, using a tool I wrote called [evil-ssdp](https://github.com/initstring/evil-ssdp). I created a modified version of that tool specifically to demonstrate this Firefox vulnerability. It's attached to this repository as [ffssdp.py](ffssdp.py).

We'll start with forcing all phones on the LAN to pop up a web browser to https://evil.com/.

First, just open Firefox on your Android device and let it sit there.

Next, run the exploit on a Linux laptop that is connected to the same wireless network as your Android device. The Android emulator will work, as well. Disable the firewall on your laptop while testing, or at least permit UDP broadcasts to be received.

```
# Replace "wlan0" with the wireless device on your attacking machine.
python3 ./ffssdp.py wlan0 -t "intent://example.com/#Intent;scheme=http;package=org.mozilla.firefox;end"
```

Firefox on the mobile device should go to http://example.com within a few seconds, and you'll see some logging in the attack tool as well.

Another example is to call other applications. Running the attack tool like this will pop the mail application with arbitrary text. Pretty scary to have happen on your device when you're just minding your own business:

```
# Replace "wlan0" with the wireless device on your attacking machine.
python3 ./ffssdp.py wlps0 -t "mailto:itpeeps@work.com?subject=I've%20been%20hacked&body=OH%20NOES!!!"
```

And one more, just for testing purposes. This will just pop the dialer:

```
# Replace "wlan0" with the wireless device on your attacking machine.
python3 ./ffssdp.py wlan0 -t "tel://1337h825012"
```

## 💀 Impact

This is not some super fancy memory-corruption bug that can be invoked from across the planet. It's a pretty straight-forward logic bug that basically allows you to magically click links on other peoples' phones who are in the same building as you.

The vulnerability resembles RCE (remote command execution) in that a remote (on same WiFi network) attacker can trigger the device to perform unauthorized functions with zero interaction from the end user. However, that execution is not totally arbitrary in that it can only call predefined application intents.

Had it been used in the wild, it could have targeted known-vulnerable intents in other applications. Or, it could have been used in a way similar to phishing attacks where a malicious site is forced onto the target without their knowledge in the hopes they would enter some sensitive info or agree to install a malicious application. The exploit POC can direct-link to a `.xpi` file, prompting for immediate installation of a malicious extention to compromise the browser itself.

The POC code is persistent, in that it will trigger the intent over and over until stopped. This increases the chances of someone agreeing to install a malicious package as the prompt will pop up over and over until the attacker stops running the tool.

With mobile apps, it is possible that many people remain on outdated versions for an extended period of time. This is due to the default setting of applications updating only when connected to WiFi, and the fact that some may only rarely (or never) connect to a WiFi network. Fortunately, this bug is exploitable only over WiFi, so those that cannot connect to update can also not be targeted. There is also a workaround for those who cannot update whatever reason, and that is to set `browser.casting.enabled` to `false` in `about:config`.

As a final thought, this most definitely could have been an epic rick roll,
<br>where everyone in the room running Firefox tried to figure out what the heck was going on.

## 🌎 Contact me  

For Queries: [My Instagram Profile](https://www.instagram.com/utsanjan/)  
[Check Out My YouTube Channel](https://www.youtube.com/DopeSatan)
