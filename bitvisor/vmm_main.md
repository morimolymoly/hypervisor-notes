# ENTRYPOINTからBSPとAP
マルチプロセッサー(MP)の初期化において選択された一番目のプロセッサーのこと。
ほかはAP．
BSPはvmm_main，APは，apinitproc0に飛ぶ．
とりあえずコードで本質じゃないところは省いていく．

# BSP
## vmm_main
```c
asmlinkage void
vmm_main (struct multiboot_info *mi_arg)
{
	uefi_booted = !mi_arg;
	if (!uefi_booted)
		memcpy (&mi, mi_arg, sizeof (struct multiboot_info));
	initfunc_init ();
	call_initfunc ("global");
	start_all_processors (bsp_proc, ap_proc);
}
```
`call_initfunc ("global");`で`global`なinitfuncを呼び出す．  
globalなinitfuncとは，`INITFUNC ("global3", get_shiftflags);`のような，`INITFUNC`マクロによって登録された，idに`global`というprefixがついた関数である．  
ttyとかそこらへんの関数が呼び出されるっぽい．  
`INITFUNC ("global`でgrepすればそこらへんがでてくる．  

## start_all_processors
```c
void
start_all_processors (void (*bsp_initproc) (void), void (*ap_initproc) (void))
{
	initproc_bsp = bsp_initproc;
	initproc_ap = ap_initproc;
	bsp_continue (bspinitproc1);
}
```
`initproc`変数にbsp，ap双方のinit関数のアドレスを指定したあと，`bsp_cpntinue`関数でスタックを調整してから`bspinitproc1`関数へjumpする．

## bspinitproc1
```c
static asmlinkage void
bspinitproc1 (void)
{
	printf ("Processor 0 (BSP)\n");
    ...
	if (!uefi_booted) {
		apinit_addr = 0xF000;
		ap_start ();
	} else {
        ...
	}
	initproc_bsp ();
	panic ("bspinitproc1");
}
```
`ap_start`でapの初期化関数をキック．`initproc_bsp`は`start_all_processors`で設定された，`bsp_proc`関数．

## bsp_proc
```c
static void
bsp_proc (void)
{
	call_initfunc ("bsp");
	call_parallel ();
	call_initfunc ("pcpu");
}
```
`bsp`prefixがidについているinitfuncを呼び出し，`parerel`prefixのinitfuncを呼び出してから，`pcpu`prefixの関数を呼び出す．  
`INITFUNC ("pcpu`でgrepすると，
```c
INITFUNC ("pcpu0", cpu_mmu_spt_init_pcpu);
INITFUNC ("pcpu0", nmi_init_pcpu);
INITFUNC ("pcpu0", thread_init_pcpu);
INITFUNC ("pcpu0", initipi_init_pcpu);
INITFUNC ("pcpu2", virtualization_init_pcpu);
INITFUNC ("pcpu3", panic_init_pcpu);
INITFUNC ("pcpu4", time_init_pcpu);
INITFUNC ("pcpu4", cache_init_pcpu);
INITFUNC ("pcpu5", create_pass_vm);
```
なるほど，いろいろと初期化してる．
中でも`virtualization_init_pcpu`はVT-xやAMD−Vの初期化関連だろう．  
`create_pass_vm`はVMの本格的な立ち上げ処理だろうか．  

```c
static void
create_pass_vm (void)
{
...
#endif
	current->vmctl.start_vm ();
	panic ("VM stopped.");
}
```
ビンゴ！  
ここでVMを走らせてる．  
`start_vm`は`vmctl_func`構造体の中の関数ポインタだ．  
どうやらVT-xとSVMでインターフェイスを作っているようだ．  
VT-xでは`vt_start_vm`関数のようだ．
```c
void
vt_start_vm (void)
{
	vt_paging_start ();
	vt_mainloop ();
}
```

`vt_mainloop`ではおそらくvmexitが起こるまでwhileでもしているのかな．  
```c
static void
vt_mainloop (void)
{
	enum vmmerr err;
	ulong cr0, acr;
	u64 efer;
	bool nmi;

	for (;;) {
		schedule ();
        ...
		if (current->initipi.get_init_count ())
			vt_init_signal ();
		if (current->halt) {
			vt__halt ();
			current->halt = false;
			continue;
		}
        ...
		/* when the state is switching, do single step */
		if (current->u.vt.vr.sw.enable) {
			vt__nmi ();
			vt_interrupt ();
			vt__event_delivery_setup ();
			vt_msr_own_process_msrs ();
			nmi = vt__vm_run_with_tf ();
			vt_paging_tlbflush ();
			if (!nmi) {
				vt__event_delivery_check ();
				vt__exit_reason ();
			}
		} else {	/* not switching */
			vt__nmi ();
			vt_interrupt ();
			vt__event_delivery_setup ();
			vt_msr_own_process_msrs ();
			nmi = vt__vm_run ();
			vt_paging_tlbflush ();
			if (!nmi) {
				vt__event_delivery_check ();
				vt__exit_reason ();
			}
		}
	}
}
```
`vt__vm_run`,`vt__vm_run_with_tf`関数でvmenterする．  
vmexitするとここから帰ってきて，`vt__exit_reason`でexit reasonに応じた動作をするっぽい．
