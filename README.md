# Container-LSP

## Executive Summary
In previous two articles I have discussed in detail how to handle increasing traffic demand in MPLS backbone by using Faster Auto bandwidth adjustment (https://github.com/kashif-nawaz/MPLS-Auto-BW-Junos) and by deploying Class of Service CoS (https://github.com/kashif-nawaz/MPLS-Backbone-Class-Of-Service-Design-Principles) but those arrangements are sufficient to meet demand in traffic increase. In addition to faster auto bandwidth adjustment and CoS parallel LSPs are required between 2 PEs (ideally on distinct paths) so that traffic could load balanced over equal cost multipath links.  This combination will provide an ideal recipe to handle increasing traffic needs but it would add another challenge i.e always provisioned parallel LSP will add overhead in RSVP control plane by maintaining the RSVP reservation.  So is possible to optimize this solution in a way that par rale LSP should be created dynamically as per traffic needs, answer is yes , Traffic Engineering (TE++) or Container LSP solve this problem. It enables an operator to configure LSP template to be used to dynamically create parallel LSPs based on increasing traffic demand and once demands is reduced parallel LSP would be torn down. This arrangement will reduce control plane overhead caused by pre provisioned parallel LSPs. 

Juniper Networks has published an excellent white paper (https://www.juniper.net/assets/es/es/local/pdf/whitepapers/2000587-en.pdf) to define all the classical problems related to traffic load sharing in MPLS backbone network and how those problems would be addressed using Container LSP. But that paper does not talk about operative part and in this wiki I will endeavor to describe operative part of Container LSP.

## Important Terms and Definitions 
I have followed Juniper Networks official documentation (https://www.juniper.net/documentation/us/en/software/junos/mpls/topics/topic-map/container-lsp-configuration.html) to digest important terms and definitions. Container LSP has nominal LSP which is always present and supplementary LSPs which are created to handle increasing traffic demands and when traffic demands decrease supplementary LSPs are automatically pruned thus freeing up resources reservation which ultimately reduces control plane overhead. Process for adding up supplementary LSPs is called splitting and removing supplementary LSP is called merging. Both, splitting and merging happens at regular interval (which is called normalization interval) and decision to go  for splitting or merger is controlled by various config parameters  by considering Current-Aggr-Bw (which is sum of current reservation by all supplementary or member LSPs) and New-Aggr-Bw (which is sum of traffic rate on supplementary or member LSPs). 

## Lab Topology
![Topology](./images/topology.png)


## Config Components
### LSP Template 
A template is required to be configured with keyword "template" under protocol mpls hierarchy and that template will be inherited by nominal LSP and all supplementary LSPs. To get understanding of various parameters defined in following template, please read my article on faster autobandwitdh adjustment available on (https://github.com/kashif-nawaz/MPLS-Auto-BW-Junos). 

```
label-switched-path LSP_TEMPLATE {
    template;
    priority 5 5;
    optimize-timer 60;
    least-fill;
    node-link-protection;
    in-place-lsp-bandwidth-update;
    auto-bandwidth {
        adjust-interval 300;
        adjust-threshold 3;
        adjust-threshold-activate-bandwidth 25m;
        minimum-bandwidth 0;
        maximum-bandwidth 10g;
        adjust-threshold-overflow-limit 3;
        adjust-threshold-underflow-limit 3;
    }
}
```
### Contrainer LSP
Instead of configuring label-switch-path, container-label-switched-path config hirearchy is used under mpls hierarchy, e.g container LSP defination from PE1 to other PEs (lab topology is depicted  above) is appended below:-

```
protocols {
    mpls {
apply-groups CT_LSP;
icmp-tunneling;
ipv6-tunneling;
apply-groups CT_LSP;
container-label-switched-path PE1-to-PE2 {
    to 172.172.172.2;
}
container-label-switched-path PE1-to-PE3 {
    to 172.172.172.9;
}
container-label-switched-path PE1-to-PE4 {
    to 172.172.172.10;
}
}
}
```
I have created a Junos group CT_LSP to define container LPS paramters and applied it to protocols mpls. 

```
groups {
CT_LSP 
protocols {
    mpls {
        statistics {
            file auto-bw;
            interval 50;
            auto-bandwidth;
        }
        container-label-switched-path <PE*-to*> {
            label-switched-path-template {
                LSP_TEMPLATE;
            }
            splitting-merging {
                maximum-member-lsps 3;
                minimum-member-lsps 1;
                splitting-bandwidth 150m;
                merging-bandwidth 10m;
                maximum-signaling-bandwidth 10m;
                minimum-signaling-bandwidth 10m;
                normalization {
                    normalize-interval 400;
                    failover-normalization;
                }
                sampling {
                    cut-off-threshold 1;
                    use-percentile 90;
                }
            }
        }
    }
}
}
```

Container LSP is cofigured with template that defines 
