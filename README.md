# mod_dingaling
该项目从freeswitch-1.10.7复制而来的。

# 安装缺少的依赖包
```bash
apt-get install -y libapr1-dev libaprutil1-dev
```

## 步骤
1. 将 `mod_dingaling` 复制到 `src/mod/endpoints/`
2. 将 `libdingaling` 复制到 `lib/`
3. 修改 `configure.ac` 文件，在 `src/mod/endpoints/mod_alsa/Makefile` 的下面增加一行：`src/mod/endpoints/mod_dingaling/Makefile`
4. 执行 `./rebootstrap.sh` 重新生成 Makefile 等文件。

# 设置编译标志并配置FreeSWITCH
```bash
export LDFLAGS="-lapr-1 -laprutil-1"
./configure \
--with-apr=/usr --with-apr-util=/usr \
CPPFLAGS="-I/usr/include/apr-1.0" \
CFLAGS="-I/usr/include/apr-1.0"
```