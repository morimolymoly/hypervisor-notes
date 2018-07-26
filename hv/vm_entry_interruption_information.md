# vm_entry_interruption_information
`vm_entry_interruption_information`には，以下の項目がある．
* vector ベクター
    * ただの例外番号
    * システムコール番号に対応する
    * int3なら3bit目にbitが立つ
* interruption_type
    * 例外の種類
        * external_interrupt = 0ULL;
        * non_maskable_interrupt = 2ULL;
        * hardware_exception = 3ULL;
        * software_interrupt = 4ULL;
        * privileged_software_exception = 5ULL;
        * software_exception = 6ULL;
* deliver_error_code_bit
    * エラーコードをスタックに積むかどうかをきめる
* reserved
    * TBD
* valid_bit
    * このbitがたっているときのみ，インジェクションが起こる
    * 毎回フラッシュされる

# vm entry exception error code
exceptionのエラーコード.
PageFaultならPFEC(Page Fault Error Code)

# vm entry instruction length
命令の長さを格納する．  
そのぶんRIPが進む．
