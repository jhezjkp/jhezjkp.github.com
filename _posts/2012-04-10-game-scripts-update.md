---
layout: post
title: "游戏服脚本批量更新"
description: ""
category: 游戏开发
tags: [python]
---
{% include JB/setup %}

因为供职于一家手机网游公司的缘故，我在兼做游戏开发的同时也时常做一些运营工作：更新游戏脚本。游戏么，经常有各种各样的活动，而且游戏服数量通常都不会少，所以每次更新游戏脚本都很纠结：rz...mv...cp...
#烦死了！！！！！！！！！！！
shell不行，恰好刚学了python，这东西很容易上手，有Java基础的童鞋花个一天时间通读一下[简明python教程](http://woodpecker.org.cn/abyteofpython_cn/chinese/)即可上手。废话少说，先整理一下需求：

* 支持多游戏服脚本批量更新(大部分情况下所有游戏服要上的活动都是相同的)
* 支持脚本异同比较
* 要求更新游戏服中的脚本前先备份当前脚本
* 支持日志，更新完成后要报告更新结果(多少脚本更新成功，多少脚本忽略了，如果有更新失败的脚本则显示具体的脚本名称)
* 如果更新成功删除更新源脚本

嗯，目前大概就想到了这些需求，以后想到了再说。有了明确的需求，实现起来就简单了，思路很简单:

1.  把要更新的脚本上传到服务器上的某个目录，这个目录就称为更新源，程序启动后先扫描更新源下有哪些脚本，把文件名放入一个字典(key为文件名,value为文件对象)，这个字典即我们的更新列表

2.  读取游戏服配置字典(key为游戏服名，value为该游戏服的脚本文件根目录)，循环更新各服脚本：
    * 扫描该游戏服脚本目录下的所有脚本，脚本更新列表中是否有同名脚本
    * 一旦在更新列表中找到同名脚本，则对这两个脚本进行hash，并比对hash值是否相同
    * 如果两个文件的hash值相同，则说明脚本文件无更新，忽略该脚本文件，并将忽略计数器加1
    * 如果两个文件的hash值不相同，说明该脚本文件有变更，则先备份原脚本文件，再从更新源中将同名脚本复制到该目录下，并将更新计数器加1
3.  所有服更新完后，检查更新计数器和忽略计数器，如果各服的成功更新数+忽略数=更新列表的脚本数则说明本次更新成功，可以将更新源中的脚本删除，否则则显示哪些服的哪些脚本更新失败。

以下为完整的更新脚本代码：

	#!/usr/bin/python
	# encoding=gbk
	
	'''
	游戏脚本批量更新程序
	'''
	
	import sys, os, shutil, time, datetime, logging, hashlib
	
	#更新源
	srcPath = r"f:\src"
	#更新目标位置
	#targetConfig = {"失落要塞(game4)":r"f:\data", "王者之城(game1)":r"f:\data1"}
	targetConfig = {"测试服":r"\\192.168.1.254\e$\tx\tx_g\data"}
	#脚本文件列表
	scripts = {}
	#脚本成功更新量计数字典
	updateResult = {}
	#脚本忽略更新计数字典
	ignoreResult = {}
	#更新日志
	log = "update.log"
	
	def initLogger():
		'''
		初始化日志配置
		'''
		logger = logging.getLogger()
		fileHandler = logging.FileHandler(log)
		streamHandler = logging.StreamHandler()
		fmt = logging.Formatter("%(asctime)s, %(message)s")
		fileHandler.setFormatter(fmt)
		logger.setLevel(logging.DEBUG)
		logger.addHandler(fileHandler)	
		logger.addHandler(streamHandler)
		return logger	
	
	def checkUpdateConfig():
		'''
		检查更新配置是否合法
		'''
		if not os.path.exists(srcPath):
			logger.error("更新源(%s)不存在", srcPath)
			sys.exit()
		for gameServer, targetPath in targetConfig.items():
			if not os.path.exists(targetPath):
				logger.error("%s所对应的路径%s不存在", gameServer, targetPath)
				sys.exit()
	
	def initScriptList():
		'''
		初始化需要更新的脚本列表
		'''
		logger.info("--------------- 初始化需要更新的脚本列表 ---------------")
		for f in os.listdir(srcPath):
			if os.path.isfile(os.path.join(srcPath, f)):		
				logger.info(f)
				scripts[str(f)] = f
		logger.info("--------------- 总计有%d个脚本需要更新 -----------------", len(scripts))
		return len(scripts)
		
	def hashFile(filePath):
		'''
		计算指定文件路径的hash值
		'''
		if os.path.exists(filePath) and os.path.isfile(filePath):
			with open(filePath, 'rb') as f:
				sha1obj = hashlib.sha1()
				sha1obj.update(f.read())
			return sha1obj.hexdigest()
			
	def cleanUp():
		'''
		删除更新成功的脚本
		'''	
		logger.info("--------------- 删除更新成功的脚本 ---------------")
		for script in scripts:		
			os.remove(srcPath+os.sep+script)
			logger.info("delete "+srcPath+os.sep+script)
		logger.info("------------- 更新成功的脚本清除完毕 -------------")
	
	def updateScript(gameServer, path, appendSuffix):
		for f in os.listdir(path):
			fullPath = os.path.join(path, f)
			if os.path.isfile(fullPath):			
				#如果待更新的文件中有该文件则进行文件(hash)比较
				if scripts.has_key(f):	
					srcLastTime = time.strftime('%Y-%m-%d %X', time.localtime(os.path.getmtime(srcPath+os.sep+f)))
					targetLastTime = time.strftime('%Y-%m-%d %X', time.localtime(os.path.getmtime(fullPath)))
					if hashFile(srcPath+os.sep+f) != hashFile(fullPath):					
						#如果文件不同则备份并更新
						try:							
							fileName = f.split(".")[0]
							fileSuffix = f.split(".")[-1]		
							#备份文件名	
							bak = fileName + appendSuffix + fileSuffix									
							#切换到当前目录
							os.chdir(path)
							#检查备份文件名是否已被占用
							if os.path.exists(bak):
								bak = fileName + datetime.datetime.now().strftime('_%Y%m%d_%H%M%S.') + fileSuffix	
							#重命名
							os.rename(f, bak)
							#复制文件
							shutil.copyfile(os.path.join(srcPath, f), os.path.join(path, f)) 
							#更新成功量
							updateResult[gameServer] += 1
							logger.info("%25s %s %s %s %s", f, srcLastTime, ">>>>>>", targetLastTime, os.path.join(path, f))
						except:
							logger.error(str(sys.exc_info()[0])+str(sys.exc_info()[1]))
					else:
						#忽略计数量
						ignoreResult[gameServer] += 1
						logger.info("%25s %s %s %s %s", f, srcLastTime, "======", targetLastTime, os.path.join(path, f))
			else:
				updateScript(gameServer, fullPath, appendSuffix)			
	
	if __name__ == "__main__":
		logger = initLogger()
		#检测配置
		checkUpdateConfig()
		#初始化更新脚本
		if initScriptList()==0:
			logger.info("更新源路径(%s)下没有需要更新的脚本", srcPath)
			sys.exit()
		#默认文件备份后缀
		appendSuffix = datetime.datetime.now().strftime('_%Y%m%d.')
		for gameServer, targetPath in targetConfig.items():
			updateResult[gameServer] = 0
			ignoreResult[gameServer] = 0
			logger.info("--------------- 开始更新%s(%s)脚本文件 ---------------", gameServer, targetPath)
			updateScript(gameServer, targetPath, appendSuffix)
			logger.info("--------------- %s(%s)脚本文件更新完成：总计%d个，成功%d个 忽略%d个 ---------------", gameServer, targetPath, len(scripts), updateResult[gameServer], ignoreResult[gameServer])
		
		#检查是否某个服脚本更新失败
		totalUpdateCount = 0		
		for gameServer, updateCount in updateResult.items():
			totalUpdateCount += updateCount
		totalIgnoreCount = 0
		for gameServer, ignoreCount in ignoreResult.items():
			totalIgnoreCount += ignoreCount
		if len(scripts) * len(updateResult) > totalUpdateCount+totalIgnoreCount:
			for gameServer in updateResult.keys():
				if len(scripts) > updateResult[gameServer]+ignoreResult[gameServer]:
					logger.info("xxxxxxx %s有%d个脚本更新失败 xxxxxxx", gameServer, len(scripts)-(updateResult[gameServer]+ignoreResult[gameServer]))		
		else:
			cleanUp();
			logger.info("============= 脚本更新完成 ====================")
