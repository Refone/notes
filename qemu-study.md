[TOC]

# Qemu 学习笔记

## Linux Direct Boot <sup>[link](https://qemu.weilnetz.de/doc/qemu-doc.html#direct_005flinux_005fboot)</sup>

```powershell
qemu-system-i386 -kernel arch/i386/boot/bzImage -hda root-2.4.20.img \
                 -initrd "..."
                 -append "root=/dev/hda console=ttyS0" -nographic
```

## 参考资料

[Qemu Manual](https://qemu.weilnetz.de/doc/qemu-doc.html)

