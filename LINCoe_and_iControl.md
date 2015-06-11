Start LINC-OE simple optical topology with iControl. 
Thhis document has two portions: 
1. setting up the iControl 
2. setting up the linc-oe




# 1. setting up/running the iControl #
If you haven't clone the LOOM repo, do the following:
```shell
	git clone https://github.com/FlowForwarding/loom.git
```
if you already have cloned the LOOM repo. Build and Run the icontrol: 

Build icontrol:
```shell
	cd icontrol
	make
```
Run:
```shell
	rel/icontrol/bin/icontrol console
```
This document shows how to use LINC-Switch to simulate simple optical
network like on the diagram below:

# 2. setting up the linc-oe #
Here is the topology: 
```
           |       (1)       |      |      (2)       |      |       (3)       |
[tap0]-----|[1]  <PO-SW>  [2]|~~~~~~|[1]  <O-SW>  [2]|~~~~~~|[1]  <PO-SW>  [2]|-----[tap1]
 (OS)      |     (LINC)      |      |     (LINC)     |      |      (LINC)     |      (OS)

```

> * (X) - is the logical switch number
> * PO-SW - indicates packet-optical logical switch (i.e. a switch that
> has Ethernet ports as well as optical ones)
> * O-SW - indicates optical switch (i.e. a switch that has only internally
> emulated optical ports and links between them)
> * [tapX] - is the operating system tap interface number X
> * ---- line is the packet link
> * ~~~~ line is the optical link

## Operating system configuration ##
To run the simulation two tap interfaces are required. Following commands
show how to configure them in Linux:
```shell
sudo tunctl -t tap0
sudo tunctl -t tap1
sudo ip link set dev tap0 up
sudo ip link set dev tap1 up
```

## LINC-OE configuration ##

`rel/files/sys.config` file for the network shown above should looks
as following:
```erlang
[{linc,
  [{of_config,disabled},
   {capable_switch_ports,
    [{port,1,[{interface,"tap0"}]},
     {port,2,[{interface,"dummy"}, {type, optical}]},
     {port,3,[{interface,"dummy"}, {type, optical}]},
     {port,4,[{interface,"dummy"}, {type, optical}]},
     {port,5,[{interface,"dummy"}, {type, optical}]},
     {port,6,[{interface,"tap1"}]}
    ]},
   {capable_switch_queues, []},
   {optical_links, [{{1,2}, {2,1}}, {{2,2},{3,1}}]},
   {logical_switches,
    [{switch,1,
      [{backend,linc_us4_oe},
       {controllers,[{"Switch0-Controller","localhost",6653,tcp}]},
       {controllers_listener,disabled},
       {queues_status,disabled},
       {datapath_id, "00:00:00:00:00:01:00:01"},
       {ports,[{port,1,[{queues,[]}, {port_no, 1}]},
               {port,2,[{queues,[]}, {port_no, 2}]}
              ]}]},
     {switch,2,
      [{backend,linc_us4_oe},
       {controllers,[{"Switch0-Controller","localhost",6653,tcp}]},
       {controllers_listener,disabled},
       {queues_status,disabled},
       {datapath_id, "00:00:00:00:00:01:00:02"},
       {ports,[{port,3,[{queues,[]}, {port_no, 1}]},
               {port,4,[{queues,[]}, {port_no, 2}]}
              ]}]},
     {switch,3,
      [{backend,linc_us4_oe},
       {controllers,[{"Switch0-Controller","localhost",6653,tcp}]},
       {controllers_listener,disabled},
       {queues_status,disabled},
       {datapath_id, "00:00:00:00:00:01:00:03"},
       {ports,[{port,5,[{queues,[]}, {port_no, 1}]},
               {port,6,[{queues,[]}, {port_no, 2}]}
              ]}]}
    ]}]},
 {of_protocol, [{no_multipart, false}]},
 {enetconf,
  [{capabilities,[{base,{1,1}},{startup,{1,0}},{'writable-running',{1,0}}]},
   {callback_module,linc_ofconfig},
   {sshd_ip,any},
   {sshd_port,1830},
   {sshd_user_passwords,[{"linc","linc"}]}]},
 {epcap,
  [{verbose, false},
   {stats_interval, 10},
   {buffer_size, 73400320}]},
 {lager,
  [{handlers,
    [{lager_console_backend,debug},
     {lager_file_backend,
      [{"log/error.log",error,10485760,"$D0",5},
       {"log/debug.log",debug,10485760,"$D0",5},
       {"log/console.log",info,10485760,"$D0",5}]}]}]},
 {sasl,
  [{sasl_error_logger,{file,"log/sasl-error.log"}},
   {errlog_type,error},
   {error_logger_mf_dir,"log/sasl"},
   {error_logger_mf_maxbytes,1048576000000},
   {error_logger_mf_maxfiles,5}]},
 {sync,
  [{excluded_modules, [procket]}]}].
```

## Starting LINC-OE ##
  Build LINC-Switch and run it:
  ```shell
  $ make rel && sudo rel/linc/bin/linc console
```
# Usage #
So now you have both the linc-oe and iControll set up. Time to play with them.

On one end you have a switch connected to Tap 0 and on the other end you have switch conneted to tap. Switches connect the two tap interfaces. But, If you put packets on one tap and monitoor the other tap, you will not recieve any packets. Why? Because iController needs to insert/instal flows on switches so that a path is created from tap 0 to tap 1. 
Let's create that path: 

```
(icontrol@127.0.0.1)> iof:oe_flow_tw(2,100,1,2,20).
```
Install a flow on switch 1 that will forward packets from port 1 to the optical link connected to port 2 using wavelength 20. Remeber that port 1 of switch with key 2 is connected to tap0. 
Note that switch with key=2 has a switch id=1. Switch id is the one that you set in the `sys.config` file. -> I don't know why they are different.  
Here switch with key 2 and switch id of 1 has one connection to a tap interface and one connection to optical link.



```
(icontrol@127.0.0.1)> iof:oe_flow_ww(1,100,1,20,2,20).
```
For switch with key 1, this will Install a flow on switch 2 that will take optical data from channel 20 on port 1 and put it on port 2 into the same channel. 
Here switch with key 1 and switch id of 2 is the middle switch. So there is no tap interface connected to it and has two optical connection. That's why we use `oe_flow_ww`. 



```
(icontrol@127.0.0.1)> iof:oe_flow_wt(3,100,1,20,2).
```
For switch with key 2, this will Install a flow on switch 3 that will take optical data from channel 20 on port 1 and convert it back to packet data and send it through port 2 


## Useful Funtions: ##
### Clearing flows in specific switch ###
Clears flows in switch with key equal to <Switch key>.
 ```
 iof:clear_flows(<Switch key>). 
 ```

### Flows ###
Dumps flows in switch with key equal to <Switch key>.
```
iof:flows(<Switch key>).  
```

# Additional Sources #
This tutorial was made from a comnination of the following tutorials: 
https://github.com/FlowForwarding/LINC-Switch/blob/master/docs/optical_extension.md 
https://github.com/FlowForwarding/loom/tree/master/icontrol 


