# TraceScope

Try TraceScope on [Github Pages](https://bittervan.github.io/TraceScope/) directly!

This is a tool to visualize a Ftrace log, as well as time spent on each function.

```
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo <function name> > set_graph_function
echo function_graph > current_tracer
echo 1 > tracing_on
cat trace

```

And you will get the ftrace of the function, something like

```
 5)               |  lru_add_fn() {
 5)   0.090 us    |    __rcu_read_lock();
 5)   0.090 us    |    folio_mapping();
 5)   0.090 us    |    __rcu_read_unlock();
 5)               |    __mod_lruvec_state() {
 5)   0.090 us    |      __mod_node_page_state();
 5)               |      __mod_memcg_lruvec_state() {
 5)   0.100 us    |        cgroup_rstat_updated();
 5)   0.280 us    |      }
 5)   0.650 us    |    }
 5)   0.100 us    |    __mod_zone_page_state();
 5)   0.100 us    |    mem_cgroup_update_lru_size();
 5)   1.800 us    |  }
```

And paste one function trace in the left frame, then click Generate Diagram above. You will see the calling tree of the function.

The color represents the ratio of the execution time, red is 100% and blue is 0%.
The saturation represents the ratio of the execution time within its caller, more saturate the color is, more time the function shares in its caller.
