---
title: 'Targeted network monitoring using only fping and rrdtool.'
date: '2021-02-21T14:01:23+00:00'
author: deaves
layout: post
permalink: /2021/02/targeted-network-monitoring/
categories:
    - Linux
    - 'Linux Admin'
    - 'Linux Scripts'
    - Networking
tags:
    - bandwidth
    - DevOps
    - fping
    - monitoring
    - network
    - rrdtool
    - smokeping
---

I’ve been very unhappy with the state of network monitoring applications lately. Most network monitoring tools are either too big or too arbitrary to be helpful for application support. This can be an issue when focusing on a specific application with components separated into various tiers, datacenters or locations. When network performance is in question the most helpful data is the active latency between a node and its other components (during or leading up to the time in question). If a network monitoring tool lacks any specificity of the application in question it will be viewed as too dense or cerebral to be useful; at worse it will harm troubleshooting. The quicker hard data can be accessed proving out the network layer, the quicker troubleshooting can move up the stack towards resolution.

Monolithic tools like [Cacti](https://www.davideaves.com/2016/01/cisco-ap-dot11-associations-in-cacti/) are sometimes useful, however the lighter the script is, the more nimbly it can be deployed on a wide variety of nodes. Because both [FPing](https://fping.org/) and [RRDTool](https://oss.oetiker.ch/rrdtool/) are small, useful and standard Linux packages they are ideal, so l wrote the following bash script that leverages only those 2 tools together. The data collected is roughly identical to [SmokePing](https://oss.oetiker.ch/smokeping/) but has the benefit of not dirtying a system with unnecessary packages. The script can easily be deployed by any devops deployment and is ran via crontab. Graph data can be created when or if they are needed.

#### **fping\_rrd.sh**: *Bourne-Again shell script, ASCII text executable*

```bash
#!/bin/env -S bash
## FPing data collector for RRDTOOL
#
# Crontab:
#   */5 * * * *     ${HOME}/fping_rrd.sh
#

# Enable for debuging
# set -x

STEP=300 # 5min
PINGS=20 # 20 pings

# The first ping is usually an outlier so I add an extra ping.
fping_hosts="172.31.4.1 172.31.4.4"
fping_opts="-C $((PINGS+1)) -q -B1 -r1 -i10"
rrd_path="${HOME}/public_html"
rrd_timestamp=$(date +%s)

# Verify script requirements.
for req in fping rrdtool; do
    type ${req} >/dev/null 2>&1 || {
        echo >&2 "$(basename "${0}"): I require \"${req}\" but it's not installed. Aborting."
        exit 1
    }
done

funcion calc_median ()
{
    awk '{ if ( $1 != "-" ) { fping[NR] = $1 }
           else { NR-- }
         }
     END { asort(fping);
           if (NR % 2) { print fping[(NR + 1) / 2] }
           else { print (fping[(NR / 2)] + fping[(NR / 2) + 1]) / 2.0 }
         }'
}

function rrd_create ()
{
    rrdtool create "${fping_rrd}" \
        --start now-2h --step $((STEP)) \
        DS:loss:GAUGE:$((STEP*2)):0:$((PINGS)) \
        DS:median:GAUGE:$((STEP*2)):0:180 \
        $(seq -f " DS:ping%g:GAUGE:$((STEP*2)):0:180" 1 $((PINGS++))) \
        RRA:AVERAGE:0.5:1:1008 \
        RRA:AVERAGE:0.5:12:4320 \
        RRA:MIN:0.5:12:4320 \
        RRA:MAX:0.5:12:4320 \
        RRA:AVERAGE:0.5:144:720 \
        RRA:MAX:0.5:144:720 \
        RRA:MIN:0.5:144:720
}

function rrd_update ()
{
    rrd_loss=0
    rrd_median=""
    rrd_rev=$((PINGS))
    rrd_name=""
    rrd_value="${rrd_timestamp}"

    for rrd_idx in $(seq 1 $((rrd_rev))); do
        rrd_name="${rrd_name}$([[ ${rrd_idx} -gt "1" ]] && echo ":")ping$((rrd_idx))"
        rrd_value="${rrd_value}:${fping_array[-$((rrd_rev))]}"
        rrd_median="${fping_array[-$((rrd_rev))]}\n${rrd_median}"

        [ "${fping_array[-$((rrd_rev))]}" == "-" ] && (( rrd_loss++ ))

        (( rrd_rev-- ))
    done
    rrd_median=$(printf ${rrd_median} | calc_median)

    rrdtool update "${fping_rrd}" --template $(echo ${rrd_name}:median:loss ${rrd_value}:${rrd_median}:${rrd_loss} | sed 's/-/U/g')
    unset rrd_loss rrd_median rrd_rev rrd_name rrd_value
}

fping ${fping_opts} ${fping_hosts} 2>&1 | while read fping_line; do
    fping_array=( ${fping_line} )
    fping_rrd="${rrd_path}/fping_${fping_array[0],,}.rrd"

    # Create RRD file.
    if [ ! -f "${fping_rrd}" ]; then
        rrd_create
    fi

    # Update RRD file.
    if [ -f "${fping_rrd}" ]; then
        rrd_last=$(( ${rrd_timestamp} - $(rrdtool last "${fping_rrd}") ))
        [[ $((rrd_last)) -ge $((STEP)) ]] && rrd_update
    fi && unset rrd_last
done
```

## Creating Network Monitoring Graphs

The following are 3 example scripts that use rrdtool to create graphs from the RRD files.

### Mini Graph

![fping_mini.png](/assets/fping_mini.png)

#### **graph\_mini.sh**: *Bourne-Again shell script, ASCII text executable*

```bash
#!/bin/env -S bash
## Create a  mini graph from a RRD file

# Enable for debuging
#set -x

fping_rrd="${1}"
COLOR=( "FF5500" )

# Verify script requirements.
for req in fping; do
    type ${req} >/dev/null 2>&1 || {
        echo >&2 "$(basename "${0}"): I require \"${req}\" but it's not installed. Aborting."
        exit 1
    }
done

function rrd_graph_cmd ()
{
cat << EOF
rrdtool graph "$(basename ${fping_rrd%.*})_mini.png"
--start "${START}" --end "${END}"
--title "$(date -d "${START}") ($(awk -v TIME=$TIME 'BEGIN {printf "%.1f hr", TIME/3600}'))"
--height 65 --width 600
--vertical-label "Seconds"
--color BACK#F3F3F3
--color CANVAS#FDFDFD
--color SHADEA#CBCBCB
--color SHADEB#999999
--color FONT#000000
--color AXIS#2C4D43
--color ARROW#2C4D43
--color FRAME#2C4D43
--border 1
--font TITLE:10:"Arial"
--font AXIS:8:"Arial"
--font LEGEND:8:"Courier"
--font UNIT:8:"Arial"
--font WATERMARK:6:"Arial"
--imgformat PNG
EOF
}

function rrd_graph_opts ()
{
rrd_idx=0
cat << EOF
DEF:median$((rrd_idx))="${fping_rrd}":median:AVERAGE
DEF:loss$((rrd_idx))="${fping_rrd}":loss:AVERAGE
$(for ((i=1;i<=PINGS;i++)); do echo "DEF:ping$((rrd_idx))p$((i))=\"${fping_rrd}\":ping$((i)):AVERAGE"; done)
CDEF:ploss$((rrd_idx))=loss$((rrd_idx)),20,/,100,*
CDEF:dm$((rrd_idx))=median$((rrd_idx)),0,100000,LIMIT
$(for ((i=1;i<=PINGS;i++)); do echo "CDEF:p$((rrd_idx))p$((i))=ping$((rrd_idx))p$((i)),UN,0,ping$((rrd_idx))p$((i)),IF"; done)
$(echo -n "CDEF:pings$((rrd_idx))=$((PINGS)),p$((rrd_idx))p1,UN"; for ((i=2;i<=PINGS;i++)); do echo -n ",p$((rrd_idx))p$((i)),UN,+"; done; echo ",-")
$(echo -n "CDEF:m$((rrd_idx))=p$((rrd_idx))p1"; for ((i=2;i<=PINGS;i++)); do echo -n ",p$((rrd_idx))p$((i)),+"; done; echo ",pings$((rrd_idx)),/")
$(echo -n "CDEF:sdev$((rrd_idx))=p$((rrd_idx))p1,m$((rrd_idx)),-,DUP,*"; for ((i=2;i<=PINGS;i++)); do echo -n ",p$((rrd_idx))p$((i)),m$((rrd_idx)),-,DUP,*,+"; done; echo ",pings$((rrd_idx)),/,SQRT")
CDEF:dmlow$((rrd_idx))=dm$((rrd_idx)),sdev$((rrd_idx)),2,/,-
CDEF:s2d$((rrd_idx))=sdev$((rrd_idx))
AREA:dmlow$((rrd_idx))
AREA:s2d$((rrd_idx))#${COLOR}30:STACK
LINE1:dm$((rrd_idx))#${COLOR}:"$(basename ${fping_rrd%.*} | awk -F'_' '{print $NF}')\t"
VDEF:avmed$((rrd_idx))=median$((rrd_idx)),AVERAGE
VDEF:avsd$((rrd_idx))=sdev$((rrd_idx)),AVERAGE
CDEF:msr$((rrd_idx))=median$((rrd_idx)),POP,avmed$((rrd_idx)),avsd$((rrd_idx)),/
VDEF:avmsr$((rrd_idx))=msr$((rrd_idx)),AVERAGE
GPRINT:avmed$((rrd_idx)):"Median RTT\: %5.2lfms"
GPRINT:ploss$((rrd_idx)):AVERAGE:"Loss\: %5.1lf%%"
GPRINT:avsd$((rrd_idx)):"Std Dev\: %5.2lfms"
GPRINT:avmsr$((rrd_idx)):"Ratio\: %5.1lfms\\j"
COMMENT:"Probe\: $((PINGS)) pings every $((STEP)) seconds"
COMMENT:"${fping_rrd}\\j"
EOF
}

if [ ! -r "${fping_rrd}" ]; then
    printf "${0} \"file.rrd\"\n"
else
    STEP=$(rrdtool info "${fping_rrd}" | awk '/^step/{print $NF}')
    PINGS=$(rrdtool info "${fping_rrd}" | awk '/^ds.ping.*index/{count++} END{print count}')

    START="$([ -z "${2}" ] && echo "-9 hours" || echo "${2}")"
    END="$([ -z "${3}" ] && echo "now" || echo "${3}")"
    TIME=$(( $(date -d "${END}" +%s) - $(date -d "${START}" +%s) ))

    eval $(rrd_graph_cmd; rrd_graph_opts)
fi
```

### Combined (multi) Graph

![fping_graph.png](/assets/fping_graph.png)

#### **graph\_multi.sh**: *Bourne-Again shell script, ASCII text executable*

```bash
#!/bin/env -S bash
## Create a mini graph from multiple RRDs

# Enable for debuging
#set -x

START="-9 hours"
END="now"

png_file="${1}"
rrd_files="${*:2}"

# Verify script requirements.
for req in fping; do
    type ${req} >/dev/null 2>&1 || {
        echo >&2 "$(basename "${0}"): I require \"${req}\" but it's not installed. Aborting."
        exit 1
    }
done

function rrd_graph_cmd ()
{
cat << EOF
rrdtool graph "${png_file}"
--start "${START}" --end "${END}"
--title "$(date -d "${START}") ($(awk -v TIME=$TIME 'BEGIN {printf "%.1f hr", TIME/3600}'))"
--height 115 --width 600
--vertical-label "Seconds"
--color BACK#F3F3F3
--color CANVAS#FDFDFD
--color SHADEA#CBCBCB
--color SHADEB#999999
--color FONT#000000
--color AXIS#2C4D43
--color ARROW#2C4D43
--color FRAME#2C4D43
--border 1
--font TITLE:10:"Arial"
--font AXIS:8:"Arial"
--font LEGEND:8:"Courier"
--font UNIT:8:"Arial"
--font WATERMARK:6:"Arial"
--imgformat PNG
EOF
}

function rrd_graph_opts ()
{
rrd_idx=0
for fping_rrd in ${rrd_files}
do COLOR=$(openssl rand -hex 3)
STEP=$(rrdtool info "${fping_rrd}" | awk '/^step/{print $NF}')
PINGS=$(rrdtool info "${fping_rrd}" | awk '/^ds.ping.*index/{count++} END{print count}')
cat << EOF
DEF:median$((rrd_idx))="${fping_rrd}":median:AVERAGE
DEF:loss$((rrd_idx))="${fping_rrd}":loss:AVERAGE
$(for ((i=1;i<=PINGS;i++)); do echo "DEF:ping$((rrd_idx))p$((i))=\"${fping_rrd}\":ping$((i)):AVERAGE"; done)
CDEF:ploss$((rrd_idx))=loss$((rrd_idx)),20,/,100,*
CDEF:dm$((rrd_idx))=median$((rrd_idx)),0,100000,LIMIT
$(for ((i=1;i<=PINGS;i++)); do echo "CDEF:p$((rrd_idx))p$((i))=ping$((rrd_idx))p$((i)),UN,0,ping$((rrd_idx))p$((i)),IF"; done)
$(echo -n "CDEF:pings$((rrd_idx))=$((PINGS)),p$((rrd_idx))p1,UN"; for ((i=2;i<=PINGS;i++)); do echo -n ",p$((rrd_idx))p$((i)),UN,+"; done; echo ",-")
$(echo -n "CDEF:m$((rrd_idx))=p$((rrd_idx))p1"; for ((i=2;i<=PINGS;i++)); do echo -n ",p$((rrd_idx))p$((i)),+"; done; echo ",pings$((rrd_idx)),/")
$(echo -n "CDEF:sdev$((rrd_idx))=p$((rrd_idx))p1,m$((rrd_idx)),-,DUP,*"; for ((i=2;i<=PINGS;i++)); do echo -n ",p$((rrd_idx))p$((i)),m$((rrd_idx)),-,DUP,*,+"; done; echo ",pings$((rrd_idx)),/,SQRT")
CDEF:dmlow$((rrd_idx))=dm$((rrd_idx)),sdev$((rrd_idx)),2,/,-
CDEF:s2d$((rrd_idx))=sdev$((rrd_idx))
AREA:dmlow$((rrd_idx))
AREA:s2d$((rrd_idx))#${COLOR}30:STACK
LINE1:dm$((rrd_idx))#${COLOR}:"$(basename ${fping_rrd%.*} | awk -F'_' '{print $NF}')\t"
VDEF:avmed$((rrd_idx))=median$((rrd_idx)),AVERAGE
VDEF:avsd$((rrd_idx))=sdev$((rrd_idx)),AVERAGE
CDEF:msr$((rrd_idx))=median$((rrd_idx)),POP,avmed$((rrd_idx)),avsd$((rrd_idx)),/
VDEF:avmsr$((rrd_idx))=msr$((rrd_idx)),AVERAGE
GPRINT:avmed$((rrd_idx)):"Median RTT\: %5.2lfms"
GPRINT:ploss$((rrd_idx)):AVERAGE:"Loss\: %5.1lf%%"
GPRINT:avsd$((rrd_idx)):"Std Dev\: %5.2lfms"
GPRINT:avmsr$((rrd_idx)):"Ratio\: %5.1lfms\\j"
EOF
(( rrd_idx++ ))
done && unset rrd_idx
}

if [ -z "${rrd_files}" ]; then
    printf "${0} \"file.png\" { file1.rrd ... file6.rrd }\n"
else
    TIME=$(( $(date -d "${END}" +%s) - $(date -d "${START}" +%s) ))
    eval $(rrd_graph_cmd; rrd_graph_opts)
fi
```

### SmokePing like Graph

![fping_smoke.png](/assets/fping_smoke.png)

#### **graph\_smoke.sh**: *Bourne-Again shell script, ASCII text executable*

```bash
#!/bin/env -S bash
## Create a SmokePing like graph from a RRD file

# Enable for debuging
#set -x

fping_rrd="${1}"
COLOR=( "0F0f00" "00FF00" "00BBFF" "0022FF" "8A2BE2" "FA0BE2" "C71585" "FF0000" )
LINE=".5"

# Verify script requirements.
for req in fping; do
    type ${req} >/dev/null 2>&1 || {
        echo >&2 "$(basename "${0}"): I require \"${req}\" but it's not installed. Aborting."
        exit 1
    }
done

function rrd_graph_cmd ()
{
cat << EOF
rrdtool graph "$(basename ${fping_rrd%.*})_smoke.png"
--start "${START}" --end "${END}"
--title "$(basename ${fping_rrd%.*} | awk -F'_' '{print $NF}')"
--height 95 --width 600
--vertical-label "Seconds"
--color BACK#F3F3F3
--color CANVAS#FDFDFD
--color SHADEA#CBCBCB
--color SHADEB#999999
--color FONT#000000
--color AXIS#2C4D43
--color ARROW#2C4D43
--color FRAME#2C4D43
--border 1
--font TITLE:10:"Arial"
--font AXIS:8:"Arial"
--font LEGEND:9:"Courier"
--font UNIT:8:"Arial"
--font WATERMARK:7:"Arial"
--imgformat PNG
EOF
}

function rrd_graph_opts ()
{
cat << EOF
DEF:median$((rrd_idx))="${fping_rrd}":median:AVERAGE
DEF:loss$((rrd_idx))="${fping_rrd}":loss:AVERAGE
$(for ((i=1;i<=PINGS;i++)); do echo "DEF:ping$((rrd_idx))p$((i))=\"${fping_rrd}\":ping$((i)):AVERAGE"; done)
CDEF:ploss$((rrd_idx))=loss$((rrd_idx)),20,/,100,*
CDEF:dm$((rrd_idx))=median$((rrd_idx)),0,100000,LIMIT
$(for ((i=1;i<=PINGS;i++)); do echo "CDEF:p$((rrd_idx))p$((i))=ping$((rrd_idx))p$((i)),UN,0,ping$((rrd_idx))p$((i)),IF"; done)
$(echo -n "CDEF:pings$((rrd_idx))=$((PINGS)),p$((rrd_idx))p1,UN"; for ((i=2;i<=PINGS;i++)); do echo -n ",p$((rrd_idx))p$((i)),UN,+"; done; echo ",-")
$(echo -n "CDEF:m$((rrd_idx))=p$((rrd_idx))p1"; for ((i=2;i<=PINGS;i++)); do echo -n ",p$((rrd_idx))p$((i)),+"; done; echo ",pings$((rrd_idx)),/")
$(echo -n "CDEF:sdev$((rrd_idx))=p$((rrd_idx))p1,m$((rrd_idx)),-,DUP,*"; for ((i=2;i<=PINGS;i++)); do echo -n ",p$((rrd_idx))p$((i)),m$((rrd_idx)),-,DUP,*,+"; done; echo ",pings$((rrd_idx)),/,SQRT")
CDEF:dmlow$((rrd_idx))=dm$((rrd_idx)),sdev$((rrd_idx)),2,/,-
CDEF:s2d$((rrd_idx))=sdev$((rrd_idx))
AREA:dmlow$((rrd_idx))
AREA:s2d$((rrd_idx))#${COLOR[0]}30:STACK
\
VDEF:avmed$((rrd_idx))=median$((rrd_idx)),AVERAGE
VDEF:avsd$((rrd_idx))=sdev$((rrd_idx)),AVERAGE
CDEF:msr$((rrd_idx))=median$((rrd_idx)),POP,avmed$((rrd_idx)),avsd$((rrd_idx)),/
VDEF:avmsr$((rrd_idx))=msr$((rrd_idx)),AVERAGE
LINE3:avmed$((rrd_idx))#${COLOR[1]}15:
\
COMMENT:"\t\t"
COMMENT:"Average"
COMMENT:"Maximum"
COMMENT:"Minimum"
COMMENT:"Current"
COMMENT:"Std Dev"
COMMENT:" \\j"
\
COMMENT:"Median RTT\: \t"
GPRINT:avmed$((rrd_idx)):"%.2lf"
GPRINT:median$((rrd_idx)):MAX:"%.2lf"
GPRINT:median$((rrd_idx)):MIN:"%.2lf"
GPRINT:median$((rrd_idx)):LAST:"%.2lf"
GPRINT:avsd$((rrd_idx)):"%.2lf"
COMMENT:" \\j"
\
COMMENT:"Packet Loss\:\t"
GPRINT:ploss$((rrd_idx)):AVERAGE:"%.2lf%%"
GPRINT:ploss$((rrd_idx)):MAX:"%.2lf%%"
GPRINT:ploss$((rrd_idx)):MIN:"%.2lf%%"
GPRINT:ploss$((rrd_idx)):LAST:"%.2lf%%"
COMMENT:"  -  "
COMMENT:" \\j"
\
COMMENT:"Loss Colors\:\t"
CDEF:me0=loss$((rrd_idx)),-1,GT,loss$((rrd_idx)),0,LE,*,1,UNKN,IF,median$((rrd_idx)),*
CDEF:meL0=me0,${LINE},-
CDEF:meH0=me0,0,*,${LINE},2,*,+
AREA:meL0
STACK:meH0#${COLOR[1]}:" 0/$((PINGS))"
CDEF:me1=loss$((rrd_idx)),0,GT,loss$((rrd_idx)),1,LE,*,1,UNKN,IF,median$((rrd_idx)),*
CDEF:meL1=me1,${LINE},-
CDEF:meH1=me1,0,*,${LINE},2,*,+
AREA:meL1
STACK:meH1#${COLOR[2]}:" 1/$((PINGS))"
CDEF:me2=loss$((rrd_idx)),1,GT,loss$((rrd_idx)),2,LE,*,1,UNKN,IF,median$((rrd_idx)),*
CDEF:meL2=me2,${LINE},-
CDEF:meH2=me2,0,*,${LINE},2,*,+
AREA:meL2
STACK:meH2#${COLOR[3]}:" 2/$((PINGS))"
CDEF:me3=loss$((rrd_idx)),2,GT,loss$((rrd_idx)),3,LE,*,1,UNKN,IF,median$((rrd_idx)),*
CDEF:meL3=me3,${LINE},-
CDEF:meH3=me3,0,*,${LINE},2,*,+
AREA:meL3
STACK:meH3#${COLOR[4]}:" 3/$((PINGS))"
CDEF:me4=loss$((rrd_idx)),3,GT,loss$((rrd_idx)),4,LE,*,1,UNKN,IF,median$((rrd_idx)),*
CDEF:meL4=me4,${LINE},-
CDEF:meH4=me4,0,*,${LINE},2,*,+
AREA:meL4
STACK:meH4#${COLOR[5]}:" 4/$((PINGS))"
CDEF:me10=loss$((rrd_idx)),4,GT,loss$((rrd_idx)),10,LE,*,1,UNKN,IF,median$((rrd_idx)),*
CDEF:meL10=me10,${LINE},-
CDEF:meH10=me10,0,*,${LINE},2,*,+
AREA:meL10
STACK:meH10#${COLOR[6]}:"10/$((PINGS))"
CDEF:me19=loss$((rrd_idx)),10,GT,loss$((rrd_idx)),19,LE,*,1,UNKN,IF,median$((rrd_idx)),*
CDEF:meL19=me19,${LINE},-
CDEF:meH19=me19,0,*,${LINE},2,*,+
AREA:meL19
STACK:meH19#${COLOR[7]}:"19/$((PINGS))\\j"
\
COMMENT:"Probe\: $((PINGS)) pings every $((STEP)) seconds"
COMMENT:"$(date -d "${START}" | sed 's/\:/\\\:/g') ($(awk -v TIME=$TIME 'BEGIN {printf "%.1f hr", TIME/3600}'))\\j"
EOF
}

if [ ! -r "${fping_rrd}" ]; then
    printf "${0} \"file.rrd\"\n"
else
    STEP=$(rrdtool info "${fping_rrd}" | awk '/^step/{print $NF}')
    PINGS=$(rrdtool info "${fping_rrd}" | awk '/^ds.ping.*index/{count++} END{print count}')

    START="$([ -z "${2}" ] && echo "-7 hours" || echo "${2}")"
    END="$([ -z "${3}" ] && echo "now" || echo "${3}")"
    TIME=$(( $(date -d "${END}" +%s) - $(date -d "${START}" +%s) ))

    eval $(rrd_graph_cmd; rrd_graph_opts)
fi
```
