---
layout:		post
title:		"半自动化SQL注入检测"
subtitle:	"openresty"
date:		2016-03-28
author:		"dogewatch"
header-img:	"img/404-bg.jpg"
tags:
    - web
---

> "gougougou"

## 前言

前几天在网上看到某个大牛<a href="http://zhanghang.org/">博客</a>的一篇文章，是关于如何利用openresty来进行自动化SQL注入挖掘的，心痒之下决定自己动手搭一个平台试试。搭完之后效果感觉不是太好，勉强只能算是一个半自动化的平台，还有待以后改进。

---

## 原理

一、利用nginx做正向代理，通过代理访问待检测的网站。
二、利用redis记录访问的所有请求。
三、利用sqlmap对所有请求进行SQL注入检测。
思路还是比较简单的，不过对于nginx和redis都是第一次接触的我来说，依然免不了一番折腾。

---

## 搭建

一、安装并配置redis.
---
二、安装openresty。然后在其ngnix/conf目录下添加proxy.conf，代码如下：

```
server {
	listen 9999;
	resolver 8.8.8.8 114.114.114.114 valid=3600s; //DNS IP地址

	location ~.*\.(jpg|png|jpeg|gif|js|css|ico)$ {
		 proxy_pass http://$http_host$request_uri; //这部分请求不进行代理
	}

	location / {

		default_type 'text/plain';
		access_by_lua_block {
			
			function conv2str (o)
				local rs = ""
				if type(o) == "string" then
					rs = o
				elseif type(o) == "nil" then
					rs = "nil"
				elseif type(o) == "number" then
					rs = tostring(o)
				elseif type(o) == "boolean" then
					rs = tostring(o)
				elseif type(o) == "table" then
					for k,v in pairs(o) do
						if type(v)=="string" then
							rs = rs..k.."="..v.."&"
						else
							rs = rs..k.."="..conv2str(v).."&"
						end
					end
					if string.find(rs, "&", string.len(rs)-1) ~= nil then
						rs = string.sub(rs, 0, string.len(rs)-1)
					end
				else
					error("cannot serialize a " .. type(o))
				end
				return rs
			end

			function postargs()
				ngx.req.read_body()
				local post=""
				local args, err = ngx.req.get_post_args() or {}
				post=conv2str(args)
				return post
			end

			
			function logall()
				local remote=ngx.var.remote_addr or "-"
				local host=ngx.var.http_host or "-"
				local nowtime=ngx.var.time_local or "-"
				local reqMethod=ngx.req.get_method() or "-"
				local reqUri=string.gsub(ngx.var.request_uri, "?.*", "") or "-"
				local args=""
				local post=""
				local headers=ngx.req.get_headers()
				local cookies=conv2str(headers["Cookie"])
				local useragent=conv2str(headers["User-Agent"])
				local header=conv2str(headers)
				local line=""

				args = conv2str(ngx.req.get_uri_args())
				if reqMethod == "POST" then
					args = postargs()
				end

				line = '{"method":"'..reqMethod..'", "host":"'..host..'", "uri":"'..reqUri..'", "args":"'..args..'",  "cookie":"'..cookies..'", "agent":"'..useragent..'", "remote":"'..remote..'", "nowtime":"'..nowtime..'"}'
				--ngx.say(line)

				local redis = require "resty.redis"
				local red = redis:new()
				red:set_timeout(1000) -- 1 sec
				local ok, err = red:connect("*.*.*.*", 2000) //redis所在IP地址和端口号
				if not ok then
					ngx.say("failed to connect: ", err)
					return
				end
				local res, err = red:auth("*") //redis密码
				red:hmset("proxy."..host, "http://"..host..reqUri, line) //在redis中存储所有请求
				red:close()

			end

			local ret,err = pcall(logall)
			
			if ret then
				return
			else
				ngx.say("failed ", err)
			end
		}
		proxy_pass http://$http_host$request_uri;
		proxy_set_header Host      $host;
		proxy_set_header X-Real-IP $remote_addr;
	}
}
```

这段代码的作用是开启正向代理，并把所有的请求记录在redis中。
然后添加sqlinj.conf，代码如下：

