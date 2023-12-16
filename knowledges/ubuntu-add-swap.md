# Ubuntu22.04にSwapfileを追加する
これも毎回忘れる。クソボケ。

## 設定確認
```bash
$ sudo swapon --show
```

あれば表示。そこになければないですね。

## お前を消す方法
```bash
$ sudo swapoff /swap.img
```

もしあれば一時的に無効にする  
同名で作り直すならファイル削除してから下記手順を継続


## 作成
```bash
$ sudo fallocate -l 16G /swap.img
```

一般的に積んでるメモリの倍がよいと言われている。今回は8GBの倍で設定

## 設定
```bash
$ chmod 600 /swap.img
$ sudo mkswap /swap.img
$ sudo swapon /swap.img
$ sudo swapon --show
```

設定できたらなんか表示されるはず

## 永続化
```bash
$ sudo vim /etc/fstab
```

```
/swapfile none swap sw 0 0
```

上記を追記。これやらんといちいち再起動のたびにswaponが必要になる。
