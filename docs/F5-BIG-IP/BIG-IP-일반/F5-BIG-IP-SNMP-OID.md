---
title: F5 BIG-IP - SNMP OID
author: DDAONG
category: "BIG-IP"
layout: post
---

아래 링크는 F5 공식 가이드 문서입니다.
본 문서 내용에 대한 상세 설명은 아래 문서에서 확인 가능합니다.

[Monitoring BIG-IP System Traffic with SNMP](https://techdocs.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/bigip-external-monitoring-implementations-12-1-2/12.html)

## CPU

~~~
# BIG-IP는 SNMP Response로 Delta Value를 제공합니다.
# Object Name 앞에 ‘F5-BIGIP-SYSTEM-MIB::’가 생략되어 있습니다.
# SNMP 조회 시 생략된 상태인 sysGlobalHostCpuUser5s로 조회해도 되고, F5-BIGIP-SYSTEM-MIB::sysGlobalHostCpuUser5s로 조회해도 됩니다. OID 값을 바로 사용해도 됩니다.
# Object name 마지막의 5s는 Polling Interval을 의미합니다. 5s, 1m, 5m로 구분됩니다. 이 표에서는 5s만 작성합니다.
~~~

### SNMP OID

<table>
<thead>
  <tr>
    <th>Parameter</th>
    <th>Object name</th>
    <th>OID</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td rowspan="9">Global Host CPU Usage</td>
    <td>sysGlobalHostCpuUser5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.20.14</td>
  </tr>
  <tr>
    <td>sysGlobalHostCpuNice5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.20.15</td>
  </tr>
  <tr>
    <td>sysGlobalHostCpuSystem5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.20.16</td>
  </tr>
  <tr>
    <td>sysGlobalHostCpuIdle5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.20.17</td>
  </tr>
  <tr>
    <td>sysGlobalHostCpuIrq5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.20.18</td>
  </tr>
  <tr>
    <td>sysGlobalHostCpuSoftirq5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.20.19</td>
  </tr>
  <tr>
    <td>sysGlobalHostCpuIowait5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.20.20</td>
  </tr>
  <tr>
    <td>sysGlobalHostCpuUsageRatio5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.20.21</td>
  </tr>
  <tr>
    <td>sysGlobalHostCpuUsageRatio</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.20.13</td>
  </tr>
</tbody>
</table>

- Global Host CPU UsageRatio는 시스템 CPU 사용률이며,
  User, Niced, system의 델타 합을 user, niced, system, idle, irq, softirq, iowait 델타 합으로 나눈 값입니다. 여기서 델타는, 최근 10초 간의 Stat에 대한 차이입니다.
- Global Host CPU UsageRatio5s는 산출 방법은 동일하지만 최근 5-second interval에 대한 델타 값을 사용합니다.

BIG-IP는 CPU Core 별로 SNMP OID를 지원합니다. 아래 OID를 조회하면 Core 별로 결과값이 나타납니다.

<table>
<thead>
  <tr>
    <th>Parameter</th>
    <th>Object name</th>
    <th>OID</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td>CPU Count</td>
    <td>sysMultiHostCpuCount</td>
    <td>.1.3.6.1.4.1.3375.2.1.7.4.2.1.5</td>
  </tr>
  <tr>
    <td rowspan="9">CPU Usage</td>
    <td>sysMultiHostCpuUser5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.7.4.2.1.12</td>
  </tr>
  <tr>
    <td>sysMultiHostCpuNice5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.7.4.2.1.13</td>
  </tr>
  <tr>
    <td>sysMultiHostCpuSystem5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.7.4.2.1.14</td>
  </tr>
  <tr>
    <td>sysMultiHostCpuIdle5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.7.4.2.1.15</td>
  </tr>
  <tr>
    <td>sysMultiHostCpuIrq5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.7.4.2.1.16</td>
  </tr>
  <tr>
    <td>sysMultiHostCpuSoftirq5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.7.4.2.1.17</td>
  </tr>
  <tr>
    <td>sysMultiHostCpuIowait5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.7.4.2.1.18</td>
  </tr>
  <tr>
    <td>sysMultiHostCpuUsageRatio5s</td>
    <td>.1.3.6.1.4.1.3375.2.1.7.4.2.1.19</td>
  </tr>
  <tr>
    <td>sysMultiHostCpuUsageRatio</td>
    <td>.1.3.6.1.4.1.3375.2.1.7.4.2.1.11</td>
  </tr>
</tbody>
</table>

## Memory

~~~
BIG-IP는 Memory를 TMM, Other, Swap으로 각각 나누어 사용합니다.

TMM은 실제 트래픽 처리에 사용되는 메모리, Other는 BIG-IP TMOS 구동에 필요한 Memory, Swap은 말 그대로 Swap 영역 메모리입니다.
~~~

TMSH에서 tmsh show sys memory 명령어로 메모리를 조회하면 아래와 같이 구분된 결과를 볼 수 있습니다.

세부내역은 생략하고 옮겨왔습니다.

```bash
Sys::System Memory Information
-----------------------------------------------------------------
Memory Used(%)     Current  Average  Max(since 12/03/20 14:04:34)
-----------------------------------------------------------------
TMM Memory Used  7        7   7
Other Memory Used       73       73  73
Swap Used        2        2   2
 
------------------------
sys::Host Memory (bytes)
------------------------
TMM: 0
  Total      5.1G
  Used     355.3M
  Free       4.7G
Other: 0
  Total      8.6G
  Used       6.3G
  Free       2.3G
Total: 0
  Total     13.7G
  Used       6.6G
  Free       7.1G
Swap: 0
  Total   1023.9M
  Used      16.7M
  Free    1007.2M
```

### SNMP OID

| **Parameter**       | **Object name**       | **OID**       |
| --------------------------- | ----------------------------- | ------------------------------- |
| Host Memory Total   | sysHostMemoryTotal    | .1.3.6.1.4.1.3375.2.1.7.1.1     |
| Host Memory Used    | sysHostMemoryUsed     | .1.3.6.1.4.1.3375.2.1.7.1.2     |
| TMM Memory Total    | sysGlobalTmmStatMemoryTotal   | .1.3.6.1.4.1.3375.2.1.1.2.21.28 |
| TMM Memory Used     | sysGlobalTmmStatMemoryUsed    | .1.3.6.1.4.1.3375.2.1.1.2.21.29 |
| Other(non-TMM) memory Total | sysGlobalHostOtherMemoryTotal | .1.3.6.1.4.1.3375.2.1.1.2.20.44 |
| Other(non-TMM) memory Used  | sysGlobalHostOtherMemoryUsed  | .1.3.6.1.4.1.3375.2.1.1.2.20.45 |
| Swap Memory Total   | sysGlobalHostSwapTotal        | .1.3.6.1.4.1.3375.2.1.1.2.20.46 |
| Swap Memory Used    | sysGlobalHostSwapUsed | .1.3.6.1.4.1.3375.2.1.1.2.20.47 |

Byte와 KiloByte 단위로 표시합니다. KiloByte는 Object Name 뒤에 ‘Kb’를 붙여주면 됩니다.

| **Parameter**       | **Object name**      | **OID**        |
| --------------------------- | ---------------------------- | -------------------------------- |
| Host Memory Total   | sysHostMemoryTotal   | .1.3.6.1.4.1.3375.2.1.7.1.1      |
| Host Memory Used    | sysMultiHostUsed     | .1.3.6.1.4.1.3375.2.1.7.4.2.1.3  |
| TMM Memory Total    | sysTmmStatMemoryTotal        | .1.3.6.1.4.1.3375.2.1.7.4.2.1.31 |
| TMM Memory Used     | sysTmmStatMemoryUsed | .1.3.6.1.4.1.3375.2.1.7.4.2.1.32 |
| Other(non-TMM) memory Total | sysMultiHostOtherMemoryTotal | .1.3.6.1.4.1.3375.2.1.7.4.2.1.7  |
| Other(non-TMM) memory Used  | sysMultiHostOtherMemoryUsed  | .1.3.6.1.4.1.3375.2.1.7.4.2.1.8  |
| Swap Memory Total   | sysMultiHostSwapTotal        | .1.3.6.1.4.1.3375.2.1.7.4.2.1.9  |
| Swap Memory Used    | sysMultiHostSwapUsed | .1.3.6.1.4.1.3375.2.1.7.4.2.1.10 |

Byte와 KiloByte 단위로 표시합니다. KiloByte는 Object Name 뒤에 ‘Kb’를 붙여주면 됩니다.

## Disk

~~~
# Disk의 경우 SNMP OID가 제공되지 않습니다.
# BIG-IP시스템은 Disk Used Space/Available Space를 Diskmonitor Utility를 사용해 체크합니다.
  - Diskmonitor Utility는 주기적으로 동작해 디스크 사용률을 체크하는 스크립트로,
BIG-IP file systems이 가득 차게 되면, LTM Log를 생성 및 Alertd를 통해 메일을 전달하거나
SNMP Trap을 생성하는 동작을 수행합니다.
~~~

### Disk Monitor Utility 동작 구조

#### Inspecting Volumes

Diskmonitor 유틸리티는 아래 볼륨을 기본값으로 체크합니다.

```txt
_root_ = /
config = /config
dev_shm = /dev/shm
shared = /shared
shared_rrd.1.2 = /shared/rrd.1.2
usr = /usr
var = /var
var_log = /var/log
var_loipc = /var/loipc
var_prompt = /var/prompt
var_run = /var/run
var_tmstat = /var/tmstat
shared_vmdisks = /shared/vmdisks
```

#### BIG-IP DB Variables

diskmonitor utility는 BIG-IP DB 변수를 사용하며, 활성화/비활성화, Monitoring 방식, Interval, Threshold 등이 BIG-IP DB 값으로 지정됩니다.

diskmonitor utility의 db variables은 파티션 별로 Customization이 가능합니다.

| **DB Description**  | **TMSH Command**  |
| --------------------------- | ------------------------------------------------------------- |
| **Last Available Value(%)** | list sys db platform.diskmonitor.freelast.\<Partition Name>   |
| **Alert Threshold(%)**      | list sys db platform.diskmonitor.limitalert.\<Partition Name> |
| **Warning Threshold(%)**    | list sys db platform.diskmonitor.limitwarn.\<Partition Name>  |
| **Monitor Interval(%)**     | list sys db platform.diskmonitor.interval   |

#### Crontab

crontab -l 명령어로 diskmonitor의 주기적인 동작을 확인할 수 있습니다.
매 10분마다 동작이 기본값 입니다.(11.5.x and later)

~~~
MAILTO=""
1-59/10 * * * * /usr/bin/diskmonitor
0 */4 * * * /usr/bin/diskwearoutstat
… …
~~~

#### Action - Log

Diskmonitor Utility는 Threshold를 넘으면 로그 파일 /var/log/ltm에 아래와 유사한 로그를 생성합니다.

~~~
diskmonitor: 011d005: Disk partition shared has less than 30% free
diskmonitor: 011d004: Disk partition shared has only 0% free
~~~

#### Action – SNMP Trap

생성되는 SNMP Trap은 아래와 같습니다. (Trap은 평소에 값이 조회되지 않습니다.)
‘F5-BIGIP-COMMON-MIB::’가 생략되어 있습니다.

| **Alert Name**     | **Object name**  | **OID**  |
| -------------------------- | ------------------------ | -------------------------- |
| BIGIP_DMON_ERR_DMON_ALERT  | bigipDiskPartitionWarn   | .1.3.6.1.4.1.3375.2.4.0.25 |
| BIGIP_DMON_ERR_DMON_WARN   | bigipDiskPartitionWarn   | .1.3.6.1.4.1.3375.2.4.0.25 |
| BIGIP_DMON_ERR_DMON_GROWTH | bigipDiskPartitionGrowth | .1.3.6.1.4.1.3375.2.4.0.26 |

다음 명령어로 Default Trap 설정을 조회해볼 수 있습니다.

~~~
#cat /etc/alertd/alert.conf | grep -i diskmonitor -A 10
~~~

명령어의 결과는 아래와 같습니다.

~~~
* from diskmonitor (CR38227)
 * ALERT and WARN send the same trap
 */
alert BIGIP_DMON_ERR_DMON_ALERT {
        snmptrap OID=".1.3.6.1.4.1.3375.2.4.0.25"
}
alert BIGIP_DMON_ERR_DMON_WARN {
        snmptrap OID=".1.3.6.1.4.1.3375.2.4.0.25"
}
alert BIGIP_DMON_ERR_DMON_GROWTH {
        snmptrap OID=".1.3.6.1.4.1.3375.2.4.0.26"
~~~

Diskmonitor에 대한 자세한 내용은 아래 문서에서 확인하실 수 있습니다.

[K8865: Overview of the diskmonitor utility](https://support.f5.com/csp/article/K8865)

## Throughput

~~~
# SNMP로 조회 가능한 Thoughput Response는 합산 값입니다.
# Current 등 Dashboard 및 Tmsh에서 수치를 표현할 때는 이들의 Delta 값과 계산식을 사용합니다.
# 본 문서에는 SNMP OID 값과 계산식을 표기합니다.
  - 문서 가독성을 위해 SSL Transaction (Accel.) 항목은 제외합니다.
~~~

### 용어 설명

자세한 설명은 아래 문서를 참조하시면 좋습니다.
[K50309321: Viewing BIG-IP system throughput statistics](https://support.f5.com/csp/article/K50309321)

| **항목값**       | **설명**  |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Thoughput**    | Management 인터페이스를 제외하고, TMM, PVA 스위치보드를 비롯한 BIG-IP Device의 모든 인터페이스에서 수집한 Throughput 합산 값입니다. (Dashboard에서는 Management Interface Traffic까지 포함된다고 합니다.) <br><br> In: Client-side in/out 트래픽 합계 <br> Out: Server-side in/out 트래픽 합계 <br> Service: Request(Client-side in + Server-side Out), Response(Service-side in + Client-side Out) 값 중 큰 값입니다. |

| **TMM Client-side Throughput****[Detail]** | Client-side의 TMM, PVA에 의해 처리된 합계 throughput입니다.     Client in: 유입(Ingress) 트래픽 합계   Client out: 유출(Egress) 트래픽 합계  |

| **TMM Server-side Throughput****[Detail]** | Server-side의 TMM, PVA에 의해 처리된 합계 throughput입니다.     Server in: 유입(Ingress) 트래픽 합계   Server out: 유출(Egress) 트래픽 합계 |

### SNMP OID

```txt
# Object Name 앞에 ‘F5-BIGIP-SYSTEM-MIB::’가 생략되어 있습니다.

# Object name 마지막의 5s는 Polling Interval, Average를 의미합니다. 5s, 1m, 5m로 구분됩니다.
 이 표에서는 5s만 작성합니다.

# Object Name의 sysStat 뒤에 Pva를 추가하면 PVA 트래픽을 별도 확인할 수 있습니다.
\<예시> sysStatClientBytesIn.0 => sysStatPvaClientBytesIn.0
```

<table>
<thead>
  <tr>
    <th>Statistics</th>
    <th>Group</th>
    <th>Object Name</th>
    <th>OID</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td rowspan="16">Throughput</td>
    <td rowspan="4">Client In</td>
    <td>sysStatClientBytesIn.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.3.0</td>
  </tr>
  <tr>
    <td>sysStatClientPktsIn.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.2.0</td>
  </tr>
  <tr>
    <td>sysStatClientBytesIn5s.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.60.0</td>
  </tr>
  <tr>
    <td>sysStatClientPktsIn5s.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.59.0</td>
  </tr>
  <tr>
    <td rowspan="4">Client Out</td>
    <td>sysStatClientBytesOut.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.5.0</td>
  </tr>
  <tr>
    <td>sysStatClientPktsOut.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.4.0</td>
  </tr>
  <tr>
    <td>sysStatClientBytesOut5s.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.62.0</td>
  </tr>
  <tr>
    <td>sysStatClientPktsOut5s.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.61.0</td>
  </tr>
  <tr>
    <td rowspan="4">Server In</td>
    <td>sysStatServerBytesIn.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.10.0</td>
  </tr>
  <tr>
    <td>sysStatServerPktsIn.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.9.0</td>
  </tr>
  <tr>
    <td>sysStatServerBytesIn5s.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.67.0</td>
  </tr>
  <tr>
    <td>sysStatServerPktsIn5s.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.66.0</td>
  </tr>
  <tr>
    <td rowspan="4">Server Out</td>
    <td>sysStatServerBytesOut.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.12.0</td>
  </tr>
  <tr>
    <td>sysStatServerPktsOut.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.11.0</td>
  </tr>
  <tr>
    <td>sysStatServerBytesOut5s.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.69.0</td>
  </tr>
  <tr>
    <td>sysStatServerPktsOut5s.0</td>
    <td>.1.3.6.1.4.1.3375.2.1.1.2.1.68.0</td>
  </tr>
</tbody>
</table>

### Calculation

- 각 계산식의 Polling Interval은 10s가 기본값입니다.
  - 예를 들어 \<DeltasysStatServerBytesIn> 값을 추출하려면 OID sysStatServerBytesIn에 대해 10초 간격으로 두 번의 조회가 필요합니다.

<table>
<thead>
  <tr>
    <th>Statistic</th>
    <th>Group</th>
    <th>계산식</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td rowspan="2">Throughput</td>
    <td>In</td>
    <td>( [DeltaStatClientBytesIn] + [DeltasysStatClientBytesOut] ) *8 / [interval]</td>
  </tr>
  <tr>
    <td>Out</td>
<td>( [DeltaStatServerBytesIn] + [DeltaStatServerBytesOut] )* 8 / [interval]</td>
  </tr>
  <tr>
    <td rowspan="2">TMM Client-side Throughput</td>
    <td>Client Bits In</td>
    <td>( [DeltaStatClientBytesIn] *8 ) / [interval]</td>
  </tr>
  <tr>
    <td>Client Bits Out</td>
<td>( [DeltaStatClientBytesOut]* 8 ) / [interval]</td>
  </tr>
  <tr>
    <td rowspan="2">TMM Server-side Throughput</td>
    <td>Server Bits In</td>
    <td>( [DeltaStatServerBytesIn] *8 ) / [interval]</td>
  </tr>
  <tr>
    <td>Server Bits Out</td>
<td>( [DeltaStatServerBytesOut]* 8 ) / [interval]</td>
  </tr>
  <tr>
    <td rowspan="2">New Connections Summary</td>
    <td>Client Accepts</td>
    <td>[DeltaTcpStatAccept] / [interval]</td>
  </tr>
  <tr>
    <td>Server Connects</td>
    <td>[DeltaStatServerTotConns] /[interval]</td>
  </tr>
  <tr>
    <td rowspan="2">Total New Connections</td>
    <td>Client Connects</td>
    <td>[DeltaStatClientTotConns] / [interval]</td>
  </tr>
  <tr>
    <td>Server Connects</td>
    <td>[DeltaStatServerTotConns] / [interval]</td>
  </tr>
  <tr>
    <td rowspan="2">New Accepts / Connects</td>
    <td>Client Accepts</td>
    <td>[DeltaTcpStatAccepts] / [interval]</td>
  </tr>
  <tr>
    <td>Server Connects</td>
    <td>[DeltaTcpStatConnects] / [interval]</td>
  </tr>
</tbody>
</table>
