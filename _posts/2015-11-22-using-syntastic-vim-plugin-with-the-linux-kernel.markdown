---
layout: post
title: Using Syntastic Vim Plugin With the Linux Kernel
modified:
categories:
description:
tags: []
image:
  feature:
  credit:
  creditlink:
comments:
share:
date: 2015-11-22T17:42:56+02:00
---

For those unfamilliar with the matters, I am using [Janus](https://github.com/carlhuda/janus) as my main editor, a Vim distribution with various plugins bundled, which make it behave more like an IDE. The component responsible for syntax checking is called [Syntastic](https://github.com/scrooloose/syntastic).

By default, Syntastic checker won't work out of the box with the Linux kernel sources because of the complex flags and include directories used, that Syntastic doesn't know how to reproduce when invoking `gcc`. By inspecting the Linux `Makefile`, however, the proper parameters for Syntastic can be obtained.

## Setup

This section is written mainly for people who are not already working on a particular project, so we can have common grounds for this tutorial. However, if you already have your basic kernel setup done, feel free to skip to the `Makefile inspection` section, and the rest should apply to you too.

First of all, we will have to build something using the Linux kernel Makefile. We can either do this in-tree, executing `make` in the root directory of the Linux sources, or write an out-of-tree kernel module, which I will further assume.

If you don't know how to build an out-of-tree module, you will need two special files (if you already have the sources set up for a kernel module, feel free to skip this chapter):

* a `Makefile` containing:

{% highlight bash %}
KDIR = /usr/src/linux-so2
kbuild:
     make -C $(KDIR) M=`pwd`
{% endhighlight %}

* a `Kbuild` file with contents in the likes of:

{% highlight bash %}
EXTRA_CFLAGS = -Wall -g
obj-m = my-module.o
{% endhighlight %}

Of course, you also need some kernel sources (which you can get from [here](https://github.com/torvalds/linux)). In this example, I am using some Linux 3.13 sources symbolically linked to the folder `/usr/src/linux-so2`:

{% highlight bash %}
# ln -s /home/aldi/so2/linux-3.13 /usr/src/linux-so2
{% endhighlight %}

In short, what is happening is that, by invoking Makefile, it changes the directory to that of the kernel source (`KDIR`) and invokes the Makefile found there. The Linux kernel Makefile is a huge beast that, amongst other things, checks for the `M` environment variable (set by our module to `pwd`, the current folder) so that it can return to our folder when compiling our module.
The second file (`Kbuild`) contains further details about our module (how it needs to be compiled, which source files are to be used, etc).

Just so we can check functionality easier, write a simple module that contains an error (missing semicolon in `my_init`):

{% highlight c %}
#include <linux/init.h>   /* for __init and __exit */
#include <linux/module.h> /* for module_init and module_exit */
#include <linux/printk.h> /* for printk call */

MODULE_AUTHOR("Syntastic");
MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Syntastic test module");

static int __init my_init(void)
{
        printk(KERN_DEBUG "It works!\n")    /* missing semicolon */
        return 0;
}

static void __exit my_exit(void)
{
        printk(KERN_DEBUG "Goodbye!\n");
}

module_init(my_init);
module_exit(my_exit);
{% endhighlight %}

Save the module to a C file:

    :w my-module.c

## Makefile inspection

The `Makefile` of our module does little more than call the kernel one, so we shall explore that next. What we're looking to find are the `CFLAGS` that gcc is using, so we can copy and pass them to Syntastic and it can emulate what gcc is doing.

It turns out that the `CFLAGS` are dynamically set, so they can't really be determined by simply reading the `Makefile`, but the following comment is useful:

    # Use 'make V=1' to see the full commands
In our module folder, we can activate verbose output for module compilation. giving us access to all parameters used by gcc. A sample output may be:

{% highlight bash %}
$ make V=1
make -C /usr/src/linux-so2 M=/home/aldi/so2/my-module
make[1]: Entering directory `/home/aldi/so2/linux-3.13'
mkdir -p /home/aldi/so2/my-module/.tmp_versions ; rm -f /home/aldi/so2/my-module/.tmp_versions/*
make -f scripts/Makefile.build obj=/home/aldi/so2/my-module
  gcc -Wp,-MD,/home/aldi/so2/my-module/.my-module.o.d  -nostdinc -isystem /usr/lib/gcc/x86_64-linux-gnu/4.8/include -I/home/aldi/so2/linux-3.13/arch/x86/include -Iarch/x86/include/generated  -Iinclude -I/home/aldi/so2/linux-3.13/arch/x86/include/uapi -Iarch/x86/include/generated/uapi -I/home/aldi/so2/linux-3.13/include/uapi -Iinclude/generated/uapi -include /home/aldi/so2/linux-3.13/include/linux/kconfig.h -D__KERNEL__ -Wall -Wundef -Wstrict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -Werror-implicit-function-declaration -Wno-format-security -fno-delete-null-pointer-checks -O2 -m32 -msoft-float -mregparm=3 -freg-struct-return -mno-mmx -mno-sse -fno-pic -mpreferred-stack-boundary=2 -march=i686 -maccumulate-outgoing-args -Wa,-mtune=generic32 -ffreestanding -DCONFIG_AS_CFI=1 -DCONFIG_AS_CFI_SIGNAL_FRAME=1 -DCONFIG_AS_CFI_SECTIONS=1 -DCONFIG_AS_AVX=1 -DCONFIG_AS_AVX2=1 -pipe -Wno-sign-compare -fno-asynchronous-unwind-tables -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -Wframe-larger-than=2048 -fno-stack-protector -Wno-unused-but-set-variable -fno-omit-frame-pointer -fno-optimize-sibling-calls -g -pg -Wdeclaration-after-statement -Wno-pointer-sign -fno-strict-overflow -fconserve-stack -Werror=implicit-int -Werror=strict-prototypes -DCC_HAVE_ASM_GOTO -Wall -g -Wno-unused  -DMODULE  -D"KBUILD_STR(s)=#s" -D"KBUILD_BASENAME=KBUILD_STR(my-module)"  -D"KBUILD_MODNAME=KBUILD_STR(my-module)" -c -o /home/aldi/so2/my-module/my-module.o /home/aldi/so2/my-module/my-module.c
  if [ "-pg" = "-pg" ]; then if [ /home/aldi/so2/my-module.o != "scripts/mod/empty.o" ]; then /home/aldi/so2/linux-3.13/scripts/recordmcount  "/home/aldi/so2/my-module/my-module.o"; fi; fi;
(cat /dev/null;   echo kernel//home/aldi/so2/my-module/my-module.ko;) > /home/aldi/so2/my-module/modules.order
make -f /home/aldi/so2/linux-3.13/scripts/Makefile.modpost
  find /home/aldi/so2/my-module/.tmp_versions -name '*.mod' | xargs -r grep -h '\.ko$' | sort -u | sed 's/\.ko$/.o/' | scripts/mod/modpost   -i /home/aldi/so2/linux-3.13/Module.symvers -I /home/aldi/so2/my-module/Module.symvers  -o /home/aldi/so2/my-module/Module.symvers -S -w  -s -T -
  gcc -Wp,-MD,/home/aldi/so2/my-module/.my-module.mod.o.d  -nostdinc -isystem /usr/lib/gcc/x86_64-linux-gnu/4.8/include -I/home/aldi/so2/linux-3.13/arch/x86/include -Iarch/x86/include/generated  -Iinclude -I/home/aldi/so2/linux-3.13/arch/x86/include/uapi -Iarch/x86/include/generated/uapi -I/home/aldi/so2/linux-3.13/include/uapi -Iinclude/generated/uapi -include /home/aldi/so2/linux-3.13/include/linux/kconfig.h -D__KERNEL__ -Wall -Wundef -Wstrict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -Werror-implicit-function-declaration -Wno-format-security -fno-delete-null-pointer-checks -O2 -m32 -msoft-float -mregparm=3 -freg-struct-return -mno-mmx -mno-sse -fno-pic -mpreferred-stack-boundary=2 -march=i686 -maccumulate-outgoing-args -Wa,-mtune=generic32 -ffreestanding -DCONFIG_AS_CFI=1 -DCONFIG_AS_CFI_SIGNAL_FRAME=1 -DCONFIG_AS_CFI_SECTIONS=1 -DCONFIG_AS_AVX=1 -DCONFIG_AS_AVX2=1 -pipe -Wno-sign-compare -fno-asynchronous-unwind-tables -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -Wframe-larger-than=2048 -fno-stack-protector -Wno-unused-but-set-variable -fno-omit-frame-pointer -fno-optimize-sibling-calls -g -pg -Wdeclaration-after-statement -Wno-pointer-sign -fno-strict-overflow -fconserve-stack -Werror=implicit-int -Werror=strict-prototypes -DCC_HAVE_ASM_GOTO -Wall -g -Wno-unused  -D"KBUILD_STR(s)=#s" -D"KBUILD_BASENAME=KBUILD_STR(my-module.mod)"  -D"KBUILD_MODNAME=KBUILD_STR(my-module)" -DMODULE  -c -o /home/aldi/so2/my-module/my-module.mod.o /home/aldi/so2/my-module/my-module.mod.c
  ld -r -m elf_i386 -T /home/aldi/so2/linux-3.13/scripts/module-common.lds --build-id  -o /home/aldi/so2/my-module/my-module.ko /home/aldi/so2/my-module/my-module.o /home/aldi/so2/my-module/my-module.mod.o
make[1]: Leaving directory `/home/aldi/so2/linux-3.13'
{% endhighlight %}

## Makefile output parsing

From the above generated output, only parts of one line are of interest to us, the one in which our source code (`/home/aldi/so2/my-module/my-module.c`) is turned into object code. If these parameters are not provided, then Syntastic will complain about include files not found, or duplicate file names in different folders. So we need to copy the `CFLAGS` of the first gcc command into a file called `.syntastic_c_config` (note: they're not marked as `CFLAGS`, simply select everything from the second parameter until you hit `-c`):

{% highlight bash %}
-nostdinc -isystem /usr/lib/gcc/x86_64-linux-gnu/4.8/include -I/home/aldi/so2/linux-3.13/arch/x86/include -Iarch/x86/include/generated  -Iinclude -I/home/aldi/so2/linux-3.13/arch/x86/include/uapi -Iarch/x86/include/generated/uapi -I/home/aldi/so2/linux-3.13/include/uapi -Iinclude/generated/uapi -include /home/aldi/so2/linux-3.13/include/linux/kconfig.h -D__KERNEL__ -Wall -Wundef -Wstrict-prototypes -Wno-trigraphs -fno-strict-aliasing -fno-common -Werror-implicit-function-declaration -Wno-format-security -fno-delete-null-pointer-checks -O2 -m32 -msoft-float -mregparm=3 -freg-struct-return -mno-mmx -mno-sse -fno-pic -mpreferred-stack-boundary=2 -march=i686 -maccumulate-outgoing-args -Wa,-mtune=generic32 -ffreestanding -DCONFIG_AS_CFI=1 -DCONFIG_AS_CFI_SIGNAL_FRAME=1 -DCONFIG_AS_CFI_SECTIONS=1 -DCONFIG_AS_AVX=1 -DCONFIG_AS_AVX2=1 -pipe -Wno-sign-compare -fno-asynchronous-unwind-tables -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -Wframe-larger-than=2048 -fno-stack-protector -Wno-unused-but-set-variable -fno-omit-frame-pointer -fno-optimize-sibling-calls -g -pg -Wdeclaration-after-statement -Wno-pointer-sign -fno-strict-overflow -fconserve-stack -Werror=implicit-int -Werror=strict-prototypes -DCC_HAVE_ASM_GOTO -Wall -g -Wno-unused  -DMODULE  -D"KBUILD_STR(s)=#s" -D"KBUILD_BASENAME=KBUILD_STR(my-module)"  -D"KBUILD_MODNAME=KBUILD_STR(my-module)"
{% endhighlight %}

Now we need to do some parsing on this output, because Syntastic wants each parameter to be on a separate line. We can `sed` the text, replacing each whitespace with a new line. In Vim:

        :%s/ /\r/g

We have two more things to do:
  * by default, Syntastic treats include paths as relative to the configuration file. Obviously, we need to change those paths so that they are absolute (prefix them with `$(KDIR)` as described above, in our case `/usr/linux/so2/`). We can also obtain this with `sed`. Replace `/usr/src/linux-so2/` with your appropriate `KDIR` in the following Vim command:

        :%s!\m^-I\zs\ze[^/]!/usr/src/linux-so2/!
  * Syntastic internally encloses some of its parameters to `gcc` in single quotes, so they survive shell expansion. However, it can be seen in the last 3 parameters (`KBUILD_BASENAME`, `KBUILD_MODNAME` and `KBUILD_STR`) include characters which have a special meaning to the shell - `(...)`, `#` - and have to be escaped. They are also defined with quotes in the Linux Makefile, for the same reason (to prevent the shell from expanding them), but Syntastic automatically does that too. The end result is that these parameters get escaped twice, which is not what we want (`gcc` will think that by `'-D"KBUILD_STR(s)=#s"'` we want to define the preprocessor macro named `KBUILD_STR(s)=#s`, instead of what is really intended). In order to solve this we must unquote everything that came from the Linux Makefile:

        :%s/"//g

The final version should look like this:

{% highlight bash %}
-nostdinc
-isystem
/usr/lib/gcc/x86_64-linux-gnu/4.8/include
-I/usr/src/linux-so2/arch/x86/include
-I/usr/src/linux-so2/arch/x86/include/generated

-I/usr/src/linux-so2/include
-I/usr/src/linux-so2/arch/x86/include/uapi
-I/usr/src/linux-so2/arch/x86/include/generated/uapi
-I/usr/src/linux-so2/include/uapi
-I/usr/src/linux-so2/include/generated/uapi
-include
/usr/src/linux-so2/include/linux/kconfig.h
-D__KERNEL__
-Wall
-Wundef
-Wstrict-prototypes
-Wno-trigraphs
-fno-strict-aliasing
-fno-common
-Werror-implicit-function-declaration
-Wno-format-security
-fno-delete-null-pointer-checks
-O2
-m32
-msoft-float
-mregparm=3
-freg-struct-return
-mno-mmx
-mno-sse
-fno-pic
-mpreferred-stack-boundary=2
-march=i686
-maccumulate-outgoing-args
-Wa,-mtune=generic32
-ffreestanding
-DCONFIG_AS_CFI=1
-DCONFIG_AS_CFI_SIGNAL_FRAME=1
-DCONFIG_AS_CFI_SECTIONS=1
-DCONFIG_AS_AVX=1
-DCONFIG_AS_AVX2=1
-pipe
-Wno-sign-compare
-fno-asynchronous-unwind-tables
-mno-sse
-mno-mmx
-mno-sse2
-mno-3dnow
-mno-avx
-Wframe-larger-than=2048
-fno-stack-protector
-Wno-unused-but-set-variable
-fno-omit-frame-pointer
-fno-optimize-sibling-calls
-g
-pg
-Wdeclaration-after-statement
-Wno-pointer-sign
-fno-strict-overflow
-fconserve-stack
-Werror=implicit-int
-Werror=strict-prototypes
-DCC_HAVE_ASM_GOTO
-Wall
-g

-DMODULE

-DKBUILD_STR(s)=#s
-DKBUILD_BASENAME=my-module
-DKBUILD_MODNAME=my-module
{% endhighlight %}

## Testing

We have to save this into a file named `.syntastic_c_config` and reference it in our `.vimrc` (or `.vimrc.after`, in the case of [Janus](https://github.com/carlhuda/janus), a Vim distro that I'm using):


{% highlight bash %}
    let g:syntastic_c_config_file = '.syntastic_c_config'
{% endhighlight %}

Note that this step can be omitted, since `.syntastic_c_config` is also the default name that Syntastic searches for.
By default, Syntastic starts looking for a config file in the current folder, then upwards until either it hits the root, or it finds a valid config file. As such, it makes sense to only place the configuration file where it is valid (in our example, in the root directory of the kernel stuff, `/home/aldi/so2/`), so it won't interfere with other projects.

Also, there are two more features that can cause Syntastic not to work properly with the Linux kernel. By default, it adds `-I. -I.. -Iinclude -Iincludes -I../include -I../includes` to the parameters it passes to `gcc`. Although this may do no harm, it can be deactivated with the following option:

{% highlight bash %}
    let g:syntastic_c_no_default_include_dirs = 1
{% endhighlight %}
Secondly, Syntastic adds the flag `-std=gnu99`, which has the possibility to cause problems. To disable it, use the option:

{% highlight bash %}
    let g:syntastic_c_compiler_options = ''
{% endhighlight %}

Now open the file `my-module.c` we wrote at the beginning. If you don't have `g:syntastic_check_on_open` enabled, then either type `:w` or `:SyntasticCheck`. By now, Syntastic should have run and found the error (missing semicolon).
If it seems like nothing happens, run `:Errors` in Vim to see the checker's output. You can also set `g:syntastic_debug` to 1 in your vimrc, and check what parameters got passed from Syntastic to gcc by using the command `:messages`.

Congratulations, if you have come this far then you now have Syntastic working with the Linux kernel sources!
![Screenshot](https://github.com/vladimiroltean/blog/blob/master/syntastic_vim_screenshot.png)

## Further reference

<https://github.com/scrooloose/syntastic/wiki/C%3A---gcc>
<https://github.com/scrooloose/syntastic/wiki/Configuration-Files>
