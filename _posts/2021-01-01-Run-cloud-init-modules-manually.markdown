---
layout: post
title:  "Run cloud-init modules manually"
date:   2021-01-01 01:01:00 +0000
categories: jekyll update
---
Following command will remove all the dynamic data from previous runs and initialize cloud-init.

```
sudo rm -rf /var/lib/cloud/* ;  sudo cloud-init init
```

Use the following command to run a module in cloud-init manually.

```
sudo cloud-init single --name <module_name>
sudo cloud-init single --name cc_timezone
```

All the cloud-init modules are located in /usr/lib/pythonx.x/dist-packages/cloudinit/config/ directory.

To run a particular set of modules in a cloud-init stage

```
sudo cloud-init modules --mode <stage_name>
```
Example:
```
sudo cloud-init modules --mode config
```
