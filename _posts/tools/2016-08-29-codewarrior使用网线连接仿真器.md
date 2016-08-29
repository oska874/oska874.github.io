freescale 的 codewarrior 仿真器支持 usb 和 ethernet 两种接口。默认使用 usb 接口烧录，但是很容易出问题，所以就选在使用网络模式。

1. 使用网络模式需要使用 micro usb 口供电，还需要在 codewarrior 中设置 `hardware connection` 为 `Ethernet` ，并填写仿真器的实际 IP 地址。
2. 在PC 上使用串口工具访问仿真器，输入命令 `netparam` 获取仿真的配置：
  
  ```
    core> netparam
    ethernet_address        = 00:04:9F:03:AE:EF
    bootconfig              = dhcp:FSL03AEEF (192.168.0.91)
    static_ip_address       = <none>
    static_dns_server       = <none>
    static_hosts            = <none>
    static_routes           = <none>
  ```
  默认情况下，仿真器使用 DHCP 自动获取 IP，也可以使用静态 ip ：
  
  ```
  netparam bootconfig static
  netparam add_host OS 192.168.2.24
  netparam static_ip_address 192.168.2.231
  ```
  这几条命令作用分别是：a. 设置为静态 IP 模式；b. 添加主机 OS 的 ip ；c. 设置仿真器的 ip。
  
现在就可以 pc 就可以通过网线连接仿真器了，而且此时 usb 串口也可以不在连接 PC 了，只要 usb 继续供电即可。
