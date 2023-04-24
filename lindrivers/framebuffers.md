# Framebuffer drivers

A framebuffer driver proides an interface for the following:
	* The display mode setting
	* Memory access to the video buffer
	* Basic 2D acceleration operations (for example, scrolling)

To provide this interface, the frambuffer drivers generally talks to the HW dire
ctly, there are well-known framebuffer drivers, such as:
	* intelfb - a framebuffer for various intel 8xx/9xx compatible graphdev
	* vesafb - a framebuffer driver that uses the VESA standart i-face
	* mxcfb - a framebuffer driver for the i.MX6 chip series

# driver data structures:

Framebuffer drivers depend heavily on four data structures, all defined in dire
ctory `#include </linux/fb.h>`. These structures are:`struct fb_var_screeninfo`,
`struct fb_fix_screen_info`, `struct fb_cmap` - three are made available to and
from the user space code. and `struct fb_info`.

1. The kernel uses an instance of `struct fb_var_screeninfo` to hold variable pr
	operties of the videocard. These values are those defined by the user:

```
struct fb_var_screeninfo {
	__u32 xres; /* visible resolution */
	__u32 yres;
	__u32 xres_virtual; /* virtual resolution */
	__u32 yres_virtual;
	__u32 xoffset; /* offset from virtual to visible resolution */
	__u32 yoffset;
	__u32 bits_per_pixel; /* # of bits needed to hold a pixel */
	[...]
/* Timing: All values in pixclocks, except pixclock (of course) */
	__u32 pixclock;		/* pixel clock in ps (pico seconds) */
	__u32 left_margin;	/* time from sync to picture */
	__u32 right_margin;	/* time from picture to sync */
	__u32 upper_margin;	/* time from sync to picture */
	__u32 lower_margin;
	__u32 hsync_len;	/* length of horizontal sync */
	__u32 vsync_len;	/* length of vertical sync */
	__u32 rotate;		/* angle we rotate counter clockwise */
};
```
2. Prorerties, that are fixed either by manufacturer or developer and can't be
	changed. Generally the hardware information. A good example of this is
	the start of the framebuffer memory, which cannot change, even by the
	user-space program. The kernel holds such information in an instance of
	the `struct fb_fix_screeninfo` structure:
```
struct fb_fix_screeninfo {
	char id[16];		/* identification string example "TT Builtin" */
	unsigned long smem_start;	/* Start of frame buffer mem */
					/* (physical address) */
	__u32 smem_len;/* Length of frame buffer mem */
	__u32 type;		/* see FB_TYPE_* */
	__u32 type_aux;		/* Interleave for interleaved Planes */
	__u32 visual;		/* see FB_VISUAL_* */
	__u16 xpanstep;		/* zero if no hardware panning */
	__u16 ypanstep;		/* zero if no hardware panning */
	__u16 ywrapstep;	/* zero if no hardware ywrap */
	__u32 line_length;	/* length of a line in bytes */
	unsigned long mmio_start; /* Start of Memory Mapped I/O
				   *(physical address) */
	__u32 mmio_len;		/* Length of Memory Mapped I/O */
	__u32 accel;		/* Indicate to driver which */
				/* specific chip/card we have */
	__u16 capabilities; /* see FB_CAP_* */
};
```
3. The `struct fb_cmap` structure specifies the color map, which is used to store
	the user's- definition of colors in a manner the kernel can understand,
	in order to send it to the underlying video hardware. You can use this
	structure to define the RGB ratio that you defire for different colors:
```
struct fb_cmap {
	__u32 start;	/* First entry */
	__u32 len;	/* Number of entries */
	__u16 *red;	/* Red values */
	__u16 *green;	/* Green values */
	__u16 *transp;	/* Transparrency */
};
```
4. The `struct fb_info` structure represents the framebuffer itself and it's the
	main data structure of framebuffer drivers. Unlike the other preceding
	structures, `fb_info` exists only in the kernel and is not part of the
	user-space framebuffer API:
