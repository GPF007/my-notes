# Linux一些常用的命令

1. ssh通过设置-o可以保持连接，从而在闲停时不会被中断

   ```bash
   ssh -o ServerAliveInterval=60 user@sshserver
   #expample
   ssh -o ServerAliveInterval=60 gpf@192.168.0.1
   ```

   