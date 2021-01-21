---
title: 完美解决方案：实现在big sur中新增文件目录下任意读写。（例如新增/data目录）
date: 2021-01-21 09:08:03
tags: bigsur trick osx
---

1. 先在home目录下创建一个可以读写的目录，例如/Users/david/data

2. 编辑

   ```bash
   sudo vim /etc/synthetic.conf
   ```

   

3. 在synthetic.conf文件中添加一行（注意：/Users/david/data是你自己创建的可读写的目录，可以自定义。用来做为/data实际存储的目录。

   ```
   data    /Users/david/data
   ```

   

   **中间的分隔符一定要是tab**

   重启后会创建一个/data的软链接，指向/Users/david/data)

