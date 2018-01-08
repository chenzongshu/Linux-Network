
有两台服务器,需要免密拷贝和登录,可以如下设置

在A上

```
cd ~/.ssh

ssh-keygen -t rsa -P ""  #然后一路回车到底

scp id_rsa.pub root@192.168.201.225:/root #把生成的公钥拷贝到B上
```

在B上

```
cd ~
cat id_rsa.pub >> .ssh/authorized_keys
```
这样,A就可以免密登录B,或者拷贝东西到B,如果需要B到A也这样,反过来设置下即可
