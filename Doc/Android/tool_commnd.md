# ADB

```
adb reboot
adb shell
adb devices
adb install apk
adb shell pm clear <PackageName> //清除应用数据
```
# keyTool

获取应用的md5、sha1等信息
```
keytool -printcert -file platform.x509.pem 
keytool -printcert -jarfile xxx.apk //检查apk
keytool -printcert -file xxxx.RSA //解压apk获取rsa文件
```

# Git

```
git cherry-pick 7c32be61 合并某一次commit的提交
```