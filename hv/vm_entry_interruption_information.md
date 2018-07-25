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
    * error codeが格納される
    * page faultならPFEC(Page Fault Error Code)
    * TBD
* error_code_valid
    * deliver_error_code_bitを有効にするを設定する
* reserved
    * TBD
* valid_bit
    * TBD
