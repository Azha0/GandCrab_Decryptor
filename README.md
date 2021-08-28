# 简介

> 本解密工具主要针对勒索病毒GandCrab v5.2的两个版本进行修复的。在分析GandCrab V5.2时，偶然发现v5.2版本就目前我所发现的有两个不同的版本，修复工作也考虑到两者的差异，进行兼容两者都修复。



## 两个同为V5.2版本的不同变种基本信息

|      样本名称      | 样本伪装图标 |               MD5                |
| :----------------: | :----------: | :------------------------------: |
| GandCrab V5.2（1） |  浅色文件夹  | DE46B3B7F13F12769524755BB0A105FE |
| GandCrab V5.2（2） |   Word文档   | 445DD888ED51E331FDCF2FA89199CCA6 |

GandCrab V5.2（1）和GandCrab V5.2（2）的图标如下图：

![image-20191204100427506](https://cdn.jsdelivr.net/gh/Azha0/ImgHosting@1.1/ImgFromBlog/01_样本分析报告/09_GandCrabV5.2解密工具/image-20191204100427506.png)

我所发现的两个GandCrab V5.2，作者的加密RSA私钥一样，加密手法一样，大体上没有什么太大的区别，==但在一些行为上有些许的差别，这些差别也影响了修复的方式==。

## GandCrab v5.2（1）与GandCrab V5.2(2)的差别

**注册表差异**

GandCrab V5.2(2）将随机生成的加密后缀保存到注册表：`HKLM(HKCU)\SOFTWARE\\ex_data\\data`中的`ext`中，并将加密的==密钥信息==（经过加密的用于加密本地RSA私钥的Salsa20的Key和IV，以及加密后的本地RSA私钥）保存在注册表：`HkLM(HKCU)\SOFTWARE\\keys_data\\data`中的`private`。

而GandCrab V5.2(1)==并没有将加密相关的信息写入注册表==，这将严重影响修复工具获取密钥信息（经过加密的用于加密本地RSA私钥的Salsa20的Key和IV，以及加密后的本地RSA私钥）的方法，导致==无法直接读取注册表获取密钥信息。==

**勒索说明差异**

从文本内容看勒索说明都是一样的，唯一的差别在于勒索说明的文件名不同，V5.2（1）为`MANUAL.txt`，V5.2（2）为`DECRYPT.txt`如图：

![image-20191204102908024](https://cdn.jsdelivr.net/gh/Azha0/ImgHosting@1.1/ImgFromBlog/01_样本分析报告/09_GandCrabV5.2解密工具/image-20191204102908024.png)

GandCrab V5.2（2）瑞星的详细分析报告： http://it.rising.com.cn/fanglesuo/19523.html 

# 加解密逻辑

## 加密逻辑

![GrandCrab](https://cdn.jsdelivr.net/gh/Azha0/ImgHosting@1.1/ImgFromBlog/01_样本分析报告/09_GandCrabV5.2解密工具/GrandCrab.png)

首先通过硬编码解密出Salsa20的key和IV，然后通过Salsa20解密出作者的RSA-2048的公钥（RSA2048PublicKey），然后生成本地的RSA-2048密钥对，再用随机生成的Salsa20LocRSAPriv的Key和IV去加密本地RSA的私钥（LocRSAPrivateKey)，然后再用作者的RSA-2048公钥（RSA2048PublicKey）加密随机生成的Salsa20LocRSAPriv的Key和IV，并将加密后的本地RSA私钥（LocRSAPrivateKey）和加密后的Salsa20LocRSAPriv的Key和IV共同保存起来，V5.2（1）将这些密钥数据经过Base64加密保存在勒索说明中，如图：

![image-20191204104848604](https://cdn.jsdelivr.net/gh/Azha0/ImgHosting@1.1/ImgFromBlog/01_样本分析报告/09_GandCrabV5.2解密工具/image-20191204104848604.png)

而V5.2（2）将这些密钥数据保存在注册表`HkLM(HKCU)\SOFTWARE\\keys_data\\data`中的`private`中。注册表中和解密后的勒索说明中的密钥数据如下图：

![Key](https://cdn.jsdelivr.net/gh/Azha0/ImgHosting@1.1/ImgFromBlog/01_样本分析报告/09_GandCrabV5.2解密工具/Key.png)

然后在随机生成Salsa20File的Key和IV去加密文件，并用本地生成的LocRSAPublicKey去加密Salsa20File的Key和IV，并将其保存在文件末尾`0x21C`的位置，文件中的密钥数据布局，如下图：

![image-20191204105430575](https://cdn.jsdelivr.net/gh/Azha0/ImgHosting@1.1/ImgFromBlog/01_样本分析报告/09_GandCrabV5.2解密工具/image-20191204105430575.png)

其中对加密文件时，有特殊的判断和不同的处理，作者给定了一个要加密的文件后缀表和一个不加密的文件后缀表。

- 当文件在要加密文件后缀表中，则全文加密。
- 当文件不在加密文件后缀表中，也不在不加密的文件后缀表中，则只加密前1M的数据，不全文加密。
- 当文件小于1M，并且不在不加密文件后缀表中，只加密当前数据大小。

加密后缀表：

```
.1st.602.docb.xlm.xlsx.xlsm.xltx.xltm.xlsb.xla.xlam.xll.xlw.ppt.pot.pps.pptx.pptm.potx.potm.ppam.ppsx.ppsm.sldx.sldm.xps.xls.xlt._doc.dotm._docx.abw.act.adoc.aim.ans.apkg.apt.asc.asc.ascii.ase.aty.awp.awt.aww.bad.bbs.bdp.bdr.bean.bib.bib.bibtex.bml.bna.boc.brx.btd.bzabw.calca.charset.chart.chord.cnm.cod.crwl.cws.cyi.dca.dfti.dgs.diz.dne.dot.doc.docm.dotx.docx.docxml.docz.dox.dropbox.dsc.dvi.dwd.dx.dxb.dxp.eio.eit.emf.eml.emlx.emulecollection.epp.err.err.etf.etx.euc.fadein.template.faq.fbl.fcf.fdf.fdr.fds.fdt.fdx.fdxt.fft.fgs.flr.fodt.fountain.fpt.frt.fwd.fwdn.gmd.gpd.gpn.gsd.gthr.gv.hbk.hht.hs.hwp.hwp.hz.idx.iil.ipf.ipspot.jarvis.jis.jnp.joe.jp1.jrtf.jtd.kes.klg.klg.knt.kon.kwd.latex.lbt.lis.lnt.log.lp2.lst.lst.ltr.ltx.lue.luf.lwp.lxfml.lyt.lyx.man.mbox.mcw.md5.me.mell.mellel.min.mnt.msg.mw.mwd.mwp.nb.ndoc.nfo.ngloss.njx.note.notes.now.nwctxt.nwm.nwp.ocr.odif.odm.odo.odt.ofl.opeico.openbsd.ort.ott.p7s.pages.pages-tef.pdpcmd.pfx.pjt.plain.plantuml.pmo.prt.prt.psw.pu.pvj.pvm.pwd.pwdp.pwdpl.pwi.pwr.qdl.qpf.rad.readme.rft.ris.rpt.rst.rtd.rtf.rtfd.rtx.run.rvf.rzk.rzn.saf.safetext.sam.sam.save.scc.scm.scriv.scrivx.sct.scw.sdm.sdoc.sdw.se.session.sgm.sig.skcard.sla.sla.gz.smf.sms.ssa.story.strings.stw.sty.sublime-project.sublime-workspace.sxg.sxw.tab.tab.tdf.tdf.template.tex.text.textclipping.thp.tlb.tm.tmd.tmdx.tmv.tmvx.tpc.trelby.tvj.txt.u3i.unauth.unx.uof.uot.upd.utf8.utxt.vct.vnt.vw.wbk.webdoc.wn.wp.wp4.wp5.wp6.wp7.wpa.wpd.wpd.wpd.wpl.wps.wps.wpt.wpt.wpw.wri.wsd.wtt.wtx.xbdoc.xbplate.xdl.xdl.xwp.xwp.xwp.xy.xy3.xyp.xyw.zabw.zrtf.zw
```

不加密后缀表：

```
.ani.cab.cpl.cur.diagcab.diagpkg.dll.drv.lock.hlp.ldf.icl.icns.ico.ics.lnk.key.idx.mod.mpa.msc.msp.msstyles.msu.nomedia.ocx.prf.rom.rtp.scr.shs.spl.sys.theme.themepack.exe.bat.cmd.gandcrab.KRAB.CRAB.zerophage_i_like_your_pictures
```

## 解密逻辑

由于GandCrab V5.2，根据我发现的两个版本的不同，进行兼容修复。

由于V5.2（2）将密钥数据写入了注册表，而V5.2（1）没有写注册表。也正因为V5.2（1）没有写注册表，所以在修复的时候需要输入一个被加密的目录来获取密钥。

在修复的时候，首先读取注册表密钥信息，当读取不到时，便会遍历当前目录去寻找勒索说明文件，并获取勒索说明中的密钥数据，并进行解密。

#  附加：去花脚本

GandCrab V5.2中有大量的花指令感染分析，其中每个函数入口点都有花指令，将导致IDA无法F5。我在分析时写了如下去花脚本，若需要再次分析GandCrab V5.2的样本时，可以使用。

```python
def patch_junkcode(addr):
    data = list(get_bytes(addr,28))
    if((ord(data[0]) == 0xB9) and (ord(data[1]) == 0xD2) and (ord(data[2]) == 0xC3) and ord(data[3]) == 0x01 and (ord(data[5]) == 0xE8) and (ord(data[6]) == 0x0B)):
			for i in range(0,28):
				patch_byte(addr+i,0x90)
base = 0x401000
len = 0x411000-base
for i in range(len):
	patch_junkcode(base+i)

AnalyzeArea(base, 0x411000) 
print 'Finished'
```
