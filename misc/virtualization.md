# 仮想化概要
## Type1とType2
* Type1 - ベアメタルハイパーバイザ．ハードウェアの上で直接動作する．(Xen,KVM,Hyper-V,Bareflank,BitVisor)
* Type2 - ホストOSの上でプロセスとして動く.(VirtualBox, VMWare, QEMU, Bochs)

## 完全仮想化と準仮想化
### 完全仮想化
ハードウェアからBIOS,UEFIなどすべてをエミュレートする仮想化．  
VirtualBoxやVMWareなどが有名．  
ハードウェアの仮想化はコストが高い．
### 準仮想化
ハードウェアの仮想化を行わず，仮想ハードウェアにアクセスするように改変したOSを動かすもの．  
完全仮想化よりはやい．
