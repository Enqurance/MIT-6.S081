# MIT-6.S081

## QEMU运行前环境变量配置命令

```bash
export RISCV_HOME=/opt/riscv-gnu-toolchain
export PATH=${PATH}:${RISCV_HOME}/bin
source ~/.zshrc
```

## QEMU启动GDB进行调试

开启一个shell，执行：

```shell
riscv64-unknown-elf-gdb
```

再开启一个shell，执行：

```shell
make qemu-gdb
```

在ARM架构的机器上，偶尔会出现no symbol table is loaded等情况。在MacOS 14的机器上，我通过如下方式成功开启GDB调试：

- 配置环境变量

  ```shell
  export RISCV_HOME=/opt/riscv-gnu-toolchain
  export PATH=${PATH}:${RISCV_HOME}/bin
  source ~/.zshrc
  ```

- 开启一个shell，执行

  ```shell
  make qemu-gdb
  ```

- 在启动GDB的时候指定Kernel文件

  ```shell
  riscv64-unknown-elf-gdb kernel/kernel
  ```

  进入GDB后，手动连接到QEMU的shell

  ```shell
  target remote 127.0.0.1:25501
  ```

  即可开始插入断点进行调试。
