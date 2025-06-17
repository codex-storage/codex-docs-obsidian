
I am using Parallels Desktop.

In order to be able to compile I needed to increase ==the amount of RAM available== to my VM. Default was `2048 MB` which was too little - VM was hanging and/or the build process crashed (`killed`). I set it to `8192 MB`:

### GCC

We are currently using GCC-14. What you have on your Ubuntu is most probably GCC-13. So first to install GCC-14:

```bash
sudo apt install gcc-14
which gcc-14
/usr/bin/gcc-14
```

And then to make sure it is used as default `gcc`:

```bash
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-14 10
gcc --version
gcc (Ubuntu 14.2.0-4ubuntu2~24.04) 14.2.0
Copyright (C) 2024 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
### Rust

Run the following command to install Rust (I took this command from our CI):
```bash
curl --proto '=https' --tlsv1.2 -sSf [https://sh.rustup.rs/](https://sh.rustup.rs/) | sh -s -- --default-toolchain=1.79.0 -y
```

>[!note]
>After installing Rust remember to either open new terminal window or source the correct Rust environment in your current terminal.

After that you should be able to build:

```nim
make -j2
make -j2 te
```