```
struct fb_info {
	[...]
	struct fb_var_screeninfo var;	/* Variable screen information.
					 * Discussed earlier. */
	struct fb_fix_screeninfo fix;	/* Fixed screen information. */
	struct fb_cmap cmap;		/ Color map. */
	struct fb_ops *fbops;		/* Driver operations.*/
	char __iomem *screen_base;	/* Frame buffer's virtual address */
	unsigned long screen_size;	/* Frame buffer's size */
	[...]
	struct device *device;		/* This is the parent */
	struct device *dev;		/* This is this fb device */
#ifdef CONFIG_FB_BACKLIGHT
	/* assigned backlight device */
	/* set before framebuffer registration,
	 * remove after unregister */
	struct backlight_device *bl_dev;
	/* Backlight level curve */
	struct mutex bl_curve_mutex;
	u8 bl_curve[FB_BACKLIGHT_LEVELS];
#endif
[...]
	void *par; /* Pointer to private memory */
};
```
The `struct fb_info` structure should be always allocated dynamically, using a
`framebuffer_alloc()`, which is a kernel (framebuffer core) helper function to
allocate memory for framebuffer devices, along woith the private data memory:
```
struct fb_info *framebuffer_alloc(size_t size, struct device *dev);

void framebuffer_release(struct fb_info *info);
```
Once set up, a framebuffer should be (un)registered with the kernel using funcs:
```
int register_framebuffer(struct fb_info, *fb_info);

int unregister_framebuffer(struct fb_info *fb_info);
```
Allocation and registering should be done during device probing, whereas unregis
tering and deallocation should be done from within the drivers `remove()` func.

# device methods

In the `struct fb_info` structure there is a `.fbops` filed, which is an instance
of the `struct fb_ops` structure. This structure contains the collection of func
tions that are needed to perform some opeartions of the framebuffer device. These
are entry points for __fbdev__ and __fbcon__ tools. The following definition is:
```
struct fb_ops {
	struct module *owner;
	int (*fb_open)(struct fb_info *info, int user);
	int (*fb_release)(struct fb_info *info, int user);

	/* For framebuffers with strange nonlinear layouts or that do not work
	 * with normal memory mapped access:
	 */
	ssize_t (*fb_read)(struct fb_info *info, char __user *buf,
			size_t count, loff_t *ppos);
	ssize_t (*fb_write)(struct fb_info *info, const char __user *buf,
			size_t count, loff_t *ppos);

	/* Checks var and eventually tweaks it to something supported
	 * DO NOT MODIFY PAR
	 */
	int (*fb_check_var)(struct fb_var_screeninfo *var, strcut fb_info *info);

	/* set the video mode according to info->var */
	int (*fb_set_par)(struct fb_info *info);

	/* set color register */
	int (*fb_setcolreg)(unsigned regno, unsigned red, unsiged green,
			unsigned blue, unsigned transp, struct fb_info *info);

	/* set color registers in batch */
	int (*fb_setcmap)(struct fb_cmap *cmap, struct fb_info *info);

	/* blank display */
	int (*fb_blank)(int blank_mode, struct fb_info *info);

	/* pan display */
	int (*fb_pan_display)(struct fb_var_screeninfo *var, struct fb_info *info);

	/* Draws a rectangle */
	void (*fb_fillrect)(struct fb_info *info, const struct fb_fillrect *rect);

	/* Copy data from area to another */
	void (*fb_copyarea)(struct fb_info *info, const struct fb_copyarea *region);

	/* Draws a image to the display */
	void (*fb_imageblit)(struct fb_info *info, const struct fb_image *image);

	/* Draws cursor */
	int (*fb_cursor)(struct fb_info *info, struct fb_cursor *cursor);

	/* wait for blit idle, optional */
	int (*fb_sync)(struct fb_info *info);

	/* perform fb specific ioctl (optional) */
	int (*fb_ioctl)(struct fb_info *info, unsigned int cmd, unsigned long arg);

	/* Handle 32bit compat ioctl (optional) */
	int (*fb_compat_ioctl)(struct fb_info *info, unsigned cmd, unsigned long arg);

	/* Perform fb specific mmap */
	int (*fb_mmap)(struct fb_info *info, struct vm_area_struct *vma);

	/* Get capability giver var */
	void (*fb_get_caps(struct fb_info *info, struct fb_blit_caps *caps,
			struct fb_var_screeninfo *var);

	/* Teardown any resources to do with this framebuffer */
	void (*fb_destroy)(struct fb_info *info);

	[...]
};
```
# Framebuffer Driver methods

