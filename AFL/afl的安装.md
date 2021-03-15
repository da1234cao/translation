看仓库中的doc/INSTALL文档

```shell
# CC=/usr/bin/clang make --just-print
CC=/usr/bin/clang make
# 上上面这样，别运行make install。否则安装之后不好卸载

# 我使用stow进行管理安装：https://github.com/da1234cao/dotfiles
# CC=/usr/bin/clang PREFIX=/usr/local/stow/afl  make install --just-print
CC=/usr/bin/clang PREFIX=/usr/local/stow/afl  make
CC=/usr/bin/clang CXX=/usr/bin/clang++ PREFIX=/usr/local/stow/afl LLVM_CONFIG=/usr/bin/llvm-config-10  make

sudo PREFIX=/usr/local/stow/afl make install
cd /usr/local/stow
stow afl
```