```
server {
	listen 80;
	server_name *.*.*.*; //平台所在IP地址

	location = /sqlinj {
		allow 192.168.49.0/24;
		allow 192.168.48.0/24;
		deny all;

		default_type 'text/plain';
		access_by_lua_block {
			ngx.header.content_type="application/json;charset=utf8"
			local cjson = require "cjson"
			local cjson2 = cjson.new()
			local cjson_safe = require "cjson.safe"
			local redis = require "resty.redis"
			local red = redis:new()
			red:set_timeout(1000) -- 1 sec
			local ok, err = red:connect("*.*.*.*", 6379) //redis所在IP和端口
			if not ok then
				ngx.say("failed to connect: ", err)
				return
			end
			local res, err = red:auth("*") //redis密码
			red:select("1")
			local sqlinj = red:hkeys('sqlinj')
			ngx.say("共发现sql注入漏洞: "..table.getn(sqlinj).." 个")
			
			for key, value in pairs(sqlinj) do
				req = cjson.decode(value)
				ngx.say('\n'..req['method']..'\thttp://'..req['host']..req['uri']..'?'..req['args'])
				ngx.say(value)
				ngx.say(red:hget('sqlinj',value))
			end
			red:close()
		}	
	}
}
```

这段代码是在80端口开一个网页展示SQLMAP扫描的结果。
在nginx.conf最后加上这样一段代码把上述两个配置文件引入进去：

```
include ./proxy.conf;
include ./sqlinj.conf;
```

重启nginx后，在浏览器设置代理，然后访问一个网站（例如www.wooyun.org），可以看到在redis中已经记录下所有请求。

```
127.0.0.1:6379> keys *
1) "proxy.widget.weibo.com"
2) "proxy.www.wooyun.org"
```
---
三、创建param.py，设置一些参数

```
task_new = "task/new"
task_del = "task/<taskid>/delete"
admin_task_list = "admin/<taskid>/list"
admin_task_flush = "admin/<taskid>/flush"
option_task_list = "option/<taskid>/list"
option_task_get = "option/<taskid>/get"
option_task_set = "option/<taskid>/set"
scan_task_start = "scan/<taskid>/start"
scan_task_stop = "scan/<taskid>/stop"
scan_task_kill = "scan/<taskid>/kill"
scan_task_status = "scan/<taskid>/status"
scan_task_data = "scan/<taskid>/data"
scan_task_log = "scan/<taskid>/log/<start>/<end>"
scan_task_log = "scan/<taskid>/log"
download_task = "download/<taskid>/<target>/<filename:path>"
# config
taskid = "<taskid>"
dbOld = 0 //两个db，db0以hash的方式存储所有请求。
dbNew = 1 //db1以set的方式存储带检测的请求。
host = '*.*.*.*' //redis IP地址
port = 6379  //redis端口号
password = '*' //redis密码

joblist = 'job.set'
sqlinj = 'sqlinj'
```
---
四、因为原始的请求是以HASH的方式存储的，不利于随机取数据，因此创建一个preprocess.py脚本将数据以set的方式存储在db1中

```
import json
import redis
import sys
import param

def queryKey(key):
	r = redis.StrictRedis(host=param.host, port=param.port, password=param.password, db=param.dbOld)
	return r.keys(key)

def queryHashFields(key):
	r = redis.StrictRedis(host=param.host, port=param.port, password=param.password, db=param.dbOld)
	return r.hkeys(key)

def queryHashValue(key, filed):
	r = redis.StrictRedis(host=param.host, port=param.port, password=param.password, db=param.dbOld)
	return r.hget(key, filed)

def setAdd(key, value):
	r = redis.StrictRedis(host=param.host, port=param.port, password=param.password, db=param.dbNew)
	r.sadd(key, value)

def main():
	if len(sys.argv) < 2:
		print "缺少参数： python %s hostname" % sys.argv[0]
		exit()
	hostname = sys.argv[1]
	print hostname
	keys = queryKey(hostname)
	for key in keys:
		print key
		fileds = queryHashFields(key)
		total = len(fileds)
		times = 0
		print "共计%d个请求记录" % total
		for filed in fileds:
			times = times + 1
			req = queryHashValue(key, filed)
			setAdd(param.joblist, req)
			if times*10%total == 0:
				print str(times*100.0/total) + "%"
	print "finish..."

if __name__ == '__main__':
	main()
```

hostname就是db0中请求的key name，例如前例中的proxy.www.wooyun.org。

---
五、下载sqlmap，进入目录运行sqlmapapi.py（python sqlmapapi.py -s -H 127.0.0.1）

---
六、创建脚本autoinj.py，该脚本负责从db1中取数据进行扫描