Drivers shurely consist of `probe()` and `remove()` functions. Prior to going fu
rther with these methods descriptions, let us set up our `fb_ops` structure:
```
static struct fb_ops myfb_ops = {
	.owner		= THISMODULE,
	.fb_check_par	= myfb_check_var,
	.fb_set_par	= myfb_set_par,
	.fb_setcolreg	= myfb_setcolreg,
	.fb_fillrect	= cfb_fillrect,		/* Those three hooks are  */
	.fb_copyarea	= cfb_copyarea,		/* non-accelerated and	  */
	.fb_imageblit	= cfb_imageblit,	/* are provided by kernel */
	.fb_blank	= myfb_blank,
};
```
The driver probe function is in change of initializating the hardware, creating
the `struct fb_info` structure using the `framebuffer_alloc()` function, and us
ing `register_framebuffer()` on it. You can use a memory-mapped approach for yo
ur driver, or non-memory-map, if you screen is sitting on SPi buses, in this ca
se, the bus-specific routines should be used:
```
static int myfg_probe(struct platform_device *pdev)
{
	struct fb_info *info;
	struct resource *res;
	[...]

	dev_info(&pdev->dev, "My framebuffer driver"\n);
	/*
	 * Query resource, like DMA channels, I/O memory,
	 * regulators, and so on.
	 */
	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	if( !res )
		return -ENODEV;

	/* use request_mem_region(), ioremap() and so on */
	pwr = regulator_get(&pdev->dev, "lcd");

	info = framebuffer_alloc(sizeof(struct my_private_struct), &pdev->dev);
	if( !info )
		return -ENOMEM;

	/* Device init and default info value */
	[...]
	info->fbops = &myfb_ops;

	/* Clock setup, using devm_clk_get() and so on */
	[...]

	/* DMA setup, using dma_alloc_coherent() and so on */
	[...]

	/* Register with the kernel */
	ret = register framebuffer(info);

	hardware_enable_controller(my_private_struct);

	return 0;
}
```
And, of course, the `remove()` function sould release all acquired resources:
```
static int myfb_remove(struct platform device *pdev)
{
	/* iounmap() memory and release_mem_region() */
	[...]
	/* reverse DMA, dma_free*() */
	[...]
	hardware_disable_controller(fbi);

	/* first unregister, and then free the memory */
	inregister_framebuffer(info);
	framebuffer_release(info);

	return 0;
}
```
# Detailed `struct fb_ops`

Note that there is a good example of existing linux kernel framebuffer driver,
you can use this as a template or understanding the basic driver structions and
functions (look at `drivers/video/fbdev/vfb.c`). Also, see `/Documentation/fb/*`

## Checking information

For checking framebuffer parameters use the `fb_ops->fb_check_var` with prototype
```
int (*fb_check_var)(struct fb_var_screeninfo *var, struct fb_info *info);
```
This function should check framebuffer variable parameters and adjust it to valid
values. `var` represents the framebuffer variable parametersm which should be che
cked and adjusted, see the example:
```
static int myfb_check_var(struct fb_var_screeninfo *var,
struct fb_info *info)
{
	if (var->xres_virtual < var->xres)
	var->xres_virtual = var->xres;
	if (var->yres_virtual < var->yres)
	var->yres_virtual = var->yres;
	if ((var->bits_per_pixel != 32) && (var->bits_per_pixel != 24) &&
		(var->bits_per_pixel != 16) && (var->bits_per_pixel != 12) &&
		(var->bits_per_pixel != 8))
		var->bits_per_pixel = 16;

	switch (var->bits_per_pixel) {
	case 8:
		/* Adjust red*/
		var->red.length = 3;
		var->red.offset = 5;
		var->red.msb_right = 0;
		/*adjust green*/
		var->green.length = 3;
		var->green.offset = 2;
		var->green.msb_right = 0;
		/* adjust blue */
		var->blue.length = 2;
		var->blue.offset = 0;
		var->blue.msb_right = 0;
		/* Adjust transparency */
		var->transp.length = 0;
		var->transp.offset = 0;
		var->transp.msb_right = 0;
		break;
	case 16:
		[...]
		break;
	case 24:
		[...]
		break;
	case 32:
		var->red.length = 8;
		var->red.offset = 16;
		var->red.msb_right = 0;
		var->green.length = 8;
		var->green.offset = 8;
		var->green.msb_right = 0;
		var->blue.length = 8;
		var->blue.offset = 0;
		var->blue.msb_right = 0;
		var->transp.length = 8;
		var->transp.offset = 24;
		var->transp.msb_right = 0;
		break;
	}
	/*
	 * Any other field in *var* can be adjusted
	 * like var->xres, var->yres, var->bits_per_pixel,
	 * var->pixclock and so on.
	 */
	return 0;
}
```

