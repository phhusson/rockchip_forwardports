== How to build me ==

Assuming you have your kernel build environment in KERNEL_ARGS (i.e. ARCH=arm CROSS_COMPILE=...).  
Assuming you have your kernel source tree in KERNEL_SRC.  
To build, use:
> ${KERNEL_ARGS} make M=$PWD -C ${KERNEL_SRC} CONFIG_MALI_MIDGARD=m CONFIG_MALI_DEVFREQ=y CONFIG_MALI_DMA_FENCE=y CONFIG_MALI_EXPERT=y CONFIG_MALI_PLATFORM_THIRDPARTY=y CONFIG_MALI_PLATFORM_THIRDPARTY_NAME=rk

This will generate rockchip-vpu.ko, mali_kbase.ko and vtl_ts_ct_ct36x.ko.  
You'll need additional DT modifications to enable those feature, see below.

== VPU (h264 decode) ==

To get rockchip VPU working on a mainline kernel, you'll need to change your DTS.  
Include:

>         vpu: video-codec@ff9a0000 {
>                 compatible = "rockchip,rk3288-vpu";
>                 reg = <0xff9a0000 0x800>;
>                 interrupts = <GIC_SPI 9 IRQ_TYPE_LEVEL_HIGH>,
>                                 <GIC_SPI 10 IRQ_TYPE_LEVEL_HIGH>;
>                 interrupt-names = "vepu", "vdpu";
>                 clocks = <&cru ACLK_VCODEC>, <&cru HCLK_VCODEC>;
>                 clock-names = "aclk", "hclk";
>                 power-domains = <&power RK3288_PD_VIDEO>;
>                 iommus = <&vpu_mmu>;
>                 assigned-clocks = <&cru ACLK_VCODEC>;
>                 assigned-clock-rates = <400000000>;
>                 status = "okay";
>         };
> 
>         vpu_mmu: iommu@ff9a0800 {
>                 compatible = "rockchip,iommu";
>                 reg = <0xff9a0800 0x100>;
>                 interrupts = <GIC_SPI 11 IRQ_TYPE_LEVEL_HIGH>;
>                 interrupt-names = "vpu_mmu";
>                 power-domains = <&power RK3288_PD_VIDEO>;
>                 #iommu-cells = <0>;
>         };

Then build and install rockchip-vpu here  
Finally, install rockchip VPU userland, you can choose either:
 - VDPAU ( https://github.com/rockchip-linux/libvdpau-rockchip )
 - VAAPI ( https://github.com/rockchip-linux/rockchip-va-driver )

At the moment the most stable driver is VAAPI, which can be tested with VLC.

== GPU (Mali, 3D Acceleration) ==

You'll need to change your DTS here as well:

>        gpu: gpu@ffa30000 {
>        	compatible = "arm,malit764",
>        		     "arm,malit76x",
>        		     "arm,malit7xx",
>        		     "arm,mali-midgard";
>        	reg = <0xffa30000 0x10000>;
>        	interrupts = <GIC_SPI 6 IRQ_TYPE_LEVEL_HIGH>,
>        		     <GIC_SPI 7 IRQ_TYPE_LEVEL_HIGH>,
>        		     <GIC_SPI 8 IRQ_TYPE_LEVEL_HIGH>;
>        	interrupt-names = "JOB", "MMU", "GPU";
>        	clocks = <&cru ACLK_GPU>;
>        	clock-names = "clk_mali";
>        	operating-points = <
>        		/* KHz uV */
>        		600000 1250000
>        		/* 500000 1200000 - See crosbug.com/p/33857 */
>        		400000 1100000
>        		300000 1000000
>        		200000 950000
>        		100000 950000
>        	>;
>        	#cooling-cells = <2>; /* min followed by max */
>        	power-domains = <&power RK3288_PD_GPU>;
>        	status = "okay";
>        	mali-supply = <&vdd_gpu>;
>        };

Please note that here mali-supply line is board-specific!  
If your board already has a vdd_gpu, then it should be a good guess.

As for the userspace, you have two choices.  
You can use the driver provided by Mali or the one provided by Rockchip.  
ATM, it seems like the one provided by ARM works only for Wayland,
and the one provided by Rockchip only works for X11.

The Rockhip mali can be found on https://github.com/rockchip-linux/libmali  
Take libmali-midgard.so for RK3288.
Check https://github.com/rockchip-linux/libmali/blob/rockchip/debian/libmali-rk-midgard0.links
for the list of symlinks to do with that libmali.so.  
You can also generate debian packages from this git repo.
You'll then want a X server working with this. Take rockchip's one.  
Sources can be found on https://github.com/rockchip-linux/xserver
and binaries on https://github.com/rockchip-linux/rk-rootfs-build

The ARM mali drivers can be found on http://malideveloper.arm.com/resources/drivers/arm-mali-midgard-gpu-user-space-drivers/  
Take latest variant for "Firefly".
