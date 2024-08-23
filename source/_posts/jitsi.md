---
title: jitsi
date: 2024-08-20 14:16:08
tags:
---

# Installation des prerequis
sudo apt install -y gnupg2 nginx-full sudo curl

sudo apt update

sudo apt install apt-transport-https

sudo apt-add-repository universe

sudo apt update

# Modification du hostname
sudo hostnamectl set-hostname mydomain

vim /etc/hosts

```
127.0.0.1     localhost
192.168.1.127     mydomain
127.0.1.1     nom de la machine
```

# en root ajout jitsi au repository
echo 'deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/' | sudo tee /etc/apt/sources.list.d/jitsi-stable.list > /dev/null

curl https://download.jitsi.org/jitsi-key.gpg.key | sudo sh -c 'gpg --dearmor > /usr/share/keyrings/jitsi-keyring.gpg'

sudo apt update

# Configuration du pare-feu
sudo ufw allow 80/tcp

sudo ufw allow 443/tcp

sudo ufw allow 10000/udp

sudo ufw allow 22/tcp

sudo ufw allow 3478/udp

sudo ufw allow 5349/tcp

# Installation Jitsi
sudo apt install jitsi-meet

# Saisissez le nom de domaine
# Selectionne  self-signed certificate

# en root
nano /etc/prosody/conf.avail/mydomain.cfg.lua

```
VirtualHost "mydomain"
...
authentication = "internal_plain"
...
```

# au dessus du block precedent, inserez la conf ci-dessous en respectant les tabulations

```
VirtualHost "guest.mydomain"
    authentication = "anonymous"
    c2s_require_encryption = false
    modules_enabled = {
        "bosh";
        "pubsub";
        "ping";
        "speakerstats";
        "turncredentials";
        "conference_duration";
    }
```


# Configuration de Jicofo, composant permettant de manager les sessions utilisateurs
nano /etc/jitsi/jicofo/sip-communicator.properties

```
org.jitsi.jicofo.auth.URL=XMPP:mydomain
```

# Configuration de Prosody en fonction de l'exemple ci-dessous
nano /etc/prosody/conf.d/mydomain.cfg.lua

```
-- internal muc component
Component "internal.auth.mydomain" "muc"
    storage = "memory"
    modules_enabled = {
          "ping";
        }
        admins = { "focus@auth.mydomain", "jvb@auth.mydomain", "jibri@auth.mydomain" }
        muc_room_locking = false
        muc_room_default_public_jids = true
        muc_room_cache_size = 1000
        c2s_require_encryption = false
```

```
VirtualHost "recorder.mydomain"
    modules_enabled = {
        "ping";
    }
    authentication = "internal_hashed"
    c2s_require_encryption = false
    allow_empty_token = true
```

# Creation d'un compte admin pour jitsi
sudo prosodyctl register lbc mydomain lbclbc

# Ajout de Jitsi au SIP
nano /etc/jitsi/jicofo/sip-communicator.properties

```
org.jitsi.jicofo.BRIDGE_MUC=JvbBrewery@internal.auth.mydomain
org.jitsi.jicofo.jibri.BREWERY=JibriBrewery@internal.auth.mydomain
org.jitsi.jicofo.jibri.PENDING_TIMEOUT=90
```

# En root
prosodyctl register jibri auth.mydomain lbclbc
prosodyctl register recorder recorder.mydomain lbclbc


# Redemarrage des services
sudo systemctl restart {prosody,jicofo,jitsi-videobridge2,nginx}
sudo systemctl status {prosody,jicofo,jitsi-videobridge2,nginx}

# Testez si Jitsi demande un login apres avoir lancer une reunion
mydomain

# En root, configuration loopback interfaces
apt -y install linux-image-extra-virtual

echo "snd_aloop" >> /etc/modules

modprobe snd-aloop

lsmod | grep snd_aloop

Vous aurez la sortie suivante:
snd_aloop 24576 0
snd_pcm 98304 1 snd_aloop
snd 81920 3 snd_timer,snd_aloop,snd_pcm


# installation ffmpeh
sudo apt install ffmpeg

# installation chrome, preferable pour debian
curl -o - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add

echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list

apt update

apt install google-chrome-stable

# installation chrome, pour les autres
curl -fsSL https://dl.google.com/linux/linux_signing_key.pub | gpg --dearmor -o /usr/share/keyrings/google-chrome-keyring.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/google-chrome-keyring.gpg] http://dl.google.com/linux/chrome/deb/ stable main" | tee /etc/apt/sources.list.d/google-chrome.list

apt-get -y update

apt-get -y install google-chrome-stable

apt-mark hold google-chrome-stable

# ignore l'avertissement de google chrome
mkdir -p /etc/opt/chrome/policies/managed

echo '{ "CommandLineFlagSecurityWarningsEnabled": false }' >>/etc/opt/chrome/policies/managed/managed_policies.json

# installation chromedriver pour debian
CHROME_DRIVER_VERSION=`curl -sS chromedriver.storage.googleapis.com/LATEST_RELEASE`

wget -N http://chromedriver.storage.googleapis.com/$CHROME_DRIVER_VERSION/chromedriver_linux64.zip -P ~/

unzip ~/chromedriver_linux64.zip -d ~/

rm ~/chromedriver_linux64.zip

mv -f ~/chromedriver /usr/local/bin/chromedriver

chown root:root /usr/local/bin/chromedriver

chmod 0755 /usr/local/bin/chromedriver

# installation chromedriver autre linux
CHROME_VER=$(dpkg -s google-chrome-stable | egrep "^Version" | cut -d " " -f2 | cut -d. -f1-3)

CHROMELAB_LINK="https://googlechromelabs.github.io/chrome-for-testing"

CHROMEDRIVER_LINK=$(curl -s $CHROMELAB_LINK/known-good-versions-with-downloads.json | jq -r ".versions[].downloads.chromedriver | select(. != null) | .[].url" | grep linux64 | grep "$CHROME_VER" | tail -1)

wget -O /tmp/chromedriver-linux64.zip $CHROMEDRIVER_LINK

rm -rf /tmp/chromedriver-linux64

unzip -o /tmp/chromedriver-linux64.zip -d /tmp

mv /tmp/chromedriver-linux64/chromedriver /usr/local/bin/

chown root:root /usr/local/bin/chromedriver

chmod 755 /usr/local/bin/chromedriver

# installation jibri pour debian
wget -qO - https://download.jitsi.org/jitsi-key.gpg.key | sudo apt-key add -

sh -c "echo 'deb https://download.jitsi.org stable/' > /etc/apt/sources.list.d/jitsi-stable.list"

apt update

apt install jibri

usermod -aG adm,audio,video,plugdev jibri

# installation jibri pour autre linux
curl https://download.jitsi.org/jitsi-key.gpg.key | sudo sh -c 'gpg --dearmor > /usr/share/keyrings/jitsi-keyring.gpg'

echo 'deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/' | sudo tee /etc/apt/sources.list.d/jitsi-stable.list > /dev/null

apt update

apt install jibri

usermod -aG adm,audio,video,plugdev jibri

nano /etc/jitsi/jibri/jibri.conf

```
jibri {
  // A unique identifier for this Jibri
  // TODO: eventually this will be required with no default
  id = ""
  // Whether or not Jibri should return to idle state after handling
  // (successfully or unsuccessfully) a request.  A value of 'true'
  // here means that a Jibri will NOT return back to the IDLE state
  // and will need to be restarted in order to be used again.
  single-use-mode = false
  api {
    http {
      external-api-port = 2222
      internal-api-port = 3333
    }
    xmpp {
      // See example_xmpp_envs.conf for an example of what is expected here
      environments = [
      {
        name = "prod environment"
        xmpp-server-hosts = ["mydomain"]
        xmpp-domain = "mydomain"
 
        control-muc {
                domain = "internal.auth.mydomain"
                room-name = "JibriBrewery"
                nickname = "jibri"
        }
 
        control-login {
                domain = "auth.mydomain"
                username = "jibri"
                password = "lbclbc"
        }
 
        call-login {
                domain = "recorder.mydomain"
                username = "recorder"
                password = "lbclbc"
        }
        strip-from-room-domain = "conference."
        usage-timeout = 0
        trust-all-xmpp-certs = true
}
      ]
    }
  }
  recording {
    recordings-directory = "/srv/recordings"
    # TODO: make this an optional param and remove the default
    # finalize-script = "/path/to/finalize"
  }
  streaming {
    // A list of regex patterns for allowed RTMP URLs.  The RTMP URL used
    // when starting a stream must match at least one of the patterns in
    // this list.
    rtmp-allow-list = [
      // By default, all services are allowed
      ".*"
    ]
  }
  ffmpeg {
    resolution = "1280x720"
    video-encode-preset = "veryfast"
    // The audio source that will be used to capture audio on Linux
    audio-source = "alsa"
    // The audio device that will be used to capture audio on Linux
    audio-device = "plug:bsnoop"
  }
  chrome {
    // The flags which will be passed to chromium when launching
    flags = [
      "--use-fake-ui-for-media-stream",
      "--start-maximized",
      "--kiosk",
      "--enabled",
      "--disable-infobars",
      "--autoplay-policy=no-user-gesture-required",
      // "--headless",
      "--no-sandbox",
      //"--disable-dev-shm-usage",
      "--ignore-certificate-errors"
    ]
  }
  stats {
    enable-stats-d = true
  }
  webhook {
    // A list of subscribers interested in receiving webhook events
    subscribers = []
  }
  jwt-info {
    // The path to a .pem file which will be used to sign JWT tokens used in webhook
    // requests.  If not set, no JWT will be added to webhook requests.
    # signing-key-path = "/path/to/key.pem"
 
    // The kid to use as part of the JWT
    # kid = "key-id"
 
    // The issuer of the JWT
    # issuer = "issuer"
 
    // The audience of the JWT
    # audience = "audience"
 
    // The TTL of each generated JWT.  Can't be less than 10 minutes.
    # ttl = 1 hour
  }
  call-status-checks {
    // If all clients have their audio and video muted and if Jibri does not
    // detect any data stream (audio or video) comming in, it will stop
    // recording after NO_MEDIA_TIMEOUT expires.
    no-media-timeout = 30 seconds
    // If all clients have their audio and video muted, Jibri consideres this
    // as an empty call and stops the recording after ALL_MUTED_TIMEOUT expires.
    all-muted-timeout = 10 minutes
    // When detecting if a call is empty, Jibri takes into consideration for how
    // long the call has been empty already. If it has been empty for more than
    // DEFAULT_CALL_EMPTY_TIMEOUT, it will consider it empty and stop the recording.
    default-call-empty-timeout = 30 seconds
  }
}
```
# verifier l'existence de log jibri
ls -ll /var/log/jitsi/jibri

mkdir /recordings
chown jibri:jibri /recordings

# configuration fichier prosody
nano /etc/prosody/prosody.cfg.lua

# ajouter ce block dans le fichier conf
```
-- internal muc component, meant to enable pools of jibri and jigasi clients
Component "internal.auth.mydomain" "muc"
    storage = "memory"
    modules_enabled = {
          "ping";
        }
        admins = { "focus@auth.mydomain", "jvb@auth.mydomain", "jibri@auth.mydomain" }
        muc_room_locking = false
        muc_room_default_public_jids = true
        muc_room_cache_size = 1000
        c2s_require_encryption = false
```

```
VirtualHost "recorder.mydomain"
    modules_enabled = {
        "ping";
    }
    authentication = "internal_hashed"
    c2s_require_encryption = false
    allow_empty_token = true
```

```
---Set up a MUC (multi-user chat) room server on conference.example.com:
Component "conference.mydomain" "muc"
--- Store MUC messages in an archive and allow users to access it
modules_enabled = { "muc_mam" }
```


# ajout d'un block dans le fichier conf jicofo
nano /etc/jitsi/jicofo/jicofo.conf

```
# Jicofo HOCON configuration. See reference.conf in /usr/share/jicofo/jicofo.jar for
#available options, syntax, and default values.
jicofo {
  XMPP: {
    client: {
      client-proxy: "focus.webrtc.multiverh.fr"
      xmpp-domain: "webrtc.multiverh.fr"
      domain: "auth.webrtc.multiverh.fr"
      username: "focus"
      password: "HwQVZ9V6XtoTv3FP"
    }
    trusted-domains: [ "recorder.webrtc.multiverh.fr" ]
  }
  bridge: {
    brewery-jid: "JvbBrewery@internal.auth.webrtc.multiverh.fr"
  }
  jibri {
    brewery-jid = "JibriBrewery@internal.auth.webrtc.multiverh.fr"
    pending-timeout = 90 seconds
  }
}
```

# modification du ficher /etc/jitsi/meet/mydomain-config.js
```
// recording
recordingService = {
  enabled: true,
  sharingEnabled: true,
  hideStorageWarning: false,
};

// liveStreaming
liveStreaming = {
  enabled: true,
};

hiddenDomain = "recorder.yourdomain.com";
```

# executer jibri avec java11
nano /opt/jitsi/jibri/launch.sh

# remplacer java par le script suivant
/usr/lib/jvm/java-11-openjdk-amd64/bin/java

# redemarrage des services
systemctl restart prosody jicofo jitsi-videobridge2 jibri
systemctl enable --now jibri
systemctl status prosody jicofo jitsi-videobridge2 jibri


# Ajouter ce block pour les participants sans authen
VirtualHost "guest.mydomain"
    authentication = "anonymous"
    c2s_require_encryption = false

Authentification en root
nano /etc/jitsi/meet/mydomain-config.js
anonymousdomain: 'guest.mydomain',

decommente dans la ligne 708 de toolbarButtons
toolbarButtons: [
       'camera',
       'chat',
    //    'closedcaptions',
       'desktop',
    //    'download',
    //    'embedmeeting',
    //    'etherpad',
    //    'feedback',
       'filmstrip',
    //    'fullscreen',
       'hangup',
    //    'help',
    //    'highlight',
    //    'invite',
    //    'linktosalesforce',
    //    'livestreaming',
       'microphone',
       'noisesuppression',
       'participants-pane',
    //    'profile',
       'raisehand',
       'recording',
       'security',
       'select-background',
       'settings',
    //    'shareaudio',
    //    'sharedvideo',
    //    'shortcuts',
    //    'stats',
       'tileview',
       'toggle-camera',
       'videoquality',
       'whiteboard',
    ],

lobby_muc = "lobby.webrtc.multiverh.fr"
main_muc = "conference.webrtc.multiverh.fr"