## Setting the controller's parameters

The `fb_ops->fb_set_par` hook another hardware-specific hook, responsible for
sending parameters to the hardware. It programs the hardware based on user se
ttings (`info->var`).
```
static int myfb_set_par(struct fb_info *info)
{
	struct fb_var_screeninfo *var = *info->var;

	/* Make some compute or other sanity check */
	[...]

	/*
	 * This function writes value to the hardware,
	 * in the appropriate registers:
	 */
	 set_controller_vars(var, info);

	 return 0;
}
```

## Screen blanking

the `fb_ops->fb_blank` hook is a hardware-specific hook, responsible for screen
blanking. Its prototype as follows:
```
int (*fb_blank)(int blank_mode, struct fb_info *info);
```
The `blank_mode` parameter is one of the following values:
```
enum {
	/* screen: unblanked, hsync: on, vsync: on */
	FB_BLANK_UNBLANK = VESA_NO_BLANKING,

	/* screen: blanked, hsync: on, vsync: on */
	FB_BLANK_NORMAL = VESA_NO_BLANKING + 1,

	/* screen: blanked, hsync: on, vsync: off */
	FB_BLANK_VSYNC_SUSPEND = VESA_VSYNC_SUSPEND + 1,

	/* screen: blanked, hsync: off, vsync: on */
	FB_BLANK_HSYNC_SUSPEND = VESA_HSYNC_SUSPEND + 1,

	/* screen: blanked, hsync: off, vsync: off */
	FB_BLANK_POWERDOWN = VESA_POWERDOWN + 1
};
```
The usual way of doing a blank display is to do a `switch case` on the `blank_mode`
```
static int myfb_blank(int blank_mode, struct fb_info *info)
{
	pr_debug("fb_blank: blank=%d\n", blank);

	switch(blank) {
	case FB_BLANK_POWERDOWN:
	case FB_BLANK_VSYNC_SUSPEND:
	case FB_BLANK_NORMAL:
		myfb_disable_controller(fbi);
		break;
	case FB_BLANK_UNBLANK:
		myfb_enable_controller(fbi);
		break;
	}
	return 0;
}
```
The blanking operation should disable the controller, stop its colocks and power
it down. Unblanking should perform the reverse operations.

## Accelerated methods

User video operations, such as blending, stretching, moving bitmaps. or dynamic
gradient generation, are all heave-duty tasks. The require graphics acceleration
to obtain acceptable performance. You can implement framebuffer-accelerated meth
ods using the following fields of the `struct fb_ops` structure:
```
.fb_imageblit() /* drows an image on the display and is very useful */
.fb_copyarea()	/* copies a rectangular area from one to another screen region */
.fb_fillrect()	/* fills in an optimized manner a rectangle with pxl lines */

/* If there is no hardware acceleration, use optimized kernel-generic routines */
.cfb_imageblit() /* the kernel uses ot to output a logo to the screen during startup */
.cfb_fillrect()	 /* core non-accelerated method for achieving ops of the same name */
.cfb_copyarea()	 /* this is for area copy operations */
```
# Conlcusion:

In order to write framebuffer driver, you have to do the following:
1. Fill a `struct fb_var_screen info` structure in order to provide information
	on framebuffer variable properties. Those properties can be changed in us
2. fill a `struct fb_fix_screeninfo` to provide fixed hardware parameters
3. Set up a `struct fb_ops` structure, providing nessesary callbacks for fb-core
4. Still in the `struct fb_ops` provide accelerated functions. (generic or own)
5. Set up a `struct fb_info` structure, feeding it with structures filled in the
	previous steps and call `register_framebuffer()` on it in order to have
	it registered with the kernel.
