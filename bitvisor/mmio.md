# MMIO
## 構造体
### mmio_handle
* phys_t gphys - ゲストのBARの物理アドレス
* uint len - 長さ
* void *data - data構造体のポインタ(各デバイスによってことなる？？)
* mmio_handler_t hanler - MMIOのレジスタにアクセスしてきた場合のハンドラ(特定のレジスタのエミュレートをする．ほかはパススルー)
* bool unregisterd - dirty bit
* bool unlocked_handler - pro1000ではつかってねえ．わからん．
