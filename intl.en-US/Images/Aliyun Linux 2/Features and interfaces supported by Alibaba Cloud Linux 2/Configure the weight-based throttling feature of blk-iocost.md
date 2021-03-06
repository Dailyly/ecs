# Configure the weight-based throttling feature of blk-iocost

Alibaba Cloud Linux 2 provides the weight-based throttling feature \(blk-iocost\) based on the cost model in the kernel version 4.19.81-17.al7.x86\_64 and later. blk-iocost improves the weight-based disk throttling feature of the I/O subsystem \(blkcg\) within a kernel. Both cgroup v1 and cgroup v2 interfaces support blk-iocost in Alibaba Cloud Linux 2. This topic describes the interfaces that implement throttling.

## Interface description

|Interface|Description|Configuration item|
|---------|-----------|------------------|
|cost.qos|A read/write interface whose file is stored only in the root group of blkcg. The full name of the file is `blkio.cost.qos` in cgroup v1 and `io.cost.qos` in cgroup v2. This interface provides the blk-iocost feature and limits the rate of I/O quality of service \(QoS\) based on the latency weight.

After blk-iocost is enabled, the kernel calculates the proportion of requests that exceed the read and write latency `rlat|wlat` in all requests. When the proportion is greater than `rlat|wlat`, the kernel considers the disk to be saturated and reduces the number of requests sent to the disk. By default, the value of `rlat|wlat` is set to 0, indicating that the blk-iocost feature is disabled.

|Each line of configuration in the interface file starts with the Major and Minor numbers of the disk in the `MAJ:MIN` format, followed by the following configuration items: -   enable: specifies whether to enable the blk-iocost controller to enable blk-iocost. The default value 0 indicates that blk-iocost is disabled. A value of 1 indicates that blk-iocost is enabled.
-   ctrl: the control mode. Valid values: `auto` and `user`. When the control mode is set to `auto`, the kernel automatically detects the disk type and uses built-in parameters. When the control mode is set to `user`, you must specify the following QoS control parameters:
-   rpct: the read latency percentile. Valid values: 0 to 100.
-   rlat: the read latency threshold. Unit: microseconds.
-   wpct: the write latency percentile. Valid values: 0 to 100.
-   wlat: the write latency threshold. Unit: microseconds.
-   min: the minimum scaling percentage. Valid values: 1 to 10000.
-   max: the maximum scaling percentage. Valid values: 1 to 10000. |
|cost.model|A read/write interface whose file is stored only in the root group of blkcg. The full name of the interface file is `blkio.cost.model` in cgroup v1 and `io.cost.model` in cgroup v2. The interface is used to set the cost model.|Each line of configuration in the interface file starts with the Major and Minor numbers of the disk in the `MAJ:MIN` format, followed by the following configuration items: -   ctrl: the control mode. It specifies whether to allow the user to enter model parameters. Valid values: `auto` and `user`.
-   model: the model parameter. Valid value: `linear`. You must define the following modeling parameters when the value of the model parameter is `linear`:

    -   \[r\|w\]bps: the maximum sequential I/O throughput.
    -   \[r\|w\]seqiops: the sequential input/output operations per second \(IOPS\).
    -   \[r\|w\]randiops: the random IOPS.
**Note:** You can use the tools/cgroup/iocost\_coef\_gen.py script in the kernel source code to generate the preceding parameters and then write these parameters to the interface file of cost.model to configure the cost model. |
|cost.weight|A read/write interface whose file is stored only in the sub-group of blkcg. The full name of the interface file is `blkio.cost.weight` in cgroup v1 and `io.cost.weight` in cgroup v2. This interface is used to set the weight of a sub-group. Default value: 100. Valid values: 1 to 10000. The interface can be used to set a weight for each disk or change the default weight of a sub-group.|-   If you set the weight of the interface to `<weight>`, the default weight of blkcg is changed to <weight\>.
-   If you set the port number and weight of the interface to `MAJ:MIN <weight>`, the weight of blkcg on the MAJ:MIN disk is changed to <weight\>. |

## Precautions

The blk-iocost feature allows the system to automatically configure IO monitoring \(`ctrl=auto`\). To use this feature, you must disable the `rotational` interface \(echo 0 \>\) for ultra disks, standard SSDs, enhanced SSDs, or local NVMe SSDs.

```
echo 0 > /sys/block/[$DISK_NAME]/queue/rotational    # Replace [$DISK_NAME] with the actual disk name.
```

## Example 1

Use the cost.qos interface to enable the blk-iocost feature for the `254:48` disk. If more than 95% of requests have a latency `(rlat|wlat)` of more than 5 milliseconds, the disk is considered to be saturated. The kernel will adjust the speed of the disk to send requests to fall within the interval from 50% to 150% of the original speed. The following commands are used for the cgroup v1 and cgroup v2 interfaces:

-   The command for cgroup v1:

    ```
    echo "254:48 enable=1 ctrl=user rpct=95.00 rlat=5000 wpct=95.00 wlat=5000 min=50.00 max=150.00" > /sys/fs/cgroup/blkio/blkio.cost.qos
    ```

-   The command for cgroup v2:

    ```
    echo "254:48 enable=1 ctrl=user rpct=95.00 rlat=5000 wpct=95.00 wlat=5000 min=50.00 max=150.00" > /sys/fs/cgroup/io.cost.qos
    ```


## Example 2

Use the `cost.model` interface to configure a model on the `254:48` disk based on the `linear` modeling parameters. The following commands are used for the cgroup v1 and cgroup v2 interfaces:

-   The command for cgroup v1:

    ```
    echo "254:48 ctrl=user model=linear rbps=2706339840 rseqiops=89698 rrandiops=110036 wbps=1063126016 wseqiops=135560 wrandiops=130734" > /sys/fs/cgroup/blkio/blkio.cost.model
    ```

-   The command for cgroup v2:

    ```
    echo "254:48 ctrl=user model=linear rbps=2706339840 rseqiops=89698 rrandiops=110036 wbps=1063126016 wseqiops=135560 wrandiops=130734" > /sys/fs/cgroup/io.cost.model
    ```


## Example 3

Use the `cost.weight` interface to change the default weight of blkcg1 to 50 and then set the weight of blkcg1 on the `254:48` disk to 50. The following commands are used for the cgoup v1 and cgoup v2 interfaces:

-   The command for cgroup v1:

    ```
    echo "50" > /sys/fs/cgroup/blkio/blkcg1/blkio.cost.weight    # Change the default weight to 50.
    echo "254:48 50" > /sys/fs/cgroup/blkio/blkcg1/blkio.cost.weight    # Set the weight of blkcg1 on the disk to 50.
    ```

-   The command for cgroup v2:

    ```
    echo "50" > /sys/fs/cgroup/cg1/io.cost.weight    # Change the default weight to 50.
    echo "254:48 50" > /sys/fs/cgroup/cg1/io.cost.weight    # Set the weight of blkcg1 on the disk to 50.
    ```


## Common monitoring tools

-   iocost monitor script

    The `tools/cgroup/iocost_monitor.py` script in the kernel source code uses the drgn debugger to directly obtain kernel parameters and then provides the I/O performance monitoring data. For more information about drgn, visit [drgn](https://github.com/osandov/drgn). The script is used in the following manner:

    Run the following command to monitor the I/O performance data of the vdd disk:

    ```
    ./iocost_monitor.py vdd
    ```

    A response similar to the following one is returned:

    ```
    vdd RUN  per=500.0ms cur_per=3930.839:v14620.321 busy= +1 vrate=6136.22% params=hdd
                              active    weight      hweight% inflt% dbt  delay usages%
    blkcg1                       *    50/   50   9.09/  9.09   0.00   0  0*000 009:009:009
    blkcg2                       *   500/  500  90.91/ 90.91   0.00   0  0*000 089:091:092
    ```

-   The blkio.cost.statcost.stat interface file of cgroup v1

    The Alibaba Cloud Linux 2 kernel provides the blk-iocost interface file of the cgroup v1 interface. This interface file records the QoS data of each controlled device. Run the following command to view the interface file:

    ```
    cat /sys/fs/cgroup/blkio/blkcg1/blkio.cost.stat
    ```

    A response similar to the following one is returned:

    ```
    254:48 is_active=1 active=50 inuse=50 hweight_active=5957 hweight_inuse=5957 vrate=159571
    ```

-   ftrace monitoring tool

    The Alibaba Cloud Linux 2 kernel provides the ftrace tool related to blk-iocost for kernel-side analytics. The ftrace monitoring tool is used in the following manner:

    1.  Set the `enable` attribute to 1 to enable the ftrace tool.

        ```
        echo 1 > /sys/kernel/debug/tracing/events/iocost/enable
        ```

    2.  View the output information.

        ```
        cat /sys/kernel/debug/tracing/trace_pipe
        ```

        A response similar to the following one is returned:

        ```
            dd-1593  [008] d...   688.565349: iocost_iocg_activate: [vdd:/blkcg1] now=689065289:57986587662878 vrate=137438 period=22->22 vtime=0->57986365150756 weight=50/50 hweight=65536/65536
            dd-1593  [008] d.s.   688.575374: iocost_ioc_vrate_adj: [vdd] vrate=137438->137438 busy=0 missed_ppm=0:0 rq_wait_pct=0 lagging=1 shortages=0 surpluses=1
        <idle>-0     [008] d.s.   688.608369: iocost_ioc_vrate_adj: [vdd] vrate=137438->137438 busy=0 missed_ppm=0:0 rq_wait_pct=0 lagging=1 shortages=0 surpluses=1
            dd-1594  [006] d...   688.620002: iocost_iocg_activate: [vdd:/blkcg2] now=689119946:57994099611644 vrate=137438 period=22->26 vtime=0->57993412421644 weight=250/250 hweight=65536/65536
        <idle>-0     [008] d.s.   688.631367: iocost_ioc_vrate_adj: [vdd] vrate=137438->137438 busy=0 missed_ppm=0:0 rq_wait_pct=0 lagging=1 shortages=0 surpluses=1
        <idle>-0     [008] d.s.   688.642368: iocost_ioc_vrate_adj: [vdd] vrate=137438->137438 busy=0 missed_ppm=0:0 rq_wait_pct=0 lagging=1 shortages=0 surpluses=1
        <idle>-0     [008] d.s.   688.653366: iocost_ioc_vrate_adj: [vdd] vrate=137438->137438 busy=0 missed_ppm=0:0 rq_wait_pct=0 lagging=1 shortages=0 surpluses=1
        <idle>-0     [008] d.s.   688.664366: iocost_ioc_vrate_adj: [vdd] vrate=137438->137438 busy=0 missed_ppm=0:0 rq_wait_pct=0 lagging=1 shortages=0 surpluses=1
        ```


