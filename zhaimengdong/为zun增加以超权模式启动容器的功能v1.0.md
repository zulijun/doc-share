## 为zun增加以超权模式启动容器的功能v1.0

---

1. 我们打算在`zun api`中新增一个字段`privileged`来标记创建容器时是否为超权模式，该字段接受`boolean`型变量  

   首先在`/opt/stack/zun/zun/common/validation/parameter_types.py`中添加新的变量`privileged`

   ```python
   privileged = {
       'type': ['boolean', 'null'],
   }
   ```

   然后在`/opt/stack/zun/zun/api/controllers/v1/schemas/containers.py`中的`_container_properties`添加键值对

   ```
   'privileged': parameter_types.privileged
   ```

   这样`zun url api`就可以接受参数`privileged`

2. 在`/opt/stack/zun/zun/api/controllers/v1/containers.py`中的`post()`函数中添加相关代码来处理`api`传入的`privileged`

      ```python
      ···
      privileged = container_dict.pop('privileged')#传入的参数都在container_dict中，这里把privileged的值提取出来
      new_container = objects.Container(context, **container_dict)
      new_container.create(context)
      
      kwargs = {}
      kwargs['extra_spec'] = extra_spec
      kwargs['requested_networks'] = requested_networks
      kwargs['requested_volumes'] = requested_volumes
      kwargs['privileged'] = privileged#在kwargs中加入privileged键值对，之后kwargs将被传递给zun compute相关函数来进行容器的创建
      ···
      ```

      

3. 修改`/opt/stack/zun/zun/compute/api.py`中的`container_create()`函数，添加形参`privileged`

      ```python
      def container_create(self, context, new_container, extra_spec,
                               requested_networks, requested_volumes, run,
                               privileged, pci_requests=None):
      ```

      并在该函数中修改对`rpcapi.container_create()`的调用

      ```python
      self.rpcapi.container_create(context, host_state['host'],
                                           new_container, host_state['limits'],
                                           requested_networks, requested_volumes,
                                           run, privileged, pci_requests)
      ```

      

4. 修改`/opt/stack/zun/zun/compute/rpcapi.py`中的`container_create()`函数的形参

      ```python
      def container_create(self, context, host, container, limits,
                               requested_networks, requested_volumes, run,
                               privileged, pci_requests):
      ```

      并修改其中对`_cast()`的调用

      ```python
      self._cast(host, 'container_create', limits=limits,
                         requested_networks=requested_networks,
                         requested_volumes=requested_volumes,
                         container=container,
                         run=run, privileged=privileged,
                         pci_requests=pci_requests)
      ```

5. 修改`/opt/stack/zun/zun/compute/manager.py`中的`container_create()`函数的参数，加入形参`privileged`

      ```python
      def container_create(self, context, limits, requested_networks,
                               requested_volumes, container, run, privileged, pci_requests=None):
      ```

      并修改其中对`_do_container_create()`的调用

      ```python
      created_container = self._do_container_create(
                      context, container, requested_networks, requested_volumes, privileged,
                      pci_requests, limits)
      ```

6. 修改`/opt/stack/zun/zun/compute/manager.py`中的`_do_container_create()`函数的参数，加入形参`privileged`

      ```python
      def _do_container_create(self, context, container, requested_networks,
                                   requested_volumes, privileged, pci_requests=None,
                                   limits=None, reraise=False):
      ```

      并修改其中对`_do_container_create_base()`的调用

      ```python
      created_container = self._do_container_create_base(
                          context, container, requested_networks, requested_volumes,
                          privileged, sandbox, limits, reraise)
      ```

      

7. 修改`/opt/stack/zun/zun/compute/manager.py`中的`_do_container_create_base()`的参数，加入形参`privileged`

      ```python
      def _do_container_create_base(self, context, container, requested_networks,
                                        requested_volumes, privileged, sandbox=None, limits=None,
                                        reraise=False):
      ```

      并修改其中对`driver.create()`的调用

      ```python
      container = self.driver.create(context, container, image,
                                                 requested_networks,
                                                 requested_volumes, privileged)
      ```

      

8. 修改`/opt/stack/zun/zun/container/docker/driver.py`中的`create()`函数的参数，加入形参`privileged`

      ```python
      def create(self, context, container, image, requested_networks,
                     requested_volumes, privileged):
      ```

      将`privileged`加入到`host_config`中

      ```python
      host_config['privileged'] = privileged
      ```

      接下来在

      ```python
      kwargs['host_config'] = docker.create_host_config(**host_config)
      ```

      通过函数`create_host_config()`将`host_config`以一个字典的形式加入`kwargs`中，`kwargs`将被用于调用`docker api`时进行传参

      
