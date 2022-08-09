# ARKサーバのログイン人数を監視するDiscordBotを作る
Python3でサクッと仕上げました。

## 仕上がり
![](https://i.gyazo.com/7d833d2bc87eb9878e4a4702d0cb9a15.png)

最終的な出来はこんな感じです。  
1分に1回疎通確認をして、サーバのバージョンとログイン人数をゲームアクティビティに表示しています。

## 手順
### 1. Botの追加
[Discord Developer Portal](https://discord.com/developers/applications) にログインして、New Applicationからアプリケーションを作成します。  
名前を設定するとアプリケーションの固有設定に入るので、サイドバーからBotを選択してAdd Botを選択。  

Botが作成できたらアイコンとか諸々を設定しましょう。今回はチャットとかしないのでパーミッション設定は特に必要ないです。  
TOKENに表示されてる文字列をコピーしておいてください。後で使います。


### 2. Bot用コードの作成
ARKサーバを動かしているUbuntu上のPython3で作成します。コード自体はWindowsでも動くと思います。  
今回はサーバの状態確認だけなので、特に凝ったことはしてません。

必要なライブラリのインストール
```.bash
$ pip install python-valve discord.py
```

適当な場所にコードを配置 (run.py)
```.bash
$ mkdir ~/discordbot
$ vim ~/discordbot/run.py
```

下記のコードを追加
```.py
from typing import NamedTuple

import valve.source.a2s
import discord
from discord.ext import tasks


class ServerStatus(NamedTuple):
    info: str
    status: discord.Status 


def get_server_status() -> ServerStatus:
    # ここでサーバのIPアドレス, ポート番号を指定する
    SERVER_ADDRESS = ("192.168.0.1", 27015)

    try:
        with valve.source.a2s.ServerQuerier(SERVER_ADDRESS) as server:
            info = server.info()
            return ServerStatus(f"{info['server_name'].split(' - ')[1].strip('()')} ({info['player_count']}/{info['max_players']})", discord.Status.online)
    except:
        return ServerStatus("Offline: ?/?", discord.Status.idle)


client = discord.Client()

@tasks.loop(seconds=60)
async def update_status():
    server = get_server_status()
    activity = discord.Game(server.info)
    await client.change_presence(status=server.status, activity=activity)

@client.event
async def on_ready():
    update_status.start()

# ここでTOKENを指定する
client.run('HOGEHOGEHOGEHOGEHOGE')
```

作成が終わったら動作確認をしましょう。
```.bash
$ python ~/discordbot/run.py
```

ちゃんとゲームアクティビティが更新されたらOKです。


### 3. 自動起動設定(ubuntu)
systemdで設定します。

ユーザフォルダ配下に設定ファイルを作成
```.bash
$ mkdir -p ~/.config/systemd/user/
$ vim ~/.config/systemd/user/discordbot.service
```

下記の設定を記述, run.pyの場所はよしなに設定しましょう。
```
[Unit]
Description = DiscordBot for ARK Server

[Service]
ExecStart=/usr/bin/python3 /home/user/discordbot/run.py
Restart=always

[Install]
WantedBy=default.target
```

設定ファイルの権限を設定して動作確認
```.bash
$ chmod 664 ~/.config/systemd/user/discordbot.service
$ systemctl --user start discordbot.service
$ systemctl --user status discordbot.service
```

うまく動いてたら永続化設定をしましょう。
```.bash
$ systemctl --user enable discordbot
$ sudo loginctl enable-linger
```

参考: https://zenn.dev/sweetsoundstory/books/python-discord-bot/viewer/ubuntu\


## 所見
今回はプレイ人数の監視のみしていますが、RCONとか使えばチャットからサーバの再起動とかもできると思います。  
ARK楽しいのでみんなもやろうね  

お わ り
