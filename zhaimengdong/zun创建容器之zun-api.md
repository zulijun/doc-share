调试的话这样去执行命令（通过该方法调试会报错，暂时不知道解决方法）
```shell
/usr/bin/uwsgi --procname-prefix zun-api --ini /etc/zun/zun-api-uwsgi.ini
```
与容器创建相关的入口函数在`/opt/stack/zun/zun/api/controllers/v1/containers`中的类`ContainersController`的`post()`方法，该方法接收一个字典`container_dict`，这个字典包含了容器的各种信息，一个例子（执行的命令是`zun create --name test3 --mount source=volume-test,destination=/volume-test --net network=1f0aeb31-c27b-489f-9c55-3562c366660f --image-driver glance centos-docker ping 127.0.0.1`）
```python
{
    u'auto_remove': False, 
    u'name': u'test3', 
    u'image': u'centos-docker', 
    u'labels': {}, 
    u'environment': {}, 
    u'image_driver': u'glance', 
    u'mounts': [{
        u'source': u'volume-test', 
        u'destination':u'/volume-test', 
        u'size': u''
    }], 
    u'command': u'"ping" "127.0.0.1"', 
    u'nets': [{u'network': u'1f0aeb31-c27b-489f-9c55-3562c366660f'}], u'hints': {}
}
```
执行到
```python
mounts = container_dict.pop('mounts', [])
        if mounts:
            req_version = pecan.request.version
            min_version = versions.Version('', '', '', '1.11')
            if req_version < min_version:
                raise exception.InvalidParamInVersion(param='mounts',
                                                      req_version=req_version,
                                                      min_version=min_version)

        requested_volumes = self._build_requested_volumes(context, mounts)
```
此处对`container_dict`中的`mounts`进行处理，得到`requested_volumes`，一个例子
```
[VolumeMapping(auto_remove=False,connection_info=<?>,container=<?>,container_path='/volume-test',container_uuid=<?>,created_at=<?>,id=<?>,project_id='7e81dae069094c36aa3307d4b42aedc3',updated_at=<?>,user_id='0a3fc0ad5aa842e39e67efcd9346336f',uuid=<?>,volume_id=920d4e73-4eca-446e-9790-d3b9261ce6b1,volume_provider='cinder')]
```
这个函数是针对挂载volume进行的参数处理，如果是挂载目录，在此处应重写函数

接下来构建`kwargs`，在
```
kwargs['requested_volumes'] = requested_volumes
```
将requested_volumes加入kwargs中，kwargs的一个示例
```python
{
	'run': False,
	'requested_networks': [{
		'router:external': False,
		'preserve_on_delete': False,
		'network': u '1f0aeb31-c27b-489f-9c55-3562c366660f',
		'v6-fixed-ip': '',
		'pci_request_id': None,
		'shared': False,
		'v4-fixed-ip': '',
		'port': ''
	}],
	'requested_volumes': [VolumeMapping(auto_remove = False, connection_info = << ? > , container = << ? > , container_path = '/volume-test', container_uuid = << ? > , created_at = << ? > , id = << ? > , project_id = '7e81dae069094c36aa3307d4b42aedc3', updated_at = << ? > , user_id = '0a3fc0ad5aa842e39e67efcd9346336f', uuid = << ? > , volume_id = 920 d4e73 - 4e ca - 446e-9790 - d3b9261ce6b1, volume_provider = 'cinder')],
	'extra_spec': {
		'pci_requests': ContainerPCIRequests(container_uuid = << ? > , created_at = << ? > , requests = [], updated_at = << ? > ),
		'hints': {}
	}
}
```