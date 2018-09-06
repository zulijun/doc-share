1. 获取token
```bash
TOKEN=$(openstack token issue | grep '^| id' | awk '{print$4}')
```
2. 打印出token
```bash
echo $TOKEN
gAAAAABbS_-pN_9X4LXRfjDYpktLgxuBn5ybfqZYZh3EuvbtHaAJTMNzpBUHg9C6K6HB9f-ZLNvWwGkXu0JL7f3hb0dEtPjqOSaEomOHZPf4ajFOzBmbDMhV8FxyJxg-jvrOdMKJg31_LPlemGI45CVTZJaLzl9NnhWDpQsF976twEdgYjIwMW0
```
3. 调用zun api
```
curl -g -i -X POST http://172.16.25.47/container/v1/containers -H "OpenStack-API-Version: container 1.12" -H "X-Auth-Token: gAAAAABbS_-pN_9X4LXRfjDYpktLgxuBn5ybfqZYZh3EuvbtHaAJTMNzpBUHg9C6K6HB9f-ZLNvWwGkXu0JL7f3hb0dEtPjqOSaEomOHZPf4ajFOzBmbDMhV8FxyJxg-jvrOdMKJg31_LPlemGI45CVTZJaLzl9NnhWDpQsF976twEdgYjIwMW0" -H "Content-Type: application/json" -H "Accept: application/json" -H "User-Agent: None" -d '{"auto_remove": false, "name": "test3", "image": "centos-docker", "labels": {}, "environment": {}, "image_driver": "glance", "mounts": [{"source": "volume-test", "destination": "/volume-test", "size": ""}], "command": "\"ping\" \"127.0.0.1\"", "nets": [{"network": "1f0aeb31-c27b-489f-9c55-3562c366660f"}], "hints": {}}'
```
<br></br>
*注：获取zun api的方法：*

执行zun命令，并加上参数--debug，即可在debug信息中看到URL的api指令