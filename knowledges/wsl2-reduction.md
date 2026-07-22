# DockerのせいでバカデカになったWSL2のディスクイメージを小さくする
備忘。

## 手順

### ファイル削除

WizTreeとかでどこが容量食ってるかみる。  
だいたいWSL2の `ext4.vhdx` 。

いい感じにdockerイメージを削除する。

```bash
$ docker rmi hogehoge
$ docker system prune
```

あとはrustのビルドキャッシュがクソデカになっているときもある。
githubリポジトリとかおいてるディレクトリの容量を計算して削除。

```bash
$ du -h -d 1
```

### ディスクサイズ削減

ファイル消しただけでは縮まない。

まずはWSL2を落とす。

```ps1
> wsl --shutdown
```

```ps1
> diskpart
> select vdisk file="C:\Users\{ユーザ名}\AppData\Local\Packages\{PackageFamilyName}\LocalState\ext4.vhdx"
> attach vdisk readonly
> compact vdisk
> detach vdisk
> exit
```

だいたいこれでおｋ

