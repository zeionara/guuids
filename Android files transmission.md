# Android files transmission

To pass files between `linux desktop` and `android` device, make sure that your device is accessible:

```sh
adb devices
```

If not, then you need to adjust the device settings. If it is accessible, you can open `shell` there:

```sh
adb shell
```

Here is an example of a command for moving a file to android device:

```sh
sudo adb push movie.mp4 /storage/emulated/0/Movies/
```

To move file from device to your host OS use the command like this:

```sh
sudo adb pull /storage/emulated/0/Movies/movie.mp4 /mnt/windows/
```
