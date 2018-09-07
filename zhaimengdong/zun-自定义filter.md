### 创建新的filter

由于业务需求，对ZUN现有的容器调度策略需要做一些修改，本着尽量少改动ZUN现有代码的原则，我们对ZUN的scheduler添加一个新的filter，名字为`CPUSETFilter`

1. 在`/opt/stack/zun/zun/scheduler/filters`中新建文件`cpuset_filter.py`

   ```python
   # Copyright (c) 2017 OpenStack Foundation
   # All Rights Reserved.
   #
   #    Licensed under the Apache License, Version 2.0 (the "License"); you may
   #    not use this file except in compliance with the License. You may obtain
   #    a copy of the License at
   #
   #         http://www.apache.org/licenses/LICENSE-2.0
   #
   #    Unless required by applicable law or agreed to in writing, software
   #    distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
   #    WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
   #    License for the specific language governing permissions and limitations
   #    under the License.
   
   from oslo_log import log as logging
   
   from zun.scheduler import filters
   
   LOG = logging.getLogger(__name__)
   
   
   class CPUSETFilter(filters.BaseHostFilter):
       """Filter the host by cpu and memory request of cpuset"""
   
       run_filter_once_per_request = True
   
       def host_passes(self, host_state, container, extra_spec):
           mem_available = host_state.mem_free - host_state.mem_used
           pinned_cpus_flag = False
           for numa_node in host_state.numa_topology.nodes:
               if numa_node.pinned_cpus:
                   pinned_cpus_flag = True
   
           if container.cpu_policy == 'dedicated':
               if host_state.total_containers and (not pinned_cpus_flag):
                   return False
               else:
                   for numa_node in host_state.numa_topology.nodes:
                       if len(numa_node.cpuset) - len(
                               numa_node.pinned_cpus) > container.cpu and numa_node.mem_available > int(
                               container.memory[:-1]):
                           return True
                       else:
                           return False
           else:
               container.cpu_policy = 'shared'
               if not pinned_cpus_flag:
                   if container.memory:
                       if mem_available >= container.memory:
                           return True
                       else:
                           return False
                   else:
                       return True
               else:
                   return False
   
   ```

   filter必须继承类`filters.BaseHostFilter`，并重写函数`host_passes()`，`host_passes()`中定义了具体的过滤规则，在此就不展开解释这个filter的过滤规则了

2. 在`/opt/stack/zun/zun/conf/scheduler.py`中的

   ```python
       cfg.ListOpt("enabled_filters",
                   default=[
                       "CPUFilter",
                       "RamFilter",
                       "ComputeFilter",
                       "CPUSETFilter",#新添加的
                       ],
                   ……
   ```

   `default`中添加新的filter的名字`CPUSETFilter`

   

### scheduler中调用filter的流程

1. 在类`FilterScheduler`的初始化过程中，对filter进行了初始化

   ```python
       def __init__(self):
           super(FilterScheduler, self).__init__()
           self.filter_handler = filters.HostFilterHandler()
           filter_classes = self.filter_handler.get_matching_classes(
               CONF.scheduler.available_filters)
           self.filter_cls_map = {cls.__name__: cls for cls in filter_classes}
           self.filter_obj_map = {}
           self.enabled_filters = self._choose_host_filters(self._load_filters())
   ```

2. 首先从`/opt/stack/zun/zun/conf/scheduler.py`中的`cfg.ListOpt`中获取`available_filters`

   ```python
   cfg.MultiStrOpt("available_filters",
                       default=["zun.scheduler.filters.all_filters"],
                       ……
   ```

   从定义可以看出，`available_filters`代表的是所有filter类。然后生成一个字典，里面是filter类名和类的键值对

3. 接下来要初始化类`FilterScheduler`的`enabled_filters`，

   ```python
   self.enabled_filters = self._choose_host_filters(self._load_filters())
   ```

   这个语句中调用了多个函数，我们来一一进行分析。首先是`_load_filters()`，这个函数的定义如下

   ```python
       def _load_filters(self):
           return CONF.scheduler.enabled_filters
   ```

   返回`/opt/stack/zun/zun/conf/scheduler.py`中定义的filter类

   ```python
   cfg.ListOpt("enabled_filters",
                   default=[
                       "CPUFilter",
                       "RamFilter",
                       "ComputeFilter",
                       "CPUSETFilter",
                       ],
                       ……
   ```

   然后是函数`_choose_host_filters()`，定义如下

   ```python
       def _choose_host_filters(self, filter_cls_names):
           """Choose good filters
   
           Since the caller may specify which filters to use we need
           to have an authoritative list of what is permissible. This
           function checks the filter names against a predefined set
           of acceptable filters.
           """
           if not isinstance(filter_cls_names, (list, tuple)):
               filter_cls_names = [filter_cls_names]
   
           good_filters = []
           bad_filters = []
           for filter_name in filter_cls_names:
               if filter_name not in self.filter_obj_map:
                   if filter_name not in self.filter_cls_map:
                       bad_filters.append(filter_name)
                       continue
                   filter_cls = self.filter_cls_map[filter_name]
                   self.filter_obj_map[filter_name] = filter_cls()
               good_filters.append(self.filter_obj_map[filter_name])
           if bad_filters:
               msg = ", ".join(bad_filters)
               raise exception.SchedulerHostFilterNotFound(filter_name=msg)
           return good_filters
   ```

   筛选出可用的filter类，并生成相应的对象，返回这些对象组成的列表。

   至此`enabled_filters`初始化完成

4. 类`FilterScheduler`中的函数`_schedule`调用`get_filtered_objects()`来进行计算节点的筛选，传入的参数`filters`就是类`FilterScheduler`的`enabled_filters`，函数`get_filtered_objects()`中跟filter有关的部分如下

   ```python
           for filter_ in filters:
               if filter_.run_filter_for_index(index):
                   cls_name = filter_.__class__.__name__
                   print 'cls_name: ', cls_name
                   start_count = len(list_objs)
                   objs = filter_.filter_all(list_objs, container, extra_spec)
                   ……
   ```

   通过调用每个filter对象的`filter_all()`来实现对计算节点的筛选



> 说明：上边提到的计算节点其实是`/opt/stack/zun/zun/schedulter/host_state.py`中的类`HostState`的对象，如果需要计算节点额外的信息，需要在`__init__()`中初始化这个变量，并在`_update_from_compute_node()`中为这个变量进行赋值