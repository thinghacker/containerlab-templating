# Containerlab topology to support the exercises in Nokia Border Gateway Protocol for Internet Routing

As part of my exam preparation activities where I wanted to do workbook exercises without needing access to the Nokia BGP labs and looking for an excuse to learn how the containerlab templating features work (and justify some procrastination in actual study), this containerlab topology creates the routers, links and initial configurations to support doing the exercises in the  **Nokia Border Gateway Protocol for Internet Routing Lab Workbook - Revision 4.0 Jan 2022**

![8 Router BGP Topology](bgp_yml_topology.png "Generated Containerlab Topology")


While the audience that is interested in playing around with this particular workbook is probably quite small, hopefully this provides greater value as an example on how containerlab templating might be done.  I had previously done work with jinja2 based templates however containerlab uses the templating capability of *go* which is similar in some aspects or quite different in others.

## Step #0 - Obtain Containerlab (this lab was developed using containerlab version 0.42.0)
https://containerlab.dev/install/

Note: At the time of writing containerlab 0.43.0 had a regression issue supporting template variables on link endpoints however this should be resolved in newer versions

## Step #1 - Ensure that you have a SROS docker image and valid license key

If necessary edit bgp.yml lines 6 (reference to SROS container image) and 7 (pointer to license key) to align with your environment (this lab is using SROS version 23.7.R1).

For instructions on creating a docker image if you have a https://containerlab.dev/manual/vrnetlab/
For a valid license key, reach out to your appropriate Nokia contact.

## Step #2 - Start the Container Lab Instance

This is where the 8 routers and associated interconnections described in *bgp.yml* will be instantiated

```
sudo containerlab deploy --reconfigure --topo bgp.yml
```

The *reconfigure* flag will ignore any previous router configurations that may have been saved (this may or may not be desired - exclude this flag as required)

## Step #3 - Wait until all of the routers are "Healthy"

Using the following command to check the operating state of each of the routers. 
```
watch 'docker ps --format "Name: {{.Names}}, Status: {{.Status}}" | grep clab-bgp'
```

Typically it will take around 3 minutes for the routers to be available - the Display should eventually show the following when things are ready

```
Name: clab-bgp-R7, Status: Up 3 minutes (healthy)
Name: clab-bgp-R6, Status: Up 3 minutes (healthy)
Name: clab-bgp-R2, Status: Up 2 minutes (healthy)
Name: clab-bgp-R4, Status: Up 3 minutes (healthy)
Name: clab-bgp-R8, Status: Up 2 minutes (healthy)
Name: clab-bgp-R5, Status: Up 3 minutes (healthy)
Name: clab-bgp-R1, Status: Up 3 minutes (healthy)
Name: clab-bgp-R3, Status: Up 2 minutes (healthy)
```
Hit control-c to quit the watch

## Step #4 - Jump into Router 1 and check what ports, interfaces and IGP are configured

```
$ ssh admin@clab-bgp-R1
admin@clab-bgp-r1's password:
A:admin@R1# show port

===============================================================================
Ports on Slot 1
===============================================================================
Port          Admin Link Port    Cfg  Oper LAG/ Port Port Port   C/QS/S/XFP/
Id            State      State   MTU  MTU  Bndl Mode Encp Type   MDIMDX
-------------------------------------------------------------------------------
1/1/c1        Down       Down                             conn   100GBASE-LR4*
1/1/c2        Down       Down                             conn   100GBASE-LR4*
1/1/c3        Down       Down                             conn   100GBASE-LR4*
1/1/c4        Down       Down                             conn   100GBASE-LR4*
1/1/c5        Down       Down                             conn   100GBASE-LR4*
1/1/c6        Down       Down                             conn   100GBASE-LR4*
1/1/c7        Down       Down                             conn   100GBASE-LR4*
1/1/c8        Down       Down                             conn   100GBASE-LR4*
1/1/c9        Down       Down                             conn   100GBASE-LR4*
1/1/c10       Down       Down                             conn   100GBASE-LR4*
1/1/c11       Down       Down                             conn   100GBASE-LR4*
1/1/c12       Down       Down                             conn   100GBASE-LR4*

===============================================================================
Ports on Slot A
===============================================================================
Port          Admin Link Port    Cfg  Oper LAG/ Port Port Port   C/QS/S/XFP/
Id            State      State   MTU  MTU  Bndl Mode Encp Type   MDIMDX
-------------------------------------------------------------------------------
A/1           Up    Yes  Up      1514 1514    - netw null faste  MDI
A/3           Down  No   Down    1514 1514    - netw null faste
A/4           Down  No   Down    1514 1514    - netw null faste
===============================================================================

[/]
A:admin@R1# show router interface

===============================================================================
Interface Table (Router: Base)
===============================================================================
Interface-Name                   Adm       Opr(v4/v6)  Mode    Port/SapId
   IP-Address                                                  PfxState
-------------------------------------------------------------------------------
system                           Up        Down/Down   Network system
   -                                                           -
-------------------------------------------------------------------------------
Interfaces : 1
===============================================================================

[/]
A:admin@R1# show router route-table

===============================================================================
Route Table (Router: Base)
===============================================================================
Dest Prefix[Flags]                            Type    Proto     Age        Pref
      Next Hop[Interface Name]                                    Metric
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
No. of Routes: 0
Flags: n = Number of times nexthop is repeated
       B = BGP backup route available
       L = LFA nexthop available
       S = Sticky ECMP requested
===============================================================================

[/]
```
In this case the system is active and the cards/ioms/ports are up but the routers don't have base communications with each other yet

## Step #5 - Use the config template feature to push configs into each of the routers

When invoking the use of a template, containerlab will search for files that match the defined template with the "__**[KIND]**.tmpl" - in this case, ther are Nokia SROS routers (**[KIND]** = "vr-sros") in the topology, so the file used will be "baseconfig__vr-sros.tmpl"  the filepath "." refers to the current directory hosting the files for this particular containerlab

```
sudo containerlab config -t bgp.yml -p . -l baseconfig
```

While it is possible to define a partial configuration for each of the 8 routers in this topology - storing the relevant variables in the clab topology file (bgp.yml) and having a single template (baseconfig__vr-sros.tmpl) that supports per router configuration in a dynamic manner (meaning you can add or remove routers including modification of relevant attributes) in the bgp.yml file provides quite a bit of flexibilty to get containerlab up and running and ready for action quite quickly. 

The template configuration will be used to apply the following on each of the routers:
* Configures Ethernet Ports associated with Links defined in the bgp.yml topology
* Similarly configures IP interfaces associated with Links defined in the bgp.yml topology and binds them to the relevant ports
* Configures the Router system IP interface
* Enable IS-IS on IP interfaces that are expecting it
* Create extra loopback interfaces that the Lab Workbook expects

This templating feature may not be something that you plan to use outside of containerlab based deployments as other tools may be more appropriate, however the built in functionality does have the advantage of just needing containerlab and not other tools such as ansible and the like meaning that it should be more portable and shareable with other users.

## Step #6 - Jump back into Router 1 and check what ports, interfaces and IGP are configured

Repeating step 4, we can see that the configuration has been generated and applied and is functioning

```
A:admin@R1# show port

===============================================================================
Ports on Slot 1
===============================================================================
Port          Admin Link Port    Cfg  Oper LAG/ Port Port Port   C/QS/S/XFP/
Id            State      State   MTU  MTU  Bndl Mode Encp Type   MDIMDX
-------------------------------------------------------------------------------
1/1/c1        Up         Link Up                          conn   100GBASE-LR4*
1/1/c1/1      Up    Yes  Up      9212 9212    - netw null cgige
1/1/c2        Up         Link Up                          conn   100GBASE-LR4*
1/1/c2/1      Up    Yes  Up      9212 9212    - netw null cgige
1/1/c3        Up         Link Up                          conn   100GBASE-LR4*
1/1/c3/1      Up    Yes  Up      9212 9212    - netw null cgige
1/1/c4        Down       Down                             conn   100GBASE-LR4*
1/1/c5        Down       Down                             conn   100GBASE-LR4*
1/1/c6        Down       Down                             conn   100GBASE-LR4*
1/1/c7        Down       Down                             conn   100GBASE-LR4*
1/1/c8        Down       Down                             conn   100GBASE-LR4*
1/1/c9        Down       Down                             conn   100GBASE-LR4*
1/1/c10       Down       Down                             conn   100GBASE-LR4*
1/1/c11       Down       Down                             conn   100GBASE-LR4*
1/1/c12       Down       Down                             conn   100GBASE-LR4*

===============================================================================
Ports on Slot A
===============================================================================
Port          Admin Link Port    Cfg  Oper LAG/ Port Port Port   C/QS/S/XFP/
Id            State      State   MTU  MTU  Bndl Mode Encp Type   MDIMDX
-------------------------------------------------------------------------------
A/1           Up    Yes  Up      1514 1514    - netw null faste  MDI
A/3           Down  No   Down    1514 1514    - netw null faste
A/4           Down  No   Down    1514 1514    - netw null faste
===============================================================================

[/]
A:admin@R1# show router interface

===============================================================================
Interface Table (Router: Base)
===============================================================================
Interface-Name                   Adm       Opr(v4/v6)  Mode    Port/SapId
   IP-Address                                                  PfxState
-------------------------------------------------------------------------------
system                           Up        Up/Down     Network system
   10.16.10.1/32                                               n/a
to_R2                            Up        Up/Down     Network 1/1/c2/1
   10.16.0.9/30                                                n/a
to_R3                            Up        Up/Down     Network 1/1/c3/1
   10.2.0.1/24                                                 n/a
to_R5                            Up        Up/Down     Network 1/1/c1/1
   10.16.0.2/30                                                n/a
-------------------------------------------------------------------------------
Interfaces : 4
===============================================================================

[/]
A:admin@R1# show router route-table

===============================================================================
Route Table (Router: Base)
===============================================================================
Dest Prefix[Flags]                            Type    Proto     Age        Pref
      Next Hop[Interface Name]                                    Metric
-------------------------------------------------------------------------------
10.2.0.0/24                                   Local   Local     00h01m14s  0
       to_R3                                                        0
10.16.0.0/30                                  Local   Local     00h01m14s  0
       to_R5                                                        0
10.16.0.4/30                                  Remote  ISIS      00h00m59s  18
       10.16.0.10                                                   20
10.16.0.8/30                                  Local   Local     00h01m14s  0
       to_R2                                                        0
10.16.0.20/30                                 Remote  ISIS      00h00m59s  18
       10.16.0.1                                                    20
10.16.10.1/32                                 Local   Local     00h01m14s  0
       system                                                       0
10.16.10.2/32                                 Remote  ISIS      00h00m59s  18
       10.16.0.10                                                   10
10.16.10.5/32                                 Remote  ISIS      00h00m59s  18
       10.16.0.1                                                    10
10.16.10.6/32                                 Remote  ISIS      00h00m59s  18
       10.16.0.1                                                    20
10.16.10.6/32                                 Remote  ISIS      00h00m59s  18
       10.16.0.10                                                   20
-------------------------------------------------------------------------------
No. of Routes: 10
Flags: n = Number of times nexthop is repeated
       B = BGP backup route available
       L = LFA nexthop available
       S = Sticky ECMP requested
===============================================================================
```

Adding/removing routers or links to the bgp.yml file doesn't mean that hte template needs to be adjusted.  The variable information of interest that is defined in the topology is what is used
Not all routers need to participate in IS-IS (not defining isis_area within a nodes config vars will mean the template doesnt attempt to push an IS-IS config onto that device)
Even if the router is running IS-IS, not all interfaces need to participate in it (within the link endpoint definitions, not setting isis to true means the template will exclude that interfaces from being attached to IS-IS.

In the case of R1, the interface to R3 is intended to be to a different autonmous system for this workbook and the IGP doesn't extend between these routers

```
A:admin@R1# show router interface

===============================================================================
Interface Table (Router: Base)
===============================================================================
Interface-Name                   Adm       Opr(v4/v6)  Mode    Port/SapId
   IP-Address                                                  PfxState
-------------------------------------------------------------------------------
system                           Up        Up/Down     Network system
   10.16.10.1/32                                               n/a
to_R2                            Up        Up/Down     Network 1/1/c2/1
   10.16.0.9/30                                                n/a
to_R3                            Up        Up/Down     Network 1/1/c3/1
   10.2.0.1/24                                                 n/a
to_R5                            Up        Up/Down     Network 1/1/c1/1
   10.16.0.2/30                                                n/a
-------------------------------------------------------------------------------
Interfaces : 4
===============================================================================

[/]
A:admin@R1# show router isis interface

===============================================================================
Rtr Base ISIS Instance 0 Interfaces
===============================================================================
Interface                        Level CircID  Oper      L1/L2 Metric     Type
                                               State
-------------------------------------------------------------------------------
system                           L2    1       Up        -/0              p2p
to_R2                            L2    2       Up        -/10             p2p
to_R5                            L2    4       Up        -/10             p2p
-------------------------------------------------------------------------------
Interfaces : 3
===============================================================================

[/]
```

## Step #7 - Start performing Lab Exercises

Now that the lab has been instantiated and the base configurations are in place, it is just a matter of connecting to the router of interest and performing the relevant lab tasks.

For example, if we want to work on R2, we can ssh into it and start doing things

```
ssh admin@clab-bgp-R2
```

