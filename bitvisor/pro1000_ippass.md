# pro1000 ippass
## 初期化
`PCI_DRIVER_INIT (vpn_pro1000_init);`でドライバ起動．  
`vpn_pro1000_init`がよばれる．  
こいつをとうろくする．  
```c
static struct pci_driver pro1000_driver = {
	.name		= driver_name,
	.longname	= driver_longname,
	.driver_options	= "tty,net,virtio",
	.device		= "class_code=020000,id="
			/* 31608004.pdf */
			  "8086:105e|" /* Dual port */
			  "8086:1081|"
    ...
	.new		= pro1000_new,	
	.config_read	= pro1000_config_read, // PCI Configuration Spaceをよむ
	.config_write	= pro1000_config_write,// PCI Configuration Spaceをかく
```

`vpn_pro1000_new`関数がPCI Configuration Spaceにアクセスしてどうこうな部分．  

```c
static void 
vpn_pro1000_new (struct pci_device *pci_device, bool option_tty,
		 char *option_net, bool option_virtio)
{
	int i;
	struct data2 *d2;
	struct data *d;
	void *tmp;
	struct pci_bar_info bar_info;
	struct nicfunc *virtio_net_func;

	if ((pci_device->config_space.base_address[0] &
	     PCI_CONFIG_BASE_ADDRESS_SPACEMASK) !=
	    PCI_CONFIG_BASE_ADDRESS_MEMSPACE) {
		printf ("vpn_pro1000: region 0 is not memory space\n");
		return;
	}
	if ((pci_device->base_address_mask[0] &
	     PCI_CONFIG_BASE_ADDRESS_MEMMASK) & 0xFFFF) {
		printf ("vpn_pro1000: region 0 is too small\n");
		return;
	}
	printf ("PRO/1000 found.\n");

#ifdef VTD_TRANS
        if (iommu_detected) {
                add_remap(pci_device->address.bus_no ,pci_device->address.device_no ,pci_device->address.func_no,
                          vmm_start_inf() >> 12, (vmm_term_inf()-vmm_start_inf()) >> 12, PERM_DMA_RW) ;
        }
#endif // of VTD_TRANS

	d2 = alloc (sizeof *d2);
	memset (d2, 0, sizeof *d2);
	d2->nethandle = net_new_nic (option_net, option_tty);
	alloc_pages (&tmp, NULL, (BUFSIZE + PAGESIZE - 1) / PAGESIZE);
	memset (tmp, 0, (BUFSIZE + PAGESIZE - 1) / PAGESIZE * PAGESIZE);
	d2->buf = tmp;
	d2->buf_premap = net_premap_recvbuf (d2->nethandle, tmp, BUFSIZE);
	spinlock_init (&d2->lock);
	d = alloc (sizeof *d * 6);
	for (i = 0; i < 6; i++) {
		d[i].d = d2;
		d[i].e = 0;
		pci_get_bar_info (pci_device, i, &bar_info);
		reghook (&d[i], i, &bar_info);
	}
	d->disable = false;
	d2->d1 = d;
	pro1000_enable_dma_and_memory (pci_device);
	get_macaddr (d2, d2->macaddr);
	pci_device->host = d;
	pci_device->driver->options.use_base_address_mask_emulation = 1;
	d2->pci_device = pci_device;
	d2->virtio_net = NULL;
	if (option_virtio) {
		d2->virtio_net = virtio_net_init (&virtio_net_func,
						  d2->macaddr,
						  pro1000_intr_clear,
						  pro1000_intr_set,
						  pro1000_intr_disable,
						  pro1000_intr_enable, d2);
	}
	if (d2->virtio_net) {
		pro1000_disable_io (pci_device, d);
		d2->virtio_net_msi = pci_msi_init (pci_device, pro1000_msi,
						   d2);
		if (d2->virtio_net_msi)
			virtio_net_set_msix (d2->virtio_net, 0x5,
					     pro1000_msix_disable,
					     pro1000_msix_enable, d2);
		pci_device->driver->options.use_base_address_mask_emulation =
			0;
		net_init (d2->nethandle, d2, &phys_func, d2->virtio_net,
			  virtio_net_func);
		d2->seize = true;
	} else {
		d2->seize = net_init (d2->nethandle, d2, &phys_func, NULL,
				      NULL);
	}
	if (d2->seize) {
		pci_system_disconnect (pci_device);
		/* Enabling bus master and memory space again because
		 * they might be disabled after disconnecting firmware
		 * drivers. */
		pro1000_enable_dma_and_memory (pci_device);
		seize_pro1000 (d2);
		net_start (d2->nethandle);
	}
	LIST1_PUSH (d2list, d2);
	return;
}
```

## MMIO関連
```c
	for (i = 0; i < 6; i++) {
		d[i].d = d2;
		d[i].e = 0;
		pci_get_bar_info (pci_device, i, &bar_info);
		reghook (&d[i], i, &bar_info);
	}
```
`reghook`でフック関数が登録されるが，`mmio_register(core/mmio.c)`がmmio関連を定義．  
`pci_get_bar_info`でBARを読み取る．

### pci_get_bar_info
barが64bitかとか，メモリ空間かとかそういうのをみる
```c
void
pci_get_bar_info (struct pci_device *pci_device, int n,
		  struct pci_bar_info *bar_info)
{
	pci_get_bar_info_internal (pci_device, n, bar_info, 0, NULL);
}
static u64
pci_get_bar_info_internal (struct pci_device *pci_device, int n,
			   struct pci_bar_info *bar_info, u16 offset,
			   union mem *data)
{
	enum pci_bar_info_type type;
	u32 low, high, mask, and;
	u32 match_offset;
	u64 newbase;

	if (n < 0 || n >= PCI_CONFIG_BASE_ADDRESS_NUMS)
		goto err;
	if (!(pci_device->base_address_mask_valid & (1 << n)))
		goto err;
	low = pci_device->config_space.base_address[n];
	high = 0;
	mask = pci_device->base_address_mask[n];
	match_offset = 0x10 + 4 * n;
	if ((mask & PCI_CONFIG_BASE_ADDRESS_SPACEMASK) ==
	    PCI_CONFIG_BASE_ADDRESS_MEMSPACE) {
		if ((mask & PCI_CONFIG_BASE_ADDRESS_TYPEMASK) ==
		    PCI_CONFIG_BASE_ADDRESS_TYPE64 &&
		    n + 1 < PCI_CONFIG_BASE_ADDRESS_NUMS)
			high = pci_device->config_space.base_address[n + 1];
		and = PCI_CONFIG_BASE_ADDRESS_MEMMASK;
		type = PCI_BAR_INFO_TYPE_MEM;
	} else {
		and = PCI_CONFIG_BASE_ADDRESS_IOMASK;
		type = PCI_BAR_INFO_TYPE_IO;
	}
	if (!(mask & and))
		goto err;
	bar_info->base = newbase = (low & mask & and) | (u64)high << 32;
	if (offset == match_offset)
		newbase = (data->dword & mask & and) | (u64)high << 32;
	else if (offset == match_offset + 4)
		newbase = (low & mask & and) | (u64)data->dword << 32;
	/* The bit 63 must be cleared for CPU access.  If it is set,
	 * the BAR access is for size detection. */
	if (newbase == (mask & and) || (newbase & 0x8000000000000000ULL))
		goto err;
	bar_info->len = (mask & and) & (~(mask & and) + 1);
	bar_info->type = type;
	return newbase;
err:
	bar_info->type = PCI_BAR_INFO_TYPE_NONE;
	return 0;
}
```

### reghook
ハンドラ登録
```c
static void
reghook (struct data *d, int i, struct pci_bar_info *bar)
{
	if (bar->type == PCI_BAR_INFO_TYPE_NONE)
		return;
	unreghook (d);
	d->i = i;
	d->e = 0;
	if (bar->type == PCI_BAR_INFO_TYPE_IO) {
		d->io = 1;
		d->hd = core_io_register_handler (bar->base, bar->len,
						  iohandler, d,
						  CORE_IO_PRIO_EXCLUSIVE,
						  driver_name);
	} else {
		d->mapaddr = bar->base;
		d->maplen = bar->len;
		d->map = mapmem_gphys (bar->base, bar->len, MAPMEM_WRITE);
		if (!d->map)
			panic ("mapmem failed");
		d->io = 0;
		d->h = mmio_register (bar->base, bar->len, mmhandler, d);
		if (!d->h)
			panic ("mmio_register failed");
	}
	d->e = 1;
}
```
