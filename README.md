# sng2sec
Simple prototype to feed Simple Event Correlator from Syslog-Ng daemon.

## Description
This is proof of concept for processing syslog messages received from network (Switch, Router etc.) devices. 
To reduce the event flow to upper layer event management system or ticketing system we will use event correlation software.

Objectives:
*  identify important (hw fault, routing issues ...) syslog events base on pattern matching
*  tag and classify matching events
*  **normalize** and enrich (tag, classify) syslog message flow before send to correlator 
*  move patter matching outside of correlator (for network devices we may have *n* thousand various messages per vendor/technology)
*  use correlator to reduce flow to event management system or ticketing system
*  suppress short outages, de-duplicate reoccurring events and detect status flapping

Shortly. I want to go from this:

```
2020-04-04T10:00:00+00:00 router-01.com local7.err %LINK-3-UPDOWN:  Interface FastEthernet0/44, changed state to down
```

to this on correlator input and output.

```
 2020-04-04T10:05:00+00:00 host="router-01.com" ip="1.2.89.76" eventid="syslog.cisco-link-updown" actionable="yes" alarm_type="1" component="FastEthernet0/4" correlation="PairWithWindow-05" correlation_group="LINK-3-UPDOWN" message="%LINK-3-UPDOWN: Interface FastEthernet0/4, changed state to down"
```

## Prerequisites
*  [syslog-ng](https://github.com/syslog-ng/syslog-ng) version >= 3.18
*  [sec](https://github.com/simple-evcorr/sec) version >= 2.7.12
*  Perl `Storabe` module (optional)

## Install

Copy files from this repo in respective locations.

```bash
etc/sec/correlator.conf
etc/syslog-ng/conf.d/sng.conf
etc/syslog-ng/patterndb.d/syslog-messages.xml
```

Create below directory to store SEC content.

```bash
mkdir -p /var/log/correlator
```

Restart syslog-ng and start sec correlator.

```
systemctl restart syslog-ng
sec -detach -tail -conf=/etc/sec/correlator.conf -input=/var/log/correlator/sec-input-events.log -log=/var/log/correlator/sec.log -intevents -intcontexts -dumpfts -dump=/var/log/correlator/sec.dump -debug=4 -pid=/var/run/sec2.pid
```

## Test

If there is no syslog traffics landing on your test system, you can sniff it on production system using `tcpdump` command.

```
tcpdump -v -s 0 -w syslog.dump -nn udp dst port 514
```

Replay recorded traffics from system A against system B(your testing box):

To replay recorded traffics, first rewrite MAC/IP of recorded datagrams:

```
tcprewrite --fixcsum \
  --enet-dmac=52:54:00:FF:00:00 \
  --dstipmap=0.0.0.0/0:20.30.40.50 \
  --infile=/tmp/syslog.dump \
  --outfile=/var/syslog.redump
```

Replay from A to B

```
tcpreplay -i eth0 --mbps=1 /var/syslog.redump
```

You can also test pattern matching using syslog-ng `pdbtool`

```bash
$ export PDB=/etc/syslog-ng/patterndb.d/syslog-messages.xml
$ pdbtool match -p $PDB -P '%ETH_PORT_CHANNEL-5-PORT_DOWN' -M 'port-channel23: Ethernet1/15 is down'

MESSAGE=port-channel23: Ethernet1/15 is down
PROGRAM=%ETH_PORT_CHANNEL-5-PORT_DOWN
.classifier.class=alarm
.classifier.rule_id=70c95a6a7ae94e7f2846e7bfae6ced31
subobject=port-channel23: Ethernet1/15
TAGS=.classifier.alarm

MESSAGE=port-channel23: Ethernet1/15 is down
PROGRAM=%ETH_PORT_CHANNEL-5-PORT_DOWN
.classifier.class=alarm
.classifier.rule_id=70c95a6a7ae94e7f2846e7bfae6ced31
subobject=port-channel23: Ethernet1/15
alarm.eventid=syslog.cisco-eth_port_channel-port_down
alarm.actionable=no
alarm.component=port-channel23: Ethernet1/15
alarm.correlation_group=ETH_PORT_CHANNEL-PORT-UD
alarm.alarm_type=1
alarm.correlation=PairWithWindow-05
TAGS=.classifier.alarm

```