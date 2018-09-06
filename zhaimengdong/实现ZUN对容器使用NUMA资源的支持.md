> 修改OpenStack ZUN相关流程，实现对容器使用NUMA资源的支持

1. 在针对`cpuset`的filter中加入资源限制limit，这个limit在进行claim时会用到

   ```python
           if container.cpu_policy == 'dedicated':
                if host_state.total_containers and (not pinned_cpus_flag):
                    return False
                else:
                    for numa_node in host_state.numa_topology.nodes:
                        if len(numa_node.cpuset) - len(
                                numa_node.pinned_cpus) > container.cpu and numa_node.mem_available > int(
                            container.memory[:-1]):
                            host_state.limits['cpuset'] = {'node': numa_node.id, 'cpuset_cpu': numa_node.cpuset, 'cpuset_cpu_pinned': numa_node.pinned_cpus, 'cpuset_mem': numa_node.mem_available}#新增
                            return True
                        else:
                            return False
    #/opt/stack/zun/zun/scheduler/filters/cpuset_filter.py
   ```

   当计算节点通过过滤时，计算节点的`limit`中增加NUMA节点的相关信息，包括NUMA节点的id，其中的core编号，已经用掉的core编号，以及可用的内存

2. 更新ZUN的API，加入参数`cpu_policy`，用来标记容器利用CPU的方式

   ```python
        'image': parameter_types.image_name,
        'command': parameter_types.command,
        'cpu': parameter_types.cpu,
        'cpu_policy': parameter_types.cpu_policy,#新增
        'memory': parameter_types.memory,
        'workdir': parameter_types.workdir,
        'auto_remove': parameter_types.auto_remove,
   #/opt/stack/zun/zun/api/controllers/v1/schemas/containers.py
   ```

   ```python
   cpu_policy = {
        'type': 'string',
        'enum': ['dedicated', 'shared']
    }
    #/opt/stack/zun/zun/common/validation/parameter_types.py
   ```

   先在API中加入参数`cpu_policy`，然后详细定义`cpu_policy`

3. 容器进入创建流程后，会进行资源的申请，在类`ComputeNodeTracker`的成员函数`container_claim()`中会生成类`Claim`的一个对象

   ```python
   claim = claims.Claim(context, container, self, self.compute_node,
                                pci_requests, limits=limits)
   ```

   在类`Claim`的初始化过程中需要进行NUMA资源的测试和claim

   ```python
   def __init__(self, context, container, tracker, resources, pci_requests,
                     limits=None):
            super(Claim, self).__init__()
            # Stash a copy of the container at the current point of time
            self.container = container.obj_clone()
            self._numa_topology_loaded = False
            self.tracker = tracker
            self.context = context
            self._pci_requests = pci_requests
    
            # Check claim at constructor to avoid mess code
            # Raise exception ComputeResourcesUnavailable if claim failed
            self._claim_test(resources, limits)
            if container.cpu_policy == 'dedicated':#新增
                self.claim_cpuset_cpu_for_container(container, limits)
                self.claim_cpuset_mem_for_container(container, limits)
   #/opt/stack/zun/zun/compute/claims.py
   ```

   首先进行claim test

   ```python
       def _claim_test(self, resources, limits=None):
           """Test if this claim can be satisfied.
   
           With given available resources and optional oversubscription limits
   
           This should be called before the compute node actually consumes the
           resources required to execute the claim.
   
           :param resources: available local compute node resources
           :returns: Return true if resources are available to claim.
           """
           if not limits:
               limits = {}
   
           # If an individual limit is None, the resource will be considered
           # unlimited:
           memory_limit = limits.get('memory')
           cpu_limit = limits.get('cpu')
           cpuset_limit = limits.get('cpuset', None)#新增
   
           LOG.info('Attempting claim: memory %(memory)s, '
                    'cpu %(cpu).02f CPU',
                    {'memory': self.memory, 'cpu': self.cpu})
   
           reasons = [self._test_memory(resources, memory_limit),
                      self._test_cpu(resources, cpu_limit),
                      self._test_pci()]
           if cpuset_limit:#新增
               reasons.append(self._test_cpuset_cpu(resources, cpuset_limit))
               reasons.append(self._test_cpuset_mem(resources, cpuset_limit))
           reasons = [r for r in reasons if r is not None]
           if len(reasons) > 0:
               raise exception.ResourcesUnavailable(reason="; ".join(reasons))
   
           LOG.info('Claim successful')
    #/opt/stack/zun/zun/compute/claims.py
   ```

   如果limit中有`cpuset`，则需要对NUMA节点中的可用core数量和可用内存量进行前期测试

   定义函数`_test_cpuset_cpu()`

   ```python
       def _test_cpuset_cpu(self, resources, limit):
           type_ = _("cpuset_cpu")
           unit = "core"
           total = len(limit['cpuset_cpu'])
           used = len(resources.numa_topology.nodes[limit['node']].pinned_cpus)
           requested = self.cpu
   
           return self._test(type_, unit, total, used, requested, len(limit['cpuset_cpu']))
    #/opt/stack/zun/zun/compute/claims.py
   ```

   `total`为NUMA节点内的core总量，`used`为已经用掉的core数量，`requested`为容器需要的core数量，然后通过函数`_test()`来进行资源是否足够的判断

   定义函数`_test_cpuset_mem()`

   ```python
       def _test_cpuset_mem(self, resources, limit):
           type_ = _("cpuset_mem")
           unit = "M"
           total = resources.numa_topology.nodes[limit['node']].mem_total
           used = 0
           requested = self.memory
   
           return self._test(type_, unit, total, used, requested, limit['cpuset_mem'])
   #/opt/stack/zun/zun/compute/claims.py
   ```

   `total`为NUMA节点内的内存总量，`used`为0，因为函数`_test()`中是通过limit与used的差值来获取可用量的，而调用`_test()`时给limit的实参是NUMA节点内的可用内存总量，所以`used`为0的时候才能在函数`_test()`中获得可用内存总量

   接下来要进行资源的claim

   定义函数`claim_cpuset_cpu_for_container()`

   ```python
       def claim_cpuset_cpu_for_container(self, container, limits):
           avaliable_cpu = list(set(limits['cpuset']['cpuset_cpu'])-set(limits['cpuset']																['cpuset_cpu_pinned']))
           cpuset_cpu_usage = random.sample(avaliable_cpu, int(self.cpu))
           container.cpuset_cpus = ','.join([str(x) for x in cpuset_cpu_usage])
   #/opt/stack/zun/zun/compute/claims.py
   ```

   该函数从limit中获取NUMA节点的core编号和已经用掉的core编号，计算出可用的core编号，根据容器需要的core数量，从中随机选出相同数量的core，并将这些core的编号赋值给容器的`cpuset_cpus`

   定义函数`claim_cpuset_mem_for_container()`

   ```python
       def claim_cpuset_mem_for_container(self, container, limits):
           container.cpuset_mems = str(limits['cpuset']['node'])
   #/opt/stack/zun/zun/compute/claims.py
   ```

   该函数从limit中获取这个容器将要分配的内存对应的NUMA节点编号，并将这个值赋值给容器的`cpuset_mems`

4. 接下来需要对资源用量进行更新，主要在类`ComputeNodeTracker`的函数`_update_usage()`

   ```python
       def _update_usage(self, usage, sign=1):
           mem_usage = usage['memory']
           cpus_usage = usage.get('cpu', 0)
           cpuset_cpus_usage = None#新增
           numa_node_id = 0#新增
           if 'cpuset_cpus' in usage.keys():
               cpuset_cpus_usage = usage['cpuset_cpus']
               numa_node_id = usage['node']
   
           cn = self.compute_node
           numa_topology = cn.numa_topology.nodes#新增
           cn.mem_used += sign * mem_usage
           cn.cpu_used += sign * cpus_usage
   
           # free ram may be negative, depending on policy:
           cn.mem_free = cn.mem_total - cn.mem_used
   
           cn.running_containers += sign * 1
   
           if cpuset_cpus_usage:#新增
               for numa_node in numa_topology:
                   if numa_node.id == numa_node_id:
                       numa_node.mem_available = numa_node.mem_available - mem_usage * sign
                       if sign > 0:
                           numa_node.pin_cpus(cpuset_cpus_usage)
                       else:
                           numa_node.unpin_cpus(cpuset_cpus_usage)
     #/opt/stack/zun/zun/compute/compute_node_tracker.py
   ```

   如果`usage`中有键`cpuset_cpus`，则对`cpuset_cpus_usage`和`numa_node_id`进行赋值，这两个值分别代表分配给容器的core编号和NUMA节点编号，接下来调整NUMA节点中的可用内存量和`pinned_cpus`的值。该函数中的`sign`代表容器是创建还是删除，创建为1，删除为-1