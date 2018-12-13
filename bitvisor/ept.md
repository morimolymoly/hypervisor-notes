# BitVisorのEPT周りのコールドリーディング
## vt_ept構造体
* `cnt` eptのエントリのカウント？
* `cleared`
* `ncr3tbl` eptポインタの仮想アドレス
* `ncr3tbl_phys` eptポインタ(`0x000000000000201AULL`)にセットするための物理アドレス
* `void *tbl[NUM_OF_EPTBL]`
* `phys_t tbl_phys[NUM_OF_EPTBL]`
* `cur.level`
* `cur.gphys`
* `cur.entry[EPT_LEVELS];`

## init
`vt_ept_init`関数(core/vt_ept.c)で初期化が行われる．
`vt_ept`構造体の`ncr3tbl`に1ページ分のallocがされる．`ncr3tbl_phys`はそれの物理アドレス．

## vt_paging_flush_guest_tlb
* invvpid
* invept

## vt_ept_updatecr3関数
`vt_paging_flush_guest_tlb`してからguest_pdpte0~3を書き換える．

## vt_ept_clear_all関数
クリアして`vt_paging_flush_guest_tlb`実行

## vt_ept_violation関数
`do_ept_violation`(core/vt_main.c) -> `vt_paging_npf`(core/vt_paging.c) -> `vt_ept_violation`(core/vt_ept.c)  
MMIOのフックとかをやってる．

## vt_ept_map_page_clear_cleared関数

## vt_ept_map_page_sub関数


## vt_ept_map_page関数

