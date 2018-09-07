1. `/opt/stack/zun/zun/compute/api.py`中的函数`container_create()`是创建容器的一个入口函数，其中有这么一部分代码

   ```python
   host_state = None
   try:
       host_state = self._schedule_container(context, new_container,
                                             extra_spec)
   ```

   这部分代码中的`_schedule_container()`为scheduler的入口函数，作用是选出一个合适的计算节点来创建容器

2. 函数`_schedule_container()`的代码如下

   ```python
       def _schedule_container(self, context, new_container, extra_spec):
           dests = self.scheduler_client.select_destinations(context,
                                                             [new_container],
                                                             extra_spec)
           return dests[0]
   ```

   该函数中调用了类`SchedulerClient`的函数`select_destinations()`，这个函数会返回一个计算节点

   ```python
       def select_destinations(self, context, containers, extra_spec):
           return self.driver.select_destinations(context, containers, extra_spec)
   ```

   这个函数调用类`FilterScheduler`的函数`select_destinations()`

3. 类`FilterScheduler`的函数`select_destinations()`的定义如下

   ```python
       def select_destinations(self, context, containers, extra_spec):
           """Selects destinations by filters."""
           dests = []
           for container in containers:
               host = self._schedule(context, container, extra_spec)
               host_state = dict(host=host.hostname, nodename=None,
                                 limits=host.limits)
               dests.append(host_state)
   
           if len(dests) < 1:
               reason = _('There are not enough hosts available.')
               raise exception.NoValidHost(reason=reason)
   
           return dests
   ```

   该函数的作用是通过filters来选择合适的计算节点。对于每一个待创建的容器，通过调用函数`_schedule()`进行计算节点的选择，该函数的定义如下

   ```python
       def _schedule(self, context, container, extra_spec):
           """Picks a host according to filters."""
           services = self._get_services_by_host(context)
           nodes = objects.ComputeNode.list(context)
           hosts = services.keys()
           nodes = [node for node in nodes if node.hostname in hosts]
           host_states = self.get_all_host_state(nodes, services)
           hosts = self.filter_handler.get_filtered_objects(self.enabled_filters,
                                                            host_states,
                                                            container,
                                                            extra_spec)
           if not hosts:
               msg = _("Is the appropriate service running?")
               raise exception.NoValidHost(reason=msg)
   
           return random.choice(hosts)
   ```

   需要关注的是函数`get_filtered_objects()`，其中有个参数`enabled_filters`，这个值代表本次选择计算节点用到的filters

4. 函数`get_filtered_objects()`是类`HostFilterHandler`的成员函数，类`HostFilterHandler`继承自类`BaseFilterHandler`，而且并没有重写函数`get_filtered_objects()`，所以函数`get_filtered_objects()`其实还是类`BaseFilterHandler`的成员函数，定义如下

   ```python
       def get_filtered_objects(self, filters, objs, container, extra_spec,
                                index=0):
           list_objs = list(objs)
           LOG.debug("Starting with %d host(s)", len(list_objs))
           part_filter_results = []
           full_filter_results = []
           log_msg = "%(cls_name)s: (start: %(start)s, end: %(end)s)"
           for filter_ in filters:
               if filter_.run_filter_for_index(index):
                   cls_name = filter_.__class__.__name__
                   start_count = len(list_objs)
                   objs = filter_.filter_all(list_objs, container, extra_spec)
                   if objs is None:
                       LOG.debug("Filter %s says to stop filtering", cls_name)
                       return
                   list_objs = list(objs)
                   end_count = len(list_objs)
                   part_filter_results.append(log_msg % {"cls_name": cls_name,
                                                         "start": start_count,
                                                         "end": end_count})
                   if list_objs:
                       remaining = [(getattr(obj, "host", obj),
                                     getattr(obj, "nodename", ""))
                                    for obj in list_objs]
                       full_filter_results.append((cls_name, remaining))
                   else:
                       LOG.info("Filter %s returned 0 hosts", cls_name)
                       full_filter_results.append((cls_name, None))
                       break
                   LOG.debug("Filter %(cls_name)s returned "
                             "%(obj_len)d host(s)",
                             {'cls_name': cls_name, 'obj_len': len(list_objs)})
           ……
           return list_objs
   ```

   对于每一个filter，调用函数`filter_all()`进行筛选，其中该函数传入的参数`list_objs`为计算节点列表，顺便说一下，filter继承自类`BaseHostFilter`，类`BaseHostFilter`又继承自类`BaseFilter`，由于每个filter和自类`BaseHostFilter`都没有重写函数`filter_all()`，所以其实函数`filter_all()`是在类`BaseFilter`中定义的。

5. 函数`filter_all()`定义如下

   ```python
       def filter_all(self, filter_obj_list, container, extra_spec):
           """Yield objects that pass the filter.
   
           Can be overridden in a subclass, if you need to base filtering
           decisions on all objects.  Otherwise, one can just override
           _filter_one() to filter a single object.
           """
           for obj in filter_obj_list:
               if self._filter_one(obj, container, extra_spec):
                   yield obj
   ```

   对于每一个计算节点，都通过函数`_filter_one()`来进行判断是否符合要求，这个函数是类`BaseHostFilter`的成员函数，定义如下

   ```python
       def _filter_one(self, obj, filter_properties, extra_spec):
           """Return True if the object passes the filter, otherwise False."""
           return self.host_passes(obj, filter_properties, extra_spec)
   ```

   其中调用了函数`host_passes()`来进行计算节点的选择，这个函数在每个filter中都会重写，里面具体定义了计算节点的过滤规则