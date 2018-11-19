# 種類(モジュール)
* null - パケットを破棄
* pass - パススルー(shadowingする)
* ip - bitvisorがNICを専有
* ippass - パススルーだけどBitvisorもNICがつかえる(これBareflankで実装するよ)
* vpn - vpn

# ネットワークAPI(net/netapi.c)
ここに定義．受信と送信コールバックを定義．モジュールが実態を実装．

# ippass概要
* 受信コールバック - net_ip_phys_recv
* 送信コールバック - net_ip_virt_recv
