# Proxmoxのディスクサイズ拡張
毎回忘れるので備忘

## 前提
Proxmox上に構築したUbuntu20.04  
32GB -> 64GBにする。

### Proxmox側の拡張
![image](https://github.com/sumeshi/api/assets/35072092/209158c7-0048-4997-b3a1-9e72e1233c1c)

Disk Action > Resize

![image](https://github.com/sumeshi/api/assets/35072092/3b51bfd3-810b-4e78-b1e5-3ee89fe4cce0)

増やしたい分だけ。

### Ubuntu側の拡張
```bash
$ df -h
```
![image](https://github.com/sumeshi/api/assets/35072092/a8ae700e-ace4-40a9-8fca-d7ef9344c216)  
色々書いてあるけど重要なのはlvなんとかのとこ。
これがLVMパーティションなので拡張する

```bash
$ sudo parted -l
```
このコマンドでext4であること、ディスクサイズが64GBであることが確認できる  

```bash
$ growpart /dev/sda 3
$ lsblk
```
/dev/sda3自体は64GBになった  
![image](https://github.com/sumeshi/api/assets/35072092/eaa37dd6-b181-4b46-abb4-77068f8ccf31)

```bash
$ pvresize /dev/sda3
$ pvdisplay
```
物理ボリュームもリサイズ  
![image](https://github.com/sumeshi/api/assets/35072092/dab7f2fa-77a0-4643-ba7c-a7a9d06df34c)

```bash
$ sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
$ sudo lvdisplay
```
次にLVMパーティションを最大サイズに拡張  
![image](https://github.com/sumeshi/api/assets/35072092/0f16e623-15bb-44a6-b320-a7288788e61e)

```bash
$ resize2fs
$ df -h
```
ファイルシステムに反映させておわり
![image](https://github.com/sumeshi/api/assets/35072092/c577f9a0-fefe-4336-93d7-d320477a0630)
