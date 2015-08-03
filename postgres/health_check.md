# 健康监测

在我们开发初期经常遇见这样一个问题，就是pg挂掉了，但是nginx却全然不知，依然会向pg请求查询，由于建立的是长连接，查询失败后还会继续尝试，这样造成pg马上把cpu吃满了
所以我们需要检查pg状态，pg挂掉了，就不用查了

我们的做法是每过一段时间请求一次pg查询，获取pg状态并保存到缓存中，每次pg查询时根据这个状态判断pg是否ok!看下实现代码：

	function get_status()
		local status = ngx_share:get("pg_status")
		if nil == status then
			status = true
		end

		if status  then
			local suc = ngx_share:add("pg_status", false, 60) -- only one worker working
			if suc then
				local ip = ngx.var.server_addr
				local port = ngx.var.server_port

	            check_background(false, ip, port)
			end
		end

		if status then	-- unknown
			return true
		else
			return false
		end
	end

get_status返回的就是pg状态，他做的就是获取缓存中"pg_status"的值，首次检查时由于"pg_status"是nil，所以会调用check_background，它做的就是定时检查pg的真实情况，代码如下

```Lua
	local check_background
	check_background = function (ip_add, port )

		local start_time = ngx.now()

		url_open(ip_add, port)

		local delay = 5
		local left_time = start_time + delay - ngx.now()
		if left_time < 0 then
			left_time = 0
		end

		ngx.timer.at(left_time, check_background, ip_add, port)
	end
```

可以看出这个函数是个死循环，每隔left_time时间后触发check_background，并且会触发url_open，再看下url_open的逻辑：

```Lua
	local function url_open( ip, port )
		local req_html = get_test_html(ip, port, "api/check_pg.html")

		local sock = ngx.socket.tcp()
	    local ok, err = sock:connect(ip, port)
	    if not ok then
	        ngx.log(ngx.NOTICE, string.format("failed to connect[%s:%s]: ", ip, port, err))
	        return
	    end

	    local bytes, err = sock:send(req_html)
	    if nil == bytes then
	    	sock:close()
	        ngx.log(ngx.NOTICE, "failed to send: ", err)
	        return
	    end

	    ngx.log(ngx.DEBUG, "successfully connected!")

	    sock:receive(1)
	    sock:close()

	    return
	end
```

这个函数做的就是发起一个http请求api/check_pg.html，而这个请求才是真正会处理“pg_status”的地方:

```Lua
	local config = require "conf.config"
	local json  = require "resty.dkjson"

	local ngx_share = ngx.shared.cache_ngx

	local function query_db(sql)
	  local res = ngx.location.capture('/postgres',
	      { args = {sql = sql } }
	  )

	  local status = res.status
	  local body = json.decode(res.body)

	  if status == 200 then
	     status = true
	  else
	     status = false
	  end
	  return status, body
	end

	local sql = [[select id from config limit 1]]
	local succ, res = query_db(sql)
	ngx_share:set(config.POSTGRESQL .. "pg_status", succ, 60)

	ngx.log(ngx.NOTICE, "pg checked status:", succ)
	ngx.say(succ)
```

至此实现的逻辑全部完成，需求就是每隔一定时间后检查pg状态！

后来我们的老大感觉不爽，说这种实现代码可读性不高，然后他说我们做个单实例吧，主要方案就是用lock这个类，与上个做法的区别就是pg状态的检查