```
import time
import json
import urllib
import urllib2
import redis
import requests
import param

class Autoinj(object):

	def __init__(self, server='', target='', method='', data='', cookie='', referer=''):
		super(Autoinj, self).__init__()
		self.server = server
		if self.server[-1] != '/':
			self.server = self.server + '/'
		if method == "GET":
			self.target = target + '?' + data
		else:
			self.target = target
		self.taskid = ''
		self.engineid = ''
		self.status = ''
		self.method = method
		self.data = data
		self.referer = referer
		self.cookie = cookie
		self.start_time = time.time()
		#print "server: %s \ttarget:%s \tmethod:%s \tdata:%s \tcookie:%s" % (self.server, self.target, self.method, self.data, self.cookie)

	def task_new(self):
		code = urllib.urlopen(self.server + param.task_new).read()
		self.taskid = json.loads(code)['taskid']
		return True

	def task_delete(self):
		url = self.server + param.task_del
		url = url.replace(param.taskid, self.taskid)
		requests.get(url).json()

	def scan_start(self):
		headers = {'Content-Type':'application/json'}
		url = self.server + param.scan_task_start
		url = url.replace(param.taskid, self.taskid)
		data = {'url':self.target}
		t = requests.post(url, data=json.dumps(data), headers=headers).text
		t = json.loads(t)
		self.engineid = t['engineid']
		return True

	def scan_status(self):
		url = self.server + param.scan_task_status
		url = url.replace(param.taskid, self.taskid)
		self.status = requests.get(url).json()['status']

	def scan_data(self):
		url = self.server + param.scan_task_data
		url = url.replace(param.taskid, self.taskid)
		return requests.get(url).json()

	def option_set(self):
		headers = {'Content-Type':'application/json'}
		url = self.server + param.option_task_set
		url = url.replace(param.taskid, self.taskid)
		data = {
				"user-agent":
					'''Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_2)
				 		AppleWebKit/537.36 (KHTML, like Gecko) Chrome/47.0.2526.106'''
				}
		if self.method == "POST":
			data["data"] = self.data
		if len(self.cookie)>1:
			data["cookie"] = self.cookie
		#print data

		t = requests.post(url, data=json.dumps(data), headers=headers).text
		t = json.loads(t)

	def option_get(self):
		url = self.server + param.option_task_get
		url = url.replace(param.taskid, self.taskid)
		return requests.get(url).json()

	def scan_stop(self):
		url = self.server + param.scan_task_stop
		url = url.replace(param.taskid, self.taskid)
		return requests.get(url).json()

	def scan_kill(self):
		url = self.server + param.scan_task_kill
		url = url.replace(param.taskid, self.taskid)
		return requests.get(url).json()

	def run(self):
		# 开始任务
		if not self.task_new():
			print "Error: task created failed."
			return False
		# 设置扫描参数
		self.option_set()
		# 启动扫描任务
		if not self.scan_start():
			print "Error: scan start failed."
			return False
		# 等待扫描任务
		while True:
			self.scan_status()
			if self.status == 'running':
				time.sleep(3)
			elif self.status== 'terminated':
				break
			else:
				print "unkown status"
				break
			if time.time() - self.start_time > 30000:
				error = True
				self.scan_stop()
				self.scan_kill()
				break

		# 取结果
		res = self.scan_data()
		# 删任务
		self.task_delete()
		print "耗时:" + str(time.time() - self.start_time)
		return res
```

参考乌云上的一篇<a href="http://drops.wooyun.org/tips/6653?utm_source=tuicool">文章</a>

---
七、创建console.py调用sqlinj.py进行扫描，并将结果保存至redis中

```
import sys
import json
import time
import redis
import autoinj
import param


def insertHash(key, filed, value):
	r = redis.StrictRedis(host=param.host, port=param.port, password=param.password, db=param.dbNew)
	r.hset(key, filed, value)

def randomOne(setKey):
	r = redis.StrictRedis(host=param.host, port=param.port, password=param.password, db=param.dbNew)
	return r.spop(setKey)

def main():
	if len(sys.argv) < 2:
		print "缺少参数： python %s http://127.0.0.1:8775" % sys.argv[0]
		exit()
	server = sys.argv[1]
	while True:
		try:
			req = randomOne(param.joblist)
			reqJson = json.loads(req)
			method = reqJson["method"]
			target = reqJson["host"] + reqJson["uri"]
			data = reqJson["args"]
			cookie = reqJson["cookie"]

			inj = autoinj.Autoinj(server, target, method, data, cookie)
			rs = inj.run()
			print rs

			if len(rs['data'])>0:
				print "### INJ ###"
				print rs['data'][0]
				insertHash(param.sqlinj, req, rs['data'][0])
		except Exception, e:
			print "本次扫描发生了一点意外"
			print e
			time.sleep(1)


if __name__ == '__main__':
	main()
```

---

## 使用

一、浏览器设置代理，访问目标网站。
二、运行preprocess.py。
三、启动sqlmapapi.py
四、运行console.py
五、访问80端口查看结果。

---

## 后记

折腾了一天搭好了这个平台，但是用着不是太爽，希望达到的目标是浏览器一边访问目标网站后台一边在进行扫描，得改改相关代码。