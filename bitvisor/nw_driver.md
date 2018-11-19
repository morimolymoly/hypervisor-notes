# 種類(モジュール)
* null - パケットを破棄
* pass - パススルー(shadowingする)
* ip - bitvisorがNICを専有
* ippass - パススルーだけどBitvisorもNICがつかえる(これBareflankで実装するよ)
* vpn - vpn

# ネットワークAPI(net/netapi.c)
ここに定義．受信と送信コールバックを定義．モジュールが実態を実装．

# ippass概要(ip/net_main.c)
* 受信コールバック - net_ip_phys_recv
* 送信コールバック - net_ip_virt_recv

## net_ip_phys_recv
NICがパケット受信したときによばれる．  
`p->pass`つまりippassなら`p->virt_func->send()`でゲストにパケット送信．  
`p->input_ok`つまりlwipがokならlwipが操作するようにキューに追加.(lwipは別スレッドが立ってるっぽい)

## net_ip_virt_recv
`p->pass`つまりippassならpassのようにdescriptor shadowinをして，NICに送信．

#モジュール登録
`net_register`(net/netapi.c)から登録．INITFUNCで各モジュールは自身を初期化．
`phys_func`と`virt_func`が関数テーブルの`nicfunc`構造体．  
各ドライバがこれを実装．
