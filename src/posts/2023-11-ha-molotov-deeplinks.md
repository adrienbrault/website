---
title: Using Alexa to change the Molotov.tv channel
date: 2023-11-22
---

We use [Molotov.tv](https://www.molotov.tv) on our Nvidia Shield Android TV to watch Live TV and program replays.
The application home screen isn't a list of channels, but a list of programs. The list of channels to watch live TV are in a separate menu.
The sequence of steps to turn on the TV, wait for it to start, open the Molotov.tv application, open the menu, select the channel, and wait for the channel to start is quite long.
Ideally I would just ask Alexa to change the channel. I haven't found way to control Molotov.tv on the Android TV with Alexa.

I've recently started using the [androidtv_remote](https://www.home-assistant.io/integrations/androidtv_remote/) integration with my Shield.
With this integration it is possible to launch applications from Home Assistant. It requires knowing the "Deep Link" url format of the application. For example the Netflix application is known to support `https://www.netflix.com/title` and `netflix://`.

I found [Android TV Remote - App Links/Deep Linking - Guide](https://community.home-assistant.io/t/android-tv-remote-app-links-deep-linking-guide/567921) on the HA community forum:

> This is a guide for how to find out URLs to pass to the Android TV Remote - Home Assistant 449 integration for opening apps. It’s also an attempt to collect URLs for the most common apps.

While the post has many applications deep links documented, unfortunately Molotov.tv isn't one of them.
The post suggests looking into the application AndroidManifest.xml.

I looked for `molotov.tv android tv apk download` online and downloaded the apk.
The apk is a zip file, so I extracted it and found the AndroidManifest.xml file.

```bash
$ mkdir molotov-playground
$ cd molotov-playground
$ wget https://.../molotov.apk
$ unzip molotov.apk
```

However I quickly realize that the AndroidManifest.xml file is not a text file, but a binary file:

```bash
$ file AndroidManifest.xml
AndroidManifest.xml: Android binary XML
```

[github.com/ytsutano/axmldec](https://github.com/ytsutano/axmldec) is able to convert the binary file to a text file, and is easy installable on Mac with Homebrew:

```bash
$ brew tap ytsutano/toolbox
$ brew install axmldec

$ grep --color scheme ./AndroidManifest-text.xml
      <data android:scheme="https"/>
        <data android:scheme="molotov" android:host="www.molotov.tv" android:path="/deeplink"/>
        <data android:scheme="molotov" android:host="deeplink"/>
        <data android:scheme="tv.molotov.app"/>
        <data android:scheme="http" android:host="www.molotov.tv" android:pathPrefix="/deeplink"/>
        <data android:scheme="https" android:host="www.molotov.tv" android:pathPrefix="/deeplink"/>
        <data android:scheme="type1/2132017807"/>
        <data android:scheme="fbconnect" android:host="cct.tv.molotov.app"/>
```

So the following are valid deeplinks:

- `molotov://www.molotov.tv/deeplink`
- `molotov://deeplink`
- `https://www.molotov.tv/deeplink`
- `fbconnect://cct.tv.molotov.app`

I tried to use the web app channels urls paths with these links, but it only ever opened the application, not the intended chnanel.

About to give up I tried to grep deeplink:

```
$ grep deeplink -r -C 3 .

./assets/live-channel-fetch-premium-user.json-    "label": "M6",
./assets/live-channel-fetch-premium-user.json-    "displayNumber": 5,
./assets/live-channel-fetch-premium-user.json-    "poster": "https://orig-fusion.molotov.tv/arts/i/1200x1200/Ch0SGwoUV3hY19vP4cnVk6T8szFlelzF74YSA3BuZw/png",
./assets/live-channel-fetch-premium-user.json:    "deeplink": "molotov://deeplink?id=play\u0026source=amazon-livetv\u0026type=action\u0026video_id=45\u0026video_type=channel",
./assets/live-channel-fetch-premium-user.json-    "programs": [{
./assets/live-channel-fetch-premium-user.json-      "title": "M6 Boutique",
./assets/live-channel-fetch-premium-user.json-      "description": "Pour tout savoir des dernières nouveautés en matière de beauté, d'appareils électroménagers ou d'objets en tout genre pour meubler, décorer ou bricoler...",
--
./assets/live-channel-fetch-premium-user.json-    "label": "Arte",
```

And found `./assets/live-channel-fetch-premium-user.json` which contains the deeplinks for all the channels. Seeing `source=amazon-livetv` in the URLs points to them being used for some amazon device integration.

I found that those URLs worked! Here is the HA service call that starts molotov.tv and changes the channel to TF1:

```yaml
service: media_player.play_media
data:
  media_content_type: url
  media_content_id: >-
    molotov://deeplink?id=play&source=amazon-livetv&type=action&video_id=46&video_type=channel
target:
  entity_id: media_player.shield_remote
```

I created a script using that service call, and added the script to my [cloud configuration](https://www.nabucasa.com/config/amazon_alexa/):

```yaml
cloud:
  alexa:
    filter:
      include_entities:
        - script.shield_tf1
  entity_config:
    script.shield_tf1:
      description: TF1 sur le shield / TV dans le Salon
      display_categories: TV
```

I changed the display category to TV because I had issues triggering the script from Alexa.

Now alexa understood `allume TF1 sur le shield` / `turn on TF1 on the shield` and my script was triggered, starting the molotov.tv app and opening the TF1 live tv channel.

However, when the Shield was off, the service call would turn it on, but the molotov app would not start.

So I updated the script to turn the shield if off, and wait for it to be on before opening the deep link:

```yaml
- alias: Turn on Shield if off
  choose:
    - conditions:
        - condition: state
          entity_id: media_player.shield
          state: "off"
      sequence:
        - service: media_player.turn_on
          target:
            entity_id: media_player.shield
        - wait_for_trigger:
            - platform: state
              entity_id:
                - media_player.shield
              to: "on"
              for:
                seconds: 2
          timeout:
            seconds: 30
- service: media_player.play_media
  # ...
```

The easiest way I've found so far to have scripts for all channels I care about is to create a [blueprint](https://www.home-assistant.io/docs/automation/using_blueprints/) for the script:

> Automation blueprints are pre-made automations that you can easily add to your Home Assistant instance. Each blueprint can be added as many times as you want.

When I create a new script using the blueprint, I get a form to fill with selects for the channel and media player:
![blueprint](static/media/2023-11-ha-molotov-deeplinks-blueprint-1.png)
![blueprint](static/media/2023-11-ha-molotov-deeplinks-blueprint-2.png)

I then created a script for each of the few channels we care about home. I also updated the alexa cloud configuration to include all the scripts.

Now I can ask Alexa to turn on a channel on the Shield, and it works! Home Assistant is awesome!
