# Installing hyprland on ubuntu 22.04

## Clone the repos

```sh
git clone --recursive https://github.com/hyprwm/Hyprland
git clone https://github.com/hyprwm/aquamarine.git
git clone https://github.com/hyprwm/hyprwayland-scanner.git
```

## Install GCC 14

```sh
wget https://ftp.gnu.org/gnu/gcc/gcc-14.1.0/gcc-14.1.0.tar.gz
cd gcc-14.1.0/
sudo apt install -y libgmp-dev libmpfr-dev libmpc-dev build-essential
./configure -v --build=$(uname -m)-linux-gnu --host=$(uname -m)-linux-gnu --target=$(uname -m)-linux-gnu --prefix=/usr/local/gcc-14.1.0 --enable-checking=release --enable-languages=c,c++ --disable-multilib --program-suffix=-14.1.0
make -j32 -l36
sudo make install
```
