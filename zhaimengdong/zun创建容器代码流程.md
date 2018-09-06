## zun创建容器代码流程
>环境描述 centos7,devstack/queens

首先关闭要调试的zun-compute服务
```bash
systemctl stop devstack@zun-compute
```
在`/opt/stack/zun/zun/compute/manage.py`中的函数`container_create()`中加入断点，如下
```python
def container_create(self, context, limits, requested_networks,
                     requested_volumes, container, run, pci_requests=None):
    import pdb;pdb.set_trace()#打上断点
    @utils.synchronized(container.uuid)
    def do_container_create():
        self._wait_for_volumes_available(context, requested_volumes,
                                         container)
        if not self._attach_volumes(context, container, requested_volumes):
            return
        created_container = self._do_container_create(
            context, container, requested_networks, requested_volumes,
            pci_requests, limits)
        if run and created_container:
            self._do_container_start(context, created_container)

    utils.spawn_n(do_container_create)
```
然后通过命令行在终端下直接运行服务
```bash
/bin/zun-compute
```
此时执行创建容器的命令，zun-compute进程就会弹出pdb shell，通过pdb命令即可对代码一步步执行

附：pdb命令
* b 设置断点
* c 继续执行程序, 或是跳到下个断点
* l 查看当前行的代码段
* s 进入函数
* r 执行代码直到从当前函数返回
* q 中止并退出
* n 执行下一行
* p 打印变量的值，例如p a表示打印a的值

因为暂时没有找到zun-api对创建容器的入口方法，我们直接将断点放在zun-compute中的入口方法
```
def container_create(self, context, limits, requested_networks,
                     requested_volumes, container, run, pci_requests=None):
    import pdb;pdb.set_trace()
    @utils.synchronized(container.uuid)
    def do_container_create():
        self._wait_for_volumes_available(context, requested_volumes,
                                         container)
        if not self._attach_volumes(context, container, requested_volumes):
            return
        created_container = self._do_container_create(
            context, container, requested_networks, requested_volumes,
            pci_requests, limits)
        if run and created_container:
            self._do_container_start(context, created_container)

    utils.spawn_n(do_container_create)
```
该方法的主要功能函数是`do_container_create()`，其负责创建容器过程的实现。在`do_container_create()`中先处理卷相关的问题，调用函数（`/opt/stack/zun/zun/compute/manager.py`）
```python
def _wait_for_volumes_available(self, context, volumes, container,
                                timeout=60, poll_interval=1):
    start_time = time.time()
    try:
        volumes = itertools.chain(volumes)
        volume = next(volumes)
        while time.time() - start_time < timeout:
            if self.driver.is_volume_available(context, volume):
                volume = next(volumes)
            time.sleep(poll_interval)
    except StopIteration:
        return
    msg = _("Volumes did not reach available status after"
            "%d seconds") % (timeout)
    self._fail_container(context, container, msg, unset_host=True)
    raise exception.Conflict(msg)
```
该函数主要负责获取可用的卷，但是其中部分代码未实现，如用于判断卷是否可用的函数`is_volume_available`在q版（GitHub上的master版已经写了）还没有写具体的实现代码。该函数的流程大概是：先获取卷，然后在60秒内判断这些卷是否可用，并返回判断结果（目前好像只会返回True）。
接下来通过调用函数`_do_container_create()`（`/opt/stack/zun/zun/compute/manager.py`）来创建容器，`_do_container_create()`由装饰器`@wrap_container_event(prefix='compute')`（`/opt/stack/zun/zun/common/utils.py`）装饰，该装饰器生成事件状态，标记当前任务进度。`_do_container_create()`的代码如下所示：
```python
@wrap_container_event(prefix='compute')
def _do_container_create(self, context, container, requested_networks,
                         requested_volumes, pci_requests=None,
                         limits=None, reraise=False):
    LOG.debug('Creating container: %s', container.uuid)
    try:
        rt = self._get_resource_tracker()#获取可用资源情况
        # As sriov port also need to claim, we need claim pci port before
        # create sandbox.
        with rt.container_claim(context, container, pci_requests, limits):#索要资源
            sandbox = None#这个不知道代表什么
            if self.use_sandbox:
                sandbox = self._create_sandbox(context, container,
                                               requested_networks,
                                               reraise)
                if sandbox is None:
                    return

            created_container = self._do_container_create_base(
                context, container, requested_networks, requested_volumes,
                sandbox, limits, reraise)
            return created_container
    except Exception as e:
        # Other exception has handled in create sandbox and create base,
        # exception occured here only can be the claim failed.
        with excutils.save_and_reraise_exception(reraise=reraise):
            LOG.exception("Container resource claim failed: %s",
                          six.text_type(e))
            self._fail_container(context, container, six.text_type(e),
                                 unset_host=True)
        return
```
这个函数里首先查看可用的资源，然后根据索要容器需要的资源，如cpu、内存等。代码运行到
```python
created_container = self._do_container_create_base(
                context, container, requested_networks, requested_volumes,
                sandbox, limits, reraise)
```
开始进行容器的创建。`_do_container_create_base()`（`/opt/stack/zun/zun/compute/manager.py`）的代码如下：
```python
def _do_container_create_base(self, context, container, requested_networks,
                              requested_volumes, sandbox=None, limits=None,
                              reraise=False):
    self._update_task_state(context, container, consts.IMAGE_PULLING)
    repo, tag = utils.parse_image_name(container.image)
    image_pull_policy = utils.get_image_pull_policy(
        container.image_pull_policy, tag)
    image_driver_name = container.image_driver
    try:
        image, image_loaded = image_driver.pull_image(#获取容器镜像以及容器镜像的加载状态
            context, repo, tag, image_pull_policy, image_driver_name)
        image['repo'], image['tag'] = repo, tag
        if not image_loaded:
            self.driver.load_image(image['path'])
    except exception.ImageNotFound as e:
        with excutils.save_and_reraise_exception(reraise=reraise):
            LOG.error(six.text_type(e))
            self._do_sandbox_cleanup(context, container)
            self._fail_container(context, container, six.text_type(e))
        return
    except exception.DockerError as e:
        with excutils.save_and_reraise_exception(reraise=reraise):
            LOG.error("Error occurred while calling Docker image API: %s",
                      six.text_type(e))
            self._do_sandbox_cleanup(context, container)
            self._fail_container(context, container, six.text_type(e))
        return
    except Exception as e:
        with excutils.save_and_reraise_exception(reraise=reraise):
            LOG.exception("Unexpected exception: %s",
                          six.text_type(e))
            self._do_sandbox_cleanup(context, container)
            self._fail_container(context, container, six.text_type(e))
        return

    container.task_state = consts.CONTAINER_CREATING#容器创建状态修改为container_creating
    container.image_driver = image.get('driver')#设定镜像driver
    container.save(context)
    try:
        if image['driver'] == 'glance':
            self.driver.read_tar_image(image)
        container = self.driver.create(context, container, image,#创建容器
                                       requested_networks,
                                       requested_volumes)
        self._update_task_state(context, container, None)#更新任务状态
        return container
    except exception.DockerError as e:
        with excutils.save_and_reraise_exception(reraise=reraise):
            LOG.error("Error occurred while calling Docker create API: %s",
                      six.text_type(e))
            self._do_sandbox_cleanup(context, container)
            self._fail_container(context, container, six.text_type(e),
                                 unset_host=True)
        return
    except Exception as e:
        with excutils.save_and_reraise_exception(reraise=reraise):
            LOG.exception("Unexpected exception: %s",
                          six.text_type(e))
            self._do_sandbox_cleanup(context, container)
            self._fail_container(context, container, six.text_type(e),
                                 unset_host=True)
        return
```
该函数的流程为：更新任务状态为`IMAGE_PULLING`，设定镜像的相关信息，获取镜像，更新任务状态为`CONTAINER_CREATING`，调用docker api完成容器创建。
其中调用docker api在
```python
container = self.driver.create(context, container, image,
                                       requested_networks,
                                       requested_volumes)
```
这个create函数在`/opt/stack/zun/zun/container/docker/driver.py`中，具体代码如下：
```python
def create(self, context, container, image, requested_networks,#创建容器
           requested_volumes):
    sandbox_id = container.get_sandbox_id()

    with docker_utils.docker_client() as docker:#此处docker为一个DockerHTTPClient
        network_api = zun_network.api(context=context, docker_api=docker)#network_api为zun.network.kuryr_network.KuryrNetwork
        name = container.name
        LOG.debug('Creating container with image %(image)s name %(name)s',
                  {'image': image['image'], 'name': name})
        self._provision_network(context, network_api, requested_networks)
        binds = self._get_binds(context, requested_volumes)#卷相关，获取挂载信息
        kwargs = {
            'name': self.get_container_name(container),
            'command': container.command,
            'environment': container.environment,
            'working_dir': container.workdir,
            'labels': container.labels,
            'tty': container.interactive,
            'stdin_open': container.interactive,
        }
        if not sandbox_id:
            # Sandbox is not used so it is legitimate to customize
            # the container's hostname
            kwargs['hostname'] = container.hostname

        runtime = container.runtime if container.runtime\
            else CONF.container_runtime#CONF中的runtime为'runc'

        host_config = {}
        host_config['runtime'] = runtime
        host_config['binds'] = binds
        kwargs['volumes'] = [b['bind'] for b in binds.values()]
        if sandbox_id:
            host_config['network_mode'] = 'container:%s' % sandbox_id
            # TODO(hongbin): Uncomment this after docker-py add support for
            # container mode for pid namespace.
            # host_config['pid_mode'] = 'container:%s' % sandbox_id
            host_config['ipc_mode'] = 'container:%s' % sandbox_id
        else:
            self._process_networking_config(#容器网络配置
                context, container, requested_networks, host_config,
                kwargs, docker)
        if container.auto_remove:
            host_config['auto_remove'] = container.auto_remove
        if container.memory is not None:
            host_config['mem_limit'] = container.memory
        if container.cpu is not None:
            host_config['cpu_quota'] = int(100000 * container.cpu)
            host_config['cpu_period'] = 100000
        if container.restart_policy:
            count = int(container.restart_policy['MaximumRetryCount'])
            name = container.restart_policy['Name']
            host_config['restart_policy'] = {'Name': name,
                                             'MaximumRetryCount': count}
        if container.disk:#如果磁盘大小不为0，则将磁盘大小加入host_config
            disk_size = str(container.disk) + 'G'
            host_config['storage_opt'] = {'size': disk_size}

        kwargs['host_config'] = docker.create_host_config(**host_config)
        image_repo = image['repo'] + ":" + image['tag']
        response = docker.create_container(image_repo, **kwargs)#调用docker api创建容器，api文件：/usr/lib/python2.7/site-packages/docker/api/container.py
        container.container_id = response['Id']

        addresses = self._setup_network_for_container(
            context, container, requested_networks, network_api)
        container.addresses = addresses

        response = docker.inspect_container(container.container_id)#容器信息
        self._populate_container(container, response)
        container.save(context)
        return container
```
该函数流程为：创建DockerHTTPClient，获取Kuryr接口（网络相关），获取容器网络，初始化volume参数，设置各种容器参数，设置容器网络参数，调用docker api创建容器。注意该函数中的kwargs变量，这个变量包含了docker api所需的全部信息，后期可根据需求配置合适的kwargs，实现创建容器方式的变化。
docker中创建容器的函数如下所示：
```python
def create_container(self, image, command=None, hostname=None, user=None,
                         detach=False, stdin_open=False, tty=False, ports=None,
                         environment=None, volumes=None,
                         network_disabled=False, name=None, entrypoint=None,
                         working_dir=None, domainname=None, host_config=None,
                         mac_address=None, labels=None, stop_signal=None,
                         networking_config=None, healthcheck=None,
                         stop_timeout=None, runtime=None):
    if isinstance(volumes, six.string_types):
        volumes = [volumes, ]

    config = self.create_container_config(
        image, command, hostname, user, detach, stdin_open, tty,
        ports, environment, volumes,
        network_disabled, entrypoint, working_dir, domainname,
        host_config, mac_address, labels,
        stop_signal, networking_config, healthcheck,
        stop_timeout, runtime
    )
    return self.create_container_from_config(config, name)
```
这个函数的参数很多，具体解释如下：
```
image (str): The image to run
command (str or list): The command to be run in the container
hostname (str): Optional hostname for the container
user (str or int): Username or UID
detach (bool): Detached mode: run container in the background and return container ID
stdin_open (bool): Keep STDIN open even if not attached
tty (bool): Allocate a pseudo-TTY
ports (list of ints): A list of port numbers
environment (dict or list): A dictionary or a list of strings in the following format ``["PASSWORD=xxx"]`` or ``{"PASSWORD": "xxx"}``.
volumes (str or list): List of paths inside the container to use as volumes.
network_disabled (bool): Disable networking
name (str): A name for the container
entrypoint (str or list): An entrypoint
working_dir (str): Path to the working directory
domainname (str): The domain name to use for the container
host_config (dict): A dictionary created with :py:meth:`create_host_config`.
mac_address (str): The Mac Address to assign the container
labels (dict or list): A dictionary of name-value labels (e.g. ``{"label1": "value1", "label2": "value2"}``) or a list of names of labels to set with empty values (e.g. ``["label1", "label2"]``)
stop_signal (str): The stop signal to use to stop the container(e.g. ``SIGINT``).
stop_timeout (int): Timeout to stop the container, in seconds. Default: 10
networking_config (dict): A networking configuration generated by :py:meth:`create_networking_config`.
runtime (str): Runtime to use with this container.
healthcheck (dict): Specify a test to perform to check that the container is healthy.
```
该函数首先生成config，例如
```python
{'Cmd': None, 'NetworkDisabled': False, 'StdinOnce': False, 'Hostname': None, 'WorkingDir': None, 'StopSignal': None, 'StopTimeout': None, 'Entrypoint': None, 'Env': [], 'Runtime': None, 'OpenStdin': False, 'Volumes': {}, 'MacAddress': u'fa:16:3e:c4:b3:a6', 'Tty': False, 'Domainname': None, 'Healthcheck': None, 'Image': u'cirros:latest', 'Labels': {}, 'HostConfig': {'Binds': [], 'Runtime': 'runc', 'NetworkMode': u'1f0aeb31-c27b-489f-9c55-3562c366660f'}, 'ExposedPorts': None, 'User': None, 'AttachStdin': False, 'AttachStderr': True, 'NetworkingConfig': {'EndpointsConfig': {u'1f0aeb31-c27b-489f-9c55-3562c366660f': {'IPAMConfig': {'IPv4Address': u'192.168.1.15'}}}}, 'AttachStdout': True}
```
然后根据这个config来创建容器，这个过程由docker完成。创建完成后会返回一个response，示例`{u'Id': u'dd22a08af224cd3fb85f648dd57e368be7e9d9c426e60254f704b9b37d3a6015', u'Warnings': None}`。然后将容器连接到虚拟网络中，生成容器创建结果response，容器创建过程结束

