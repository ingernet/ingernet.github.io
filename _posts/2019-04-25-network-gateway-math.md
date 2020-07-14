---
layout: page
title: "Calculating IP Ranges with Bash"
date:   2019-04-25 20:00:00 -0700
categories: bash cloudfoundry jq pcf terraform 
---


Given a gateway address and the number of IPs to include in the range, 
this is a simple function that generates a "reserved range" value.
I wrote this because reserved ranges weren't a Terraform output while I was setting up subnets in Cloud Foundry/AWS, and I was 
bored of doing the math on my fingers.


This function, `make_ranges`, is V SIMPLE and pretty bespoke! It's not good for larger blocks where reserved ranges
transcend a .255 range. But I don't need that yet. When I do, I'll write it. 

This is also customized for the blocks that PCF was looking for, which don't resemble [what people usually think of as CIDR blocks](https://www.akadia.com/services/ip_routing_on_subnets.html) - for example, the first range starts with a `.0` address, which isn't a real thing in networking. There's a good chance that you'll need to adapt this to what you're actually working on. 
But it shows you the logic/process behind calculating subnets automatically.

Anyway. Enough preamble.

```
#!/bin/env bash

function make_ranges () {
    # $1 = start address
    # $2 = how many ips to include in a block

    # take the supplied router ($1) and split the last octet into its own thing, 
    #     which is the meat of the gateway address. 
    # retain first three octets as base of ip group
    OCTETS=(${1//\./ })
    GATEWAY_OCTET_MAIN="${OCTETS[0]}.${OCTETS[1]}.${OCTETS[2]}"
    GATEWAY_OCTET_FOURTH=${OCTETS[3]}

    # bump last octest down by one digit to get subnet start
    GATEWAY_RES_START=$(($GATEWAY_OCTET_FOURTH - 1))

    # take the supplied reserved range interval ($2) and use it to generate a range string
    GATEWAY_RES_END=$((GATEWAY_RES_START + $2))
    echo "${GATEWAY_OCTET_MAIN}.${GATEWAY_RES_START}-${GATEWAY_OCTET_MAIN}.${GATEWAY_RES_END}"
}
```

Example usage, with a `tfstate` file as the datasource:
```
CP_GATEWAY0=$(terraform output -json -state terraform-state/terraform-cp.tfstate control_plane_subnet_gateways  | jq -r '.value[0]')
# ^ $CP_GATEWAY0 resolves to: 10.30.4.1

CP_GATEWAY1=$(terraform output -json -state terraform-state/terraform-cp.tfstate control_plane_subnet_gateways  | jq -r '.value[1]')
# ^ $CP_GATEWAY1 resolves to: 10.30.4.65

CP_GATEWAY2=$(terraform output -json -state terraform-state/terraform-cp.tfstate control_plane_subnet_gateways  | jq -r '.value[2]')
# ^ $CP_GATEWAY2 resolves to: 10.30.4.129

cp_reserved_range0: $(make_ranges ${CP_GATEWAY0} 10)
cp_reserved_range1: $(make_ranges ${CP_GATEWAY1} 10)
cp_reserved_range2: $(make_ranges ${CP_GATEWAY2} 10)
```

How those last three lines end up showing up:
```
# cp_reserved_range0: 10.30.4.0-10.30.4.10
# cp_reserved_range1: 10.30.4.64-10.30.4.74
# cp_reserved_range2: 10.30.4.128-10.30.4.138
```