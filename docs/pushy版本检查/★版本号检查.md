(脚本创建在个人家目录下，防止权限问题)
pushy地址
```
https://pushy-admin.reactnative.cn/
```
版本号文件路径
```
/home/lixu/apkVersion/iosVersion
/home/lixu/apkVersion/androidVersion
```
脚本crontab
```
*/10 * * * * /home/lixu/apkVersion/apkVersioncheck.sh
```
时间戳文件路径（检查脚本执行情况）
```
/home/lixu/apkVersion/dateStamp
```
### 方法
获取token
```shell
curl 'https://update.react-native.cn/api/user/login' \
-H 'user-agent:Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36' \
-H 'content-type: application/json' \
-H 'Referer: https://pushy-admin.reactnative.cn/' \
--data-raw '{"email":"gooduckx@126.com","pwd":"9e853bd8bdcc0959d2bdfba52283b116"}' | jq -r '.token'
```
ios最新版本号
```shell
curl -H "x-accesstoken: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoidXNlciIsInVpZCI6Mjc3NzIsImV4cCI6MTc2OTA2NDczODYzNCwiaWF0IjoxNzY2NDcyNzM4fQ._xknCy8f_XduGNOVj15R41BKGMSKudl4A40koOFQcu8" \
	https://update.reactnative.cn/api/app/30933/package/list\?limit\=1000 | jq -r '.data[0].name'
```

安卓最新版本号
```shell
curl -H "x-accesstoken: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoidXNlciIsInVpZCI6Mjc3NzIsImV4cCI6MTc2ODk2MTk5MjU1MiwiaWF0IjoxNzY2MzY5OTkyfQ.cAgCt9vMKUD1zruhWism2iuiRXYknHWRLe4FjJ4k98Q" \
	https://update.reactnative.cn/api/app/30932/package/list\?limit\=1000 | jq -r '.data[0].name'
```




## 完整脚本
```shell
#!/bin/bash

# 配置项
IOS_API_URL="https://update.reactnative.cn/api/app/30933/package/list?limit=1000"      # iOS版本接口地址
ANDROID_API_URL="https://update.reactnative.cn/api/app/30932/package/list?limit=1000"  # Android版本接口地址
FEISHU_WEBHOOK="https://open.feishu.cn/open-apis/bot/v2/hook/c8dfa4ea-3aa4-4669-816c-842afd1fb8ff"         # 飞书webhook地址

# 文件路径
IOS_VERSION_FILE="/home/lixu/apkVersion/iosVersion"
ANDROID_VERSION_FILE="/home/lixu/apkVersion/androidVersion"

# 版本号比较函数（使用 dpkg --compare-versions 支持 x.y.z 格式）
compare_versions() {
    local v1=$1
    local v2=$2

    if dpkg --compare-versions "$v1" lt "$v2" 2>/dev/null; then
        echo "-1"  # v1 < v2
    elif dpkg --compare-versions "$v1" gt "$v2" 2>/dev/null; then
        echo "1"   # v1 > v2
    else
        echo "0"   # v1 == v2
    fi
}

# 发送飞书消息函数（文本消息）
send_feishu_message() {
    local title=$1
    local content=$2

    local json_data=$(cat <<EOF
{
  "msg_type": "text",
  "content": {
    "text": "$title\n\n$content"
  }
}
EOF
)

    curl -s -X POST "$FEISHU_WEBHOOK" \
        -H "Content-Type: application/json" \
        -d "$json_data" > /dev/null
}
#获取token
get_token() {
    curl 'https://update.react-native.cn/api/user/login' \
        -H 'user-agent:Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36' \
        -H 'content-type: application/json' \
        -H 'Referer: https://pushy-admin.reactnative.cn/' \
        --data-raw '{"email":"gooduckx@126.com","pwd":"9e853bd8bdcc0959d2bdfba52283b116"}' | jq -r '.token'
}

TOKEN=$(get_token)

# 处理版本检查的函数
check_version() {
    local platform=$1
    local api_url=$2
    local version_file=$3

    # 获取远程版本号（从接口返回的 JSON 中取 .data[0].name 作为版本号）
    RESPONSE=$(
        curl -s -H "x-accesstoken: $TOKEN" \
            "$api_url" 2>/dev/null
    )

    # 检查响应是否有效
    if [ -z "$RESPONSE" ]; then
        # 网络错误，静默退出，不发信息
        exit 1
    fi

    REMOTE_VERSION=$(echo "$RESPONSE" | jq -r '.data[0].name' 2>/dev/null)
    
    # 检查版本号是否为空或null，错误发送信息到飞书
    if [ -z "$REMOTE_VERSION" ] || [ "$REMOTE_VERSION" == "null" ] || [ "$REMOTE_VERSION" == "" ]; then
		send_feishu_message "pushy版本号" "${platform}获取版本号为null或空值"
        exit 1
    fi

    # 读取本地版本号
    if [ ! -f "$version_file" ]; then
        echo "$REMOTE_VERSION" > "$version_file" 2>/dev/null
        return 0
    fi

    LOCAL_VERSION=$(cat "$version_file" 2>/dev/null | tr -d '[:space:]')
    if [ -z "$LOCAL_VERSION" ]; then
        echo "$REMOTE_VERSION" > "$version_file" 2>/dev/null
        return 0
    fi

    # 比较版本号
    COMPARE_RESULT=$(compare_versions "$LOCAL_VERSION" "$REMOTE_VERSION" 2>/dev/null)

    # 根据比较结果执行相应操作
    if [ "$COMPARE_RESULT" == "0" ]; then
        # 本地版本 = 远程版本，结束本平台检查,加时间戳
        echo $(date) > /home/lixu/apkVersion/dateStamp
        return 0
    elif [ "$COMPARE_RESULT" == "1" ]; then
        # 本地版本 > 远程版本（版本异常）
        MESSAGE="**${platform}版本异常**"
        send_feishu_message "版本异常" "$MESSAGE"
        return 1
    else
        # 本地版本 < 远程版本（版本更新成功）
        echo "$REMOTE_VERSION" > "$version_file" 2>/dev/null
        # 推送到飞书上的内容可按需修改，这里只发标题
        send_feishu_message "${platform}版本更新，当前版本为${REMOTE_VERSION}"
        return 0
    fi
}

# 检查 iOS 版本
check_version "iOS" "$IOS_API_URL" "$IOS_VERSION_FILE"

# 检查 Android 版本
check_version "Android" "$ANDROID_API_URL" "$ANDROID_VERSION_FILE"
```



  