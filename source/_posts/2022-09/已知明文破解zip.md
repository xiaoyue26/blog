---
title: 已知明文破解zip
date: 2022-09-26 15:40:47
tag:
- 信息安全
- 网络攻防
- zip
- pkcrack
categories: 
- 网络
- 网络安全

---


# 已知明文攻击
当我们拿到一个用密码加密的zip文件的时候，如果我们恰好还知道压缩包里某个文件的明文，就可以使用pkcrack进行已知明文破解zip。
比如我们的目标是：`密文.zip`,
我们可以直接vi一下这个zip文件可以看到它里面到底有哪些文件。（即使是加密的也可以看到里面有什么文件）
比如这个压缩包里恰好有一个`明文.png`，（有些人喜欢在所有压缩包里放版权声明文件或者广告图，这样根据文件名在网上可以直接搜到明文文件）
我们可以把这个`明文.png`手动压缩一下得到`明文.zip`。
然后开始安装`pkcrack`然后破解。


mac下可以用brew安装一个pkcrack:
```shell script
brew install pkcrack
```

或者直接下载一个`pkcrack-1.2.2`：
http://www.password-crackers.com/en/category_98/program_24.html
下载以后在src里make一下就能得到可执行文件了。
然后就简单了：
```
cd src
make
cd ..

[user@ip pkcrack-1.2.2]# ./src/pkcrack  -C 密文.zip -c 明文.png -P 明文.zip -p 明文.png -d 结果.zip -a
Files read. Starting stage 1 on Mon Sep 26 15:40:09 2022
Generating 1st generation of possible key2_822346 values...done.
Found 4194304 possible key2-values.
Now we're trying to reduce these...
Lowest number: 947 values at offset 818742
Lowest number: 901 values at offset 818736
Lowest number: 872 values at offset 818720
Lowest number: 847 values at offset 818719
Lowest number: 838 values at offset 818625
Lowest number: 804 values at offset 818589
Lowest number: 745 values at offset 818571
Lowest number: 711 values at offset 818569
Lowest number: 677 values at offset 818366
Lowest number: 657 values at offset 818337
Lowest number: 656 values at offset 818294
Lowest number: 642 values at offset 818292
Lowest number: 589 values at offset 817979
Lowest number: 572 values at offset 817970
Lowest number: 568 values at offset 817966
Lowest number: 564 values at offset 817936
Lowest number: 537 values at offset 817934
Lowest number: 533 values at offset 817907
Lowest number: 530 values at offset 817896
Lowest number: 436 values at offset 817874
Lowest number: 398 values at offset 817862
Lowest number: 394 values at offset 817833
Lowest number: 353 values at offset 817830
Lowest number: 351 values at offset 817789
Lowest number: 328 values at offset 817787
Lowest number: 322 values at offset 817785
Lowest number: 306 values at offset 817784
Lowest number: 291 values at offset 817774
Lowest number: 281 values at offset 817761
Lowest number: 279 values at offset 805546
Lowest number: 253 values at offset 805545
Lowest number: 242 values at offset 805544
Lowest number: 215 values at offset 805519
Lowest number: 212 values at offset 798445
Lowest number: 211 values at offset 798421
Lowest number: 193 values at offset 798402
Lowest number: 187 values at offset 798401
Lowest number: 185 values at offset 798397
Lowest number: 179 values at offset 798395
Lowest number: 178 values at offset 798392
Lowest number: 176 values at offset 798390
Lowest number: 175 values at offset 798382
Lowest number: 174 values at offset 798380
Lowest number: 151 values at offset 798370
Lowest number: 139 values at offset 798369
Lowest number: 127 values at offset 709771
Lowest number: 114 values at offset 709769
Lowest number: 113 values at offset 709767
Lowest number: 110 values at offset 709766
Lowest number: 106 values at offset 709764
Lowest number: 103 values at offset 709752
Lowest number: 101 values at offset 709751
Lowest number: 99 values at offset 709750
Done. Left with 99 possible Values. bestOffset is 709750.
Stage 1 completed. Starting stage 2 on Mon Sep 26 15:40:52 2022
Ta-daaaaa! key0=71fc91ef, key1=d508d7a6, key2=3e05d364
Probabilistic test succeeded for 112601 bytes.
Ta-daaaaa! key0=71fc91ef, key1=d508d7a6, key2=3e05d364
Probabilistic test succeeded for 112601 bytes.
Ta-daaaaa! key0=71fc91ef, key1=d508d7a6, key2=3e05d364
Probabilistic test succeeded for 112601 bytes.
Stage 2 completed. Starting zipdecrypt on Mon Sep 26 15:40:53 2022
Decrypting 其他密文.txt (210b089a08f3aed2247acf25)... OK!
Decrypting 明文.png (a84583e0c35dc7faaa590237)... OK!
Finished on Mon Sep 26 15:40:53 2022
```

破解后的压缩包就生成到了结果.zip里。
