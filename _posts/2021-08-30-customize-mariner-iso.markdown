---
layout: post
title:  "Customize a pre-built CBL-Mariner ISO"
date:   2021-08-30 17:08:57 +0000
categories: jekyll update
---
The steps for building CBL-Mainer images are detailed at https://github.com/microsoft/CBL-Mariner/blob/1.0/toolkit/docs/building/building.md.

Below is the command to use to customize a pre-built Mariner ISO Image.

Assume ***demo_iso-1.0.20210812.2240.iso*** is the name of the ISO that was built following the above docs.


```
# mount the ISO Image
sudo mount -o loop demo_iso-1.0.20210812.2240.iso /mnt

mkdir demo_iso_expanded
cp -r /mnt/* demo_iso_expanded/

# Make the required changes to the files under demo_iso_expanded dir


# Re-package the ISO with the the following command:
mkisofs -R -l -D -o <output_dir>/custom_demo_iso-1.0.20210812.2240.iso \
-b isolinux/isolinux.bin \
-c isolinux/boot.cat \
-no-emul-boot --boot-load-size 4 \
-boot-info-table \
-eltorito-alt-boot -e boot/grub2/efiboot.img  \
-no-emul-boot \
demo_iso_expanded

```
Check out [CBL-Mariner][CBL-Mariner] and [CBL-MarinerDemo][CBL-MarinerDemo] links for more info on how to build custom images of CBL-Mariner.

[CBL-Mariner]: https://github.com/microsoft/CBL-Mariner/blob/1.0/toolkit/docs/building/building.md
[CBL-MarinerDemo]: https://github.com/microsoft/CBL-MarinerDemo
