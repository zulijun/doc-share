> 由于业务需求，需要修改计算节点的资源模型，针对CPU和内存资源，需要将粒度细化到NUMA节点，为此在`/opt/stack/zun/zun/objects/numa.py`中的类`NUMANode`新增两个参数`mem_total`和`mem_available`，资源的统计过程也需要进行相应的修改

1. `/opt/stack/zun/zun/compute/compute_node_tracker.py`中的函数`update_available_resources()`可以得到计算节点的NUMA信息

   ```python
   numa_obj = self.container_driver.get_host_numa_topology()
   ```

2. 函数`get_host_numa_topology()`的定义如下

   ```python
       def get_host_numa_topology(self):
           numa_topo_obj = objects.NUMATopology()
           os_capability_linux.LinuxHost().get_host_numa_topology(numa_topo_obj)
           return numa_topo_obj
       #/opt/stack/zun/zun/container/driver.py
   ```

   首先创建一个类`NUMATopology`的对象，然后通过调用函数`get_host_numa_topology()`来进行`NUMATopology`的获取

3. 函数`get_host_numa_topology()`的定义如下

   ```python
       def get_host_numa_topology(self, numa_topo_obj):
           # Replace this call with a more generic call when we obtain other
           # NUMA related data like memory etc.
           #import pdb;pdb.set_trace()
           cpu_info = self.get_cpu_numa_info()
           mem_info = self.get_mem_numa_info()#新增
           floating_cpus = utils.get_floating_cpu_set()
           numa_node_obj = []
           for cpu, mem_total in zip(cpu_info.items(), mem_info):#修改
               node, cpuset = cpu#修改
               numa_node = objects.NUMANode()
               if floating_cpus:
                   allowed_cpus = set(cpuset) - (floating_cpus & set(cpuset))
               else:
                   allowed_cpus = set(cpuset)
               numa_node.id = node
               # allowed_cpus are the ones allowed to pin on.
               # Rest of the cpus are assumed to be floating
               # in nature.
               numa_node.cpuset = allowed_cpus
               numa_node.pinned_cpus = set([])
               numa_node.mem_total = mem_total#新增
               numa_node.mem_available = mem_total#新增
               numa_node_obj.append(numa_node)
           numa_topo_obj.nodes = numa_node_obj
   #/opt/stack/zun/zun/container/os_capability/host_capability.py
   ```

   新增变量`mem_info`，用来表示各NUMA节点的总内存情况，这个值是通过新添加的函数`get_mem_numa_info()`来实现的，定义如下

   ```python
       def get_mem_numa_info(self):
           try:
               output = utils.execute('numactl', '-H')
           except exception.CommandError:
               LOG.info("There was a problem while executing numactl -H, Try again without the online column.")
   
           sizes = re.findall("size\: \d*", str(output))
           mem_numa = []
           for size in sizes:
               mem_numa.append(int(size.split(' ')[1]))
           return mem_numa
       #/opt/stack/zun/zun/container/os_capability/linux/os_capability_linux.py
   ```

   获取Linux命令`numactl -H`的输出结果，从结果中分析出各NUMA节点的内存情况，并以列表的形式返回

   之后对于每一个NUMA节点，初始化其`mem_total`和`mem_available`，这两个值都为该NUMA节点内存的最大值，至此完成计算节点`NUMATopology`信息的初始化

4. 在类`ComputeNodeTracker`的函数`_get_usage_dict()`中将容器使用的NUMA节点中的CPU资源和NUMA节点编号加入变量`usage`

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
   
           if container.cpuset_cpus:
               usage['cpuset_cpus'] = set([int(i) for i in container.cpuset_cpus.split(',')])
               usage['node'] = int(container.cpuset_mems)
   
           return usage
    #/opt/stack/zun/zun/compute/compute_node_tracker.py
   ```

   `container`的变量`cpuset_cpus`为Unicode字符串，需要做一定的修改来转变成包含列表的集合，方便之后进行处理

5. 在类`ComputeNodeTracker`的函数`_update_usage()`中实现了根据容器占用资源对NUMA节点的资源信息进行更新的操作，具体定义如下

   ```python
       def _update_usage(self, usage, sign=1):
           mem_usage = usage['memory']
           cpus_usage = usage.get('cpu', 0)
           cpuset_cpus_usage = None
           numa_node_id = 0
           if 'cpuset_cpus' in usage.keys():#新增
               cpuset_cpus_usage = usage['cpuset_cpus']
               numa_node_id = usage['node']
   
           cn = self.compute_node
           numa_topology = cn.numa_topology.nodes
           cn.mem_used += sign * mem_usage
           cn.cpu_used += sign * cpus_usage
   
           # free ram may be negative, depending on policy:
           cn.mem_free = cn.mem_total - cn.mem_used
   
           cn.running_containers += sign * 1
   
           if cpuset_cpus_usage:#新增
               for numa_node in numa_topology:
                   if numa_node.id == numa_node_id:
                       numa_node.mem_available = numa_node.mem_total - mem_usage * sign
                       if sign > 0:
                           numa_node.pin_cpus(cpuset_cpus_usage)
                       else:
                           numa_node.unpin_cpus(cpuset_cpus_usage)
    #/opt/stack/zun/zun/compute/compute_node_tracker.py
   ```

   如果`usage`中有键`cpuset_cpus`，则说明容器对NUMA资源有占用，将NUMA节点的可用内存减去容器消耗的内存，并通过函数`pin_cpus()`和`unpin_cpus()`来对NUMA节点内的核心进行增删操作