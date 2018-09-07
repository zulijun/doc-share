> ZUN通过一个周期性的任务来对计算节点的资源进行统计

1. `/opt/stack/zun/zun/compute/manager.py`中的函数`inventory_host()`是资源统计的入口函数，定义如下

   ```python
   @periodic_task.periodic_task(run_immediately=True)
   def inventory_host(self, context):
       rt = self._get_resource_tracker()
       rt.update_available_resources(context)
   ```

   先获取一个类`ComputeNodeTracker`的对象，然后调用类`ComputeNodeTracker`的函数`update_available_resources()`

2. 函数`update_available_resources()`的定义如下

   ```python
       def update_available_resources(self, context):
           # Check if the compute_node is already registered
           node = self._get_compute_node(context)
           if not node:
               # If not, register it and pass the object to the driver
               numa_obj = self.container_driver.get_host_numa_topology()
               node = objects.ComputeNode(context)
               node.hostname = self.host
               node.numa_topology = numa_obj
               node.create(context)
               LOG.info('Node created for :%(host)s', {'host': self.host})
           self.container_driver.get_available_resources(node)
           self._setup_pci_tracker(context, node)
           self.compute_node = node
           self._update_available_resource(context)
           # NOTE(sbiswas7): Consider removing the return statement if not needed
           return node
   ```

   首先通过函数`_get_compute_node()`获取该计算节点对象，即类`ComputeNode`的一个对象。如果没有找到这个计算节点，那么就注册这个计算节点，并获取其相关的资源；如果成功获取该计算节点，那么先通过调用`container_driver.get_available_resources()`获取可用资源，然后调用`_update_available_resource()`来更新资源

3. 函数`get_available_resources()`的定义如下

   ```python
       def get_available_resources(self, node):
           numa_topo_obj = self.get_host_numa_topology()
           node.numa_topology = numa_topo_obj
           meminfo = self.get_host_mem()
           (mem_total, mem_free, mem_ava, mem_used) = meminfo
           node.mem_total = mem_total // units.Ki
           node.mem_free = mem_free // units.Ki
           node.mem_available = mem_ava // units.Ki
           node.mem_used = mem_used // units.Ki
           info = self.get_host_info()
           (total, running, paused, stopped, cpus,
            architecture, os_type, os, kernel_version, labels) = info
           node.total_containers = total
           node.running_containers = running
           node.paused_containers = paused
           node.stopped_containers = stopped
           node.cpus = cpus
           node.architecture = architecture
           node.os_type = os_type
           node.os = os
           node.kernel_version = kernel_version
           cpu_used = self.get_cpu_used()
           node.cpu_used = cpu_used
           node.labels = labels
   ```

   主要功能就是为计算节点的各种信息赋值，这个函数属于类`ContainerDriver`，这个类中有很多关于获取资源情况的函数

4. 函数`_update_available_resource()`的定义如下

   ```python
       def _update_available_resource(self, context):
   
           # if we could not init the compute node the tracker will be
           # disabled and we should quit now
           if self.disabled(self.host):
               return
   
           # Grab all containers assigned to this node:
           containers = objects.Container.list_by_host(context, self.host)
   
           # Now calculate usage based on container utilization:
           self._update_usage_from_containers(context, containers)
   
           # No migration for docker, is there will be orphan container? Nova has.
   
           cn = self.compute_node
   
           # update the compute_node
           self._update(cn)
           LOG.debug('Compute_service record updated for %(host)s',
                     {'host': self.host})
   ```

   首先获取获取该计算节点上的所有容器，然后通过调用函数`_update_usage_from_containers()`来从更新这些容器对资源的消耗情况

5. 函数`_update_usage_from_containers()`的定义如下

   ```python
       def _update_usage_from_containers(self, context, containers):
           """Calculate resource usage based on container utilization.
   
           This is different than the conatiner daemon view as it will account
           for all containers assigned to the local compute host, even if they
           are not currently powered on.
           """
           self.tracked_containers.clear()
   
           cn = self.compute_node
           # set some initial values, reserve room for host
           cn.cpu_used = 0
           cn.mem_free = cn.mem_total
           cn.mem_used = 0
           cn.running_containers = 0
   
           #import pdb;pdb.set_trace()
           for cnt in containers:
               self._update_usage_from_container(context, cnt)
   
           cn.mem_free = max(0, cn.mem_free)
   ```

   首先获取一个计算节点对象，然后通过调用函数`_update_usage_from_container()`对计算节点上的每一个容器更新资源的使用情况

6. 函数`_update_usage_from_container()`的定义如下

   ```python
       def _update_usage_from_container(self, context, container,
                                        is_removed=False):
           """Update usage for a single container."""
   
           uuid = container.uuid
           is_new_container = uuid not in self.tracked_containers
           is_removed_container = not is_new_container and is_removed
   
           if is_new_container:
               self.tracked_containers[uuid] = \
                   obj_base.obj_to_primitive(container)
               sign = 1
   
           if is_removed_container:
               self.tracked_containers.pop(uuid)
               sign = -1
   
           if is_new_container or is_removed_container:
               if self.pci_tracker:
                   self.pci_tracker.update_pci_for_container(context, container,
                                                             sign=sign)
   
               # new container, update compute node resource usage:
               self._update_usage(self._get_usage_dict(container), sign=sign)
   ```

   首先判断容器是新建的还是已经删除的，如果是新建的，则将标志位`sign`置为1，如果是已经删除的，则将标志位`sign`置为-1，这个值最终会影响容器的资源使用量是在已有的基础上增加还是减少。之后调用函数`_update_usage()`根据这个容器对资源的消耗来进行计算节点资源的更新

7. 函数`_update_usage()`的一个参数为函数`_get_usage_dict()`，定义如下

   ```python
       def _get_usage_dict(self, container, **updates):
           """Make a usage dict _update methods expect.
   
           Accepts an Container, and a set of updates.
           Converts the object to a dict and applies the updates.
   
           :param container: container as an object
           :param updates: key-value pairs to update the passed object.
   
           :returns: a dict with all the information from container updated
                     with updates
           """
           usage = {}
           # (Fixme): The Container.memory is string.
           memory = 0
           if container.memory:
               memory = int(container.memory[:-1])
           usage = {'memory': memory,
                    'cpu': container.cpu or 0}
   
           return usage
   ```

   该函数的作用是获取容器中的资源使用情况，并封装成一个字典

   接下来我们看看函数`_update_usage()`的定义

   ```python
           def _update_usage(self, usage, sign=1):
               mem_usage = usage['memory']
               cpus_usage = usage.get('cpu', 0)
               disk_usage = usage['disk']
   
               cn = self.compute_node
               cn.mem_used += sign * mem_usage
               cn.cpu_used += sign * cpus_usage
               cn.disk_used += sign * disk_usage
   
               # free ram may be negative, depending on policy:
               cn.mem_free = cn.mem_total - cn.mem_used
   
               cn.running_containers += sign * 1
   ```

   从变量`usage`中获取容器对各资源的使用量，然后更新计算节点的资源使用情况