# SYSTEM 権限でシェルを取る
備忘。Windows のコマンドプロンプトでSYSTEM権限を取る方法

## コマンド
[PsExec](https://learn.microsoft.com/ja-jp/sysinternals/downloads/psexec) を使う。
Microsoft が提供するユーティリティ Sysinternals Suite のひとつ。

```
› PsExec -i -s C:\Windows\System32\cmd.exe
```

### オプション
> -i  
>  リモート システム上で指定したセッションのデスクトップと対話するようにプログラムを実行します。 セッションが指定されていない場合、プロセスはコンソール セッションで実行されます。 このフラグは、(リダイレクトされた Standard IO を使用して) コンソール アプリケーションを対話的に実行する場合に必要です。

> -s  
> システム アカウントでリモート プロセスを実行します。

https://learn.microsoft.com/ja-jp/sysinternals/downloads/psexec より抜粋

## 使うことあんのこれ
実はたまに使う。  

[Sysmon](https://learn.microsoft.com/ja-jp/sysinternals/downloads/sysmon) とか使うとローカル管理者でも参照できないフォルダとか作られるので使わざるを得ないときがある。
