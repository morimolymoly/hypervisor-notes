# dom0setup
`xen/arch/x86/boot/x86_64.S`　こいつがbootするためのやつ.  
multibootに準拠しているやつなのでわりとわかりやすい.  
`call __start_xen`でxenのstartupを行っている.

## __start_xen
クソでかい関数でスタートアップの処理がつらつらと記述されている.  
おなじみ`trap_init`とかを呼び出したりしている.
全部読むのは結構だるい.  

* init_xen_time
* init_guest_cpuid

```
    /* Create initial domain 0. */
    dom0 = domain_create(get_initial_domain_id(), &dom0_cfg, !pv_shim);
    if ( IS_ERR(dom0) || (alloc_dom0_vcpu0(dom0) == NULL) )
        panic("Error creating domain 0\n");
```
