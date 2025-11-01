# V2Ray服务器配置

1. 首先使用官方安装脚本

~~~shell
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
~~~

![1761658209112](C:\Users\Lenovo\AppData\Roaming\Typora\typora-user-images\1761658209112.png)



1. 订阅转配置

   1. 创建脚本文件

      ~~~shell
      #!/bin/bash
      
      # 参数检查
      if [ $# -ne 2 ]; then
        echo "用法: $0 订阅链接 输出配置文件"
        echo "例如: $0 https://example.com/sub 配置文件.json"
        exit 1
      fi
      
      SUB_URL="$1"
      OUTPUT_FILE="$2"
      
      echo "=== V2Ray订阅转换脚本 ==="
      echo "订阅链接: ${SUB_URL:0:30}..."
      echo "输出文件: $OUTPUT_FILE"
      echo
      
      # 下载并解码订阅内容
      echo "[1/5] 下载订阅内容..."
      curl -sL "$SUB_URL" > sub_encoded.txt
      
      # 检查是否下载成功
      if grep -q "<html>" sub_encoded.txt; then
        echo "错误: 订阅下载失败，服务器返回HTML内容:"
        cat sub_encoded.txt | head -5
        exit 1
      fi
      
      echo "[2/5] 解码订阅内容..."
      base64 -d sub_encoded.txt > sub_decoded.txt 2>/dev/null
      if [ $? -ne 0 ]; then
        echo "错误: base64解码失败，可能不是标准的base64编码"
        exit 1
      fi
      
      # 检查解码后的内容
      echo "[3/5] 检查节点类型..."
      NODE_COUNT=$(wc -l < sub_decoded.txt)
      echo "找到 $NODE_COUNT 个节点"
      
      # 优先选择vmess节点
      if grep -q "^vmess://" sub_decoded.txt; then
        echo "找到vmess节点，优先使用"
        VMESS_LINK=$(grep "^vmess://" sub_decoded.txt | head -1)
        PROTOCOL="vmess"
      elif grep -q "^trojan://" sub_decoded.txt; then
        echo "找到trojan节点，将使用trojan协议"
        TROJAN_LINK=$(grep "^trojan://" sub_decoded.txt | head -1)
        PROTOCOL="trojan"
      elif grep -q "^ss://" sub_decoded.txt; then
        echo "找到ss节点，将使用shadowsocks协议"
        SS_LINK=$(grep "^ss://" sub_decoded.txt | head -1)
        PROTOCOL="ss"
      else
        echo "错误: 未找到支持的节点类型(vmess/trojan/ss)"
        exit 1
      fi
      
      # 处理不同类型的节点
      echo "[4/5] 解析节点信息..."
      
      if [ "$PROTOCOL" = "vmess" ]; then
        # 解析vmess链接
        VMESS_DATA=$(echo "${VMESS_LINK#vmess://}" | base64 -d 2>/dev/null)
        if [ $? -ne 0 ]; then
          echo "错误: vmess链接解码失败"
          exit 1
        fi
        
        # 提取vmess配置
        SERVER_ADDRESS=$(echo "$VMESS_DATA" | grep -o '"add":"[^"]*"' | cut -d'"' -f4)
        SERVER_PORT=$(echo "$VMESS_DATA" | grep -o '"port":[0-9]*' | cut -d':' -f2)
        USER_ID=$(echo "$VMESS_DATA" | grep -o '"id":"[^"]*"' | cut -d'"' -f4)
        ALTER_ID=$(echo "$VMESS_DATA" | grep -o '"aid":[0-9]*' | cut -d':' -f2 || echo "0")
        NETWORK=$(echo "$VMESS_DATA" | grep -o '"net":"[^"]*"' | cut -d'"' -f4 || echo "tcp")
        PATH=$(echo "$VMESS_DATA" | grep -o '"path":"[^"]*"' | cut -d'"' -f4 || echo "")
        TLS=$(echo "$VMESS_DATA" | grep -o '"tls":"[^"]*"' | cut -d'"' -f4 || echo "none")
        HOST=$(echo "$VMESS_DATA" | grep -o '"host":"[^"]*"' | cut -d'"' -f4 || echo "")
        
        echo "节点信息:"
        echo "  协议: vmess"
        echo "  地址: $SERVER_ADDRESS"
        echo "  端口: $SERVER_PORT"
        echo "  用户ID: ${USER_ID:0:8}..."
        echo "  alterID: $ALTER_ID"
        echo "  传输协议: $NETWORK"
        echo "  TLS: $TLS"
        
        # 生成vmess配置
        echo "[5/5] 生成V2Ray配置文件..."
        cat > "$OUTPUT_FILE" << EOF
      {
        "log": {
          "loglevel": "warning"
        },
        "inbounds": [
          {
            "port": 10808,
            "protocol": "socks",
            "sniffing": {
              "enabled": true,
              "destOverride": ["http", "tls"]
            },
            "settings": {
              "auth": "noauth"
            }
          },
          {
            "port": 10809,
            "protocol": "http",
            "sniffing": {
              "enabled": true,
              "destOverride": ["http", "tls"]
            }
          }
        ],
        "outbounds": [
          {
            "protocol": "vmess",
            "settings": {
              "vnext": [
                {
                  "address": "$SERVER_ADDRESS",
                  "port": $SERVER_PORT,
                  "users": [
                    {
                      "id": "$USER_ID",
                      "alterId": $ALTER_ID,
                      "security": "auto"
                    }
                  ]
                }
              ]
            },
            "streamSettings": {
              "network": "$NETWORK",
      EOF
      
        # 根据TLS设置添加相应配置
        if [ "$TLS" != "none" ] && [ -n "$TLS" ]; then
          cat >> "$OUTPUT_FILE" << EOF
              "security": "$TLS",
              "tlsSettings": {
                "serverName": "$HOST"
              },
      EOF
        else
          cat >> "$OUTPUT_FILE" << EOF
              "security": "none",
      EOF
        fi
      
        # 根据网络类型添加相应配置
        if [ "$NETWORK" = "ws" ]; then
          cat >> "$OUTPUT_FILE" << EOF
              "wsSettings": {
                "path": "$PATH",
                "headers": {
                  "Host": "$HOST"
                }
              }
      EOF
        elif [ "$NETWORK" = "tcp" ]; then
          cat >> "$OUTPUT_FILE" << EOF
              "tcpSettings": {}
      EOF
        fi
      
        cat >> "$OUTPUT_FILE" << EOF
            }
          },
          {
            "protocol": "freedom",
            "tag": "direct"
          }
        ],
        "routing": {
          "domainStrategy": "IPOnDemand",
          "rules": [
            {
              "type": "field",
              "ip": ["geoip:private"],
              "outboundTag": "direct"
            }
          ]
        }
      }
      EOF
      
      elif [ "$PROTOCOL" = "trojan" ]; then
        # 解析trojan链接
        # trojan://password@server:port?allowInsecure=1#remarks
        TROJAN_PASSWORD=$(echo "$TROJAN_LINK" | sed 's/trojan:\/\///' | cut -d '@' -f1)
        SERVER_INFO=$(echo "$TROJAN_LINK" | sed 's/trojan:\/\///' | cut -d '@' -f2)
        SERVER_ADDRESS=$(echo "$SERVER_INFO" | cut -d ':' -f1)
        SERVER_PORT=$(echo "$SERVER_INFO" | cut -d ':' -f2 | cut -d '?' -f1)
        ALLOW_INSECURE=$(echo "$TROJAN_LINK" | grep -o "allowInsecure=1" || echo "")
        
        echo "节点信息:"
        echo "  协议: trojan"
        echo "  地址: $SERVER_ADDRESS"
        echo "  端口: $SERVER_PORT"
        
        # 生成trojan配置
        echo "[5/5] 生成V2Ray配置文件..."
        cat > "$OUTPUT_FILE" << EOF
      {
        "log": {
          "loglevel": "warning"
        },
        "inbounds": [
          {
            "port": 10808,
            "protocol": "socks",
            "sniffing": {
              "enabled": true,
              "destOverride": ["http", "tls"]
            },
            "settings": {
              "auth": "noauth"
            }
          },
          {
            "port": 10809,
            "protocol": "http",
            "sniffing": {
              "enabled": true,
              "destOverride": ["http", "tls"]
            }
          }
        ],
        "outbounds": [
          {
            "protocol": "trojan",
            "settings": {
              "servers": [
                {
                  "address": "$SERVER_ADDRESS",
                  "port": $SERVER_PORT,
                  "password": "$TROJAN_PASSWORD"
                }
              ]
            },
            "streamSettings": {
              "network": "tcp",
              "security": "tls",
              "tlsSettings": {
                "serverName": "$SERVER_ADDRESS",
                "allowInsecure": $([ -n "$ALLOW_INSECURE" ] && echo "true" || echo "false")
              }
            }
          },
          {
            "protocol": "freedom",
            "tag": "direct"
          }
        ],
        "routing": {
          "domainStrategy": "IPOnDemand",
          "rules": [
            {
              "type": "field",
              "ip": ["geoip:private"],
              "outboundTag": "direct"
            }
          ]
        }
      }
      EOF
      
      elif [ "$PROTOCOL" = "ss" ]; then
        echo "SS协议支持正在开发中..."
        echo "请手动配置SS协议"
        exit 1
      fi
      
      echo "配置已保存到 $OUTPUT_FILE"
      echo "使用以下命令启动V2Ray:"
      echo "v2ray -c $OUTPUT_FILE"
      echo
      echo "使用以下命令设置代理环境变量:"
      echo "export http_proxy=http://127.0.0.1:10809"
      echo "export https_proxy=http://127.0.0.1:10809"
      echo
      echo "或使用SOCKS5代理:"
      echo "export http_proxy=socks5://127.0.0.1:10808"
      echo "export https_proxy=socks5://127.0.0.1:10808"
      
      # 清理临时文件
      rm -f sub_encoded.txt sub_decoded.txt
      ~~~

​                添加执行权限：`chmod +x v2ray_sub.sh`

​				使用脚本文件生成配置：`./v2ray_sub.sh "https://your-subscription-link" config.json`



3. 手动编辑配置文件：`sudo vim /usr/local/etc/v2ray/config.json`

   - VMess协议配置实例

     ~~~json
     {
       "log": {
         "loglevel": "warning"
       },
       "inbounds": [
         {
           "port": 10808,
           "protocol": "socks",
           "sniffing": {
             "enabled": true,
             "destOverride": ["http", "tls"]
           },
           "settings": {
             "auth": "noauth"
           }
         },
         {
           "port": 10809,
           "protocol": "http",
           "sniffing": {
             "enabled": true,
             "destOverride": ["http", "tls"]
           }
         }
       ],
       "outbounds": [
         {
           "protocol": "vmess",
           "settings": {
             "vnext": [
               {
                 "address": "服务器地址",
                 "port": 443,
                 "users": [
                   {
                     "id": "用户UUID",
                     "alterId": 0,
                     "security": "auto"
                   }
                 ]
               }
             ]
           },
           "streamSettings": {
             "network": "ws",
             "security": "tls",
             "tlsSettings": {
               "serverName": "服务器域名"
             },
             "wsSettings": {
               "path": "/path"
             }
           }
         }
       ]
     }
     ~~~

     

   - Trojan协议配置示例

     ~~~json
     {
       "log": {
         "loglevel": "warning"
       },
       "inbounds": [
         {
           "port": 10808,
           "protocol": "socks",
           "sniffing": {
             "enabled": true,
             "destOverride": ["http", "tls"]
           },
           "settings": {
             "auth": "noauth"
           }
         },
         {
           "port": 10809,
           "protocol": "http",
           "sniffing": {
             "enabled": true,
             "destOverride": ["http", "tls"]
           }
         }
       ],
       "outbounds": [
         {
           "protocol": "trojan",
           "settings": {
             "servers": [
               {
                 "address": "服务器地址",
                 "port": 443,
                 "password": "密码"
               }
             ]
           },
           "streamSettings": {
             "network": "tcp",
             "security": "tls",
             "tlsSettings": {
               "serverName": "服务器域名",
               "allowInsecure": false
             }
           }
         }
       ]
     }
     ~~~

4. 启用V2Ray服务

   - 1,。使用systemd管理（推荐）

     ~~~shell
     # 启动V2Ray
     sudo systemctl start v2ray
     
     # 设置开机自启
     sudo systemctl enable v2ray
     
     # 查看状态
     sudo systemctl status v2ray
     
     # 重启服务
     sudo systemctl restart v2ray
     
     # 停止服务
     sudo systemctl stop v2ray
     
     # 查看日志
     sudo journalctl -u v2ray
     ~~~

   - 2.直接运行V2Ray

     ~~~shell
     # 前台运行
     v2ray -c /usr/local/etc/v2ray/config.json
     
     # 后台运行
     nohup v2ray -c /usr/local/etc/v2ray/config.json > /dev/null 2>&1 &
     ~~~



5. 设置代理环境变量

   - 1.临时设置代理

     ~~~shell
     # HTTP代理
     export http_proxy=http://127.0.0.1:10809
     export https_proxy=http://127.0.0.1:10809
     
     # 或SOCKS5代理
     export http_proxy=socks5://127.0.0.1:10808
     export https_proxy=socks5://127.0.0.1:10808
     ~~~

   - 2.永久设置代理

     ~~~shell
     echo 'export http_proxy=http://127.0.0.1:10809' >> ~/.bashrc
     echo 'export https_proxy=http://127.0.0.1:10809' >> ~/.bashrc
     source ~/.bashrc
     ~~~

     