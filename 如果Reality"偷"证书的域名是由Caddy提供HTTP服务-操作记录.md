# 1. 搭一个正常工作的 Caddy 提供HTTPS服务
这里以域名 `rntnt.whatcanisay.ggff.net` 为例  
DNS域名解析设置 略  
Caddyfile
```
rntnt.whatcanisay.ggff.net
{
    tls Y3JhenlwZWFjZQ@gmail.com
    encode gzip

	reverse_proxy https://zelikk.blogspot.com {
		header_up Host {upstream_hostport}
	}

}
```

# 2. 搭建一个正常工作的 Reality服务端, 特别地, "偷"证书的域名是由第1步中Caddy提供HTTP服务
这里以域名 `rntnt.whatcanisay.ggff.net` 为例
```
curl -LO https://github.com/crazypeace/xray-vless-reality/raw/main/install.sh || wget -O ${_##*/} $_ && bash install.sh 4 8443 rntnt.whatcanisay.ggff.net
```

# 3. 在Docker中搭一个 Reality服务端, 使用宿主机的 Reality服务端 同样的内核和配置
```
docker run -d \
  --name reality-server \
  --network bridge \
  -v /usr/local/bin/xray:/usr/local/bin/xray:ro \
  -v /usr/local/etc/xray/config.json:/usr/local/etc/xray/config.json:ro \
  ghcr.io/xtls/xray-core:latest 
```

查询 Docker 的IP地址
```
docker ps -q | xargs docker inspect -f '{{.Name}} -> {{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

# 4. 运行 Reality客户端 对接Docker中的服务端
reality-client.json
```
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 1080,
      "listen": "127.0.0.1",
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": true
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls"
        ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "172.17.0.2",	//***按实际情况修改
            "port": 8443,
            "users": [
              {
                "id": "2046c4dc-aed9-3e35-b047-e858a827181c",	//***按实际情况修改
                "flow": "xtls-rprx-vision",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "serverName": "rntnt.whatcanisay.ggff.net",
          "publicKey": "-WQvFYoH9cv3DDdEY5lPU1kj1DW7w9HgtApO7edwEmw",	//***按实际情况修改
          "shortId": "71790f91e17fa641",	//***按实际情况修改
          "spiderX": "/"
        }
      },
      "tcpSettings": {
        "header": {
          "type": "none"
        }
      }
    }
  ]
}
```

运行 Reality客户端
```
/usr/local/bin/xray run -config /root/VLESS-cracker/reality-client.json
```

# 5. 启动POC程序
```
./vless-cracker-v1 \
	-i docker0 \
	-f "tcp port 8443 and host 172.17.0.1 and host 172.17.0.2" \
	-P characteristic.txt  \
	-l info
```

# 6. 发起Reality数据包
```
curl -x socks://127.0.0.1:1080 google.com
```

# 7. 分析日志
将打印的日志, 复制粘贴到 `vless-analyzer.html`

# 8. 探针文件说明
`characteristic.txt`  
是原作者的7个探针

`probes-1-10.txt`  
`probes-11-20.txt`  
`probes-21-30.txt`  
是 [issue 29](https://github.com/Anonymous376c1d0cf28/VLESS-cracker/issues/29) 的30个探针.  
其中, 第5, 18, 30号探针情况特殊, 需要单独测试. 在 `probes-*-*.txt` 文件中, 对应的位置填充了数据占位.  
也就是说, 当你使用 `probes-1-10.txt` `probes-11-20.txt` `probes-21-30.txt` 进行测试时, 并没有正确地实施 第5, 18, 30号探针

`probe-5-16385.txt`  
用原poc程序, 会因为探针体积大报错"probe file 'probe-5-16385.txt' line 1 is too long"   
需要使用 `probe-5-poc` 程序. 在程序中调整了 `#define MAX_PROBE_BYTES 32768`   
```
./probe-5-poc \
	-i docker0 \
	-f "tcp port 8443 and host 172.17.0.1 and host 172.17.0.2" \
	-P probe-5-16385.txt   \
	-l info
```

编译 `probe-5-poc` 的方法  
```
gcc -O2 -o probe-5-poc probe-5-mimo.c -lpcap -lpthread
```


`probe-18.txt`  
需要使用 `probe-18-poc` 程序. 在程序中调整了逻辑顺序, 将探针添加到 "client-hello" 的前面, 再合并一起发送.  
```
./probe-18-poc \
	-i docker0 \
	-f "tcp port 8443 and host 172.17.0.1 and host 172.17.0.2" \
	-P probe-18.txt  \
	-l info
```

编译 `probe-18-poc` 的方法  
```
gcc -O2 -o probe-18-poc probe-18-mimo.c -lpcap -lpthread
```

`probe-30.txt`  
单独测试即可  
```
./vless-cracker-v1 \
	-i docker0 \
	-f "tcp port 8443 and host 172.17.0.1 and host 172.17.0.2" \
	-P probe-30.txt     \
	-l info
```

