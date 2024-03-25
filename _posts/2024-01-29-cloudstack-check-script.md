---
title: Cloudstack check script
date: 2024-03-21 16:52:00 +09:00
categories: [Cloudstack, python script]
tags: [cloudstact, python]
---


※Cloudstack에서 제공하는 api를 이용하여 cloudstack 환경안의 요소들을 python을 통해 cli에서 간단히 체크 하기위한 스크립트 작성 글입니다.
※스크립트를 통해서 Cloudstack 환경에서의 리소스들에 대한 상태 및 사용량 체크가 가능합니다.
※Cloudstack 버전 및 python 등의 버전은 실제 제가 관리하는 서버 버전을 기준으로 작성 하였습니다.

[환경]
```bash
- cloudstack : 4.11.3.0
- OS : CentOS Linux release 7.9.2009 (Core)
- python : Python 2.7.5
```

# Cloudstack과 api
## cloudstack
1. 인프라스트럭처 클라우드 서비스를 생성, 관리, 디플로이를 하기위한 오픈소스 IaaS(Infrastructure as a Service) 클라우드 컴퓨팅 플랫폼입니다.
2. KVM, ESXi VMWare, XenServer/XCP와 같은 기존의 하이퍼바이저들을 사용
3. 자체 API 외에도 클라우드스택은 아마존 웹 서비스(AWS) API, 또 오픈 그리드 포럼의 오픈 클라우드 컴퓨팅 인터페이스를 지원한다.

## cloudstack_API
cloudstack은 대부분의 기능을 web api로 제공합니다.  
고급 응용을 위해서는 web api의 사용하는 것이 효율적입니다.  

- cloudstack api docs를 참고하시려면 아래 링크에 들어가시면 됩니다.  
[cloudstack api docs](https://cloudstack.apache.org/api.html)  

- 아래 링크는 cloudstack api를 호출하기위한 기본적인 방법을 설명하고 있으므로 참고 하시면 됩니다.  
[developers guide](https://docs.cloudstack.apache.org/en/latest/developersguide/developer_guide.html#the-cloudstack-api)  


# python 스크립트
cloudstack web ui에서 확인가능한 정보들을 python을 통하여 api로 필요한 값을 가공하여 사용할 수 있습니다.  
아래는 api를 활용하여 web ui에서 확인 가능한 서비스의 상태 및 리소스 사용량 데이터들을 출력하는 스크립트를 작성하였습니다.  
*※cloudstack의 api 호출에 대해서 python을 사용하는 예시를 공식사이트에서 제공하고 있기 때문에 python언어를 사용하였으며, 스크립트는 해당 서버 안에서 동작시킵니다.*  
```bash
cat << EOF > checkcloud.py
# Cloudstack Check with python
--------------------------------------------------------------------------------------------------------

#!/usr/bin/env python
# checkcloudstack.py 2

import sys
import urllib2
import urllib
import hashlib
import hmac
import base64
import json

def getReq (command):
  baseurl='http://localhost:8080/client/api?'
  request={}
  request['command']=command
  request['response']='json'
  request['apikey']='L9vB6WEuA4Lv07vQ_ITN6aWXSogEX4_3c2UUck9RlqK0p0bP4N92xBJMHSxeXOYSEaqeRPvhAX2vw0VOPs3Hug'
  secretkey='ubMFyOh_GBRGsD-UbMSH6BgdCh5_VOj-7qtsxzaxqQQZkcmBb_1Sy1l27AqYUkNBX_6oGwM8Vrm8yEVXkyAvJg'

  request_str='&'.join(['='.join([k,urllib.quote_plus(request[k])]) for k in request.keys()])

  sig_str='&'.join(['='.join([k.lower(),urllib.quote_plus(request[k].lower().replace('+','%20'))])for k in sorted(request.iterkeys())])

  sig=hmac.new(secretkey,sig_str,hashlib.sha1)
  sig=hmac.new(secretkey,sig_str,hashlib.sha1).digest()
  sig=base64.encodestring(hmac.new(secretkey,sig_str,hashlib.sha1).digest())
  sig=base64.encodestring(hmac.new(secretkey,sig_str,hashlib.sha1).digest()).strip()
  sig=urllib.quote_plus(base64.encodestring(hmac.new(secretkey,sig_str,hashlib.sha1).digest()).strip())

  req = baseurl + request_str + '&signature=' + sig

  return req


def getHealth_01 (keyword, command):
  print ('[{}]'.format(keyword))

  req = getReq (command)

  res = urllib2.urlopen(req)
  result = res.read()

  resultParsed = json.loads (result)

  responseKeyword = "{}response".format(command.lower())

  # print (json.dumps(resultParsed[responseKeyword], indent=2, sort_keys=True))

  for obj in resultParsed[responseKeyword][keyword]:
    allocationstateExist = "allocationstate" in obj

    if allocationstateExist:
      print ("{}: {}".format (obj['name'], obj['allocationstate']))
    else:
      print ("{}: {}".format (obj['name'], obj['state']))
  print("")


def getHealth_02 (keyword, command):
  print ('[{}]'.format(keyword))

  req = getReq (command)
  res = urllib2.urlopen(req)
  result = res.read()

  resultParsed = json.loads (result)

  responseKeyword = "{}response".format(command.lower())

  # print (json.dumps(resultParsed[responseKeyword], indent=2, sort_keys=True))

  for obj in resultParsed[responseKeyword][keyword]:
    hosttagsExist = "hosttags" in obj

    if hosttagsExist:
      print ("{}: {} / {}".format (obj['hosttags'], obj['state'], obj['resourcestate']))
    else:
      print ("{}: {} / {}".format (obj['name'], obj['state'], obj['resourcestate']))

  print("")


def getHealth_03 (keyword, command):
  print ('[{}]'.format(keyword))
  print ("if VM is not Running, then print!")
  print('--------------------------------------')

  req = getReq (command)
  res = urllib2.urlopen(req)
  result = res.read()

  resultParsed = json.loads (result)

  responseKeyword = "{}response".format(command.lower())

  # print (json.dumps(resultParsed[responseKeyword], indent=2, sort_keys=True))

  for obj in resultParsed[responseKeyword][keyword]:
    if (obj['state'] != 'Running'):
      print ("{}: {}".format (obj['name'], obj['state']))

  print("")


# event 출력을 위한 코드로 INFO 레벨이 아닌경우 출력(INFO만 존재시 출력X)
def getHealth_04 (keyword, command):
  print ('[{}]'.format(keyword))
  print ("if level is not INFO, then print!")
  print('--------------------------------------')

  req = getReq (command)
  res = urllib2.urlopen(req)
  result = res.read()

  resultParsed = json.loads (result)

  responseKeyword = "{}response".format(command.lower())

  # print (json.dumps(resultParsed[responseKeyword], indent=2, sort_keys=True))

  for obj in resultParsed[responseKeyword][keyword]:
    if (obj['level'] != 'INFO'):
      print(obj['created'])
      print("Domain: {} / Account: {}".format(obj['domain'], obj['account']))
      print(obj['description'])
      print('--------------------------------------')

  print("")


#alart 메시지 출력 코드로 스크립트 실행 시 숫자를 입력하면 숫자만큼의 alart 메시지 출력
#예) ./checkcloud.py 3
def getHealth_05 (keyword, command):
  print ('[{}]'.format(keyword))

  req = getReq (command)
  res = urllib2.urlopen(req)
  result = res.read()

  resultParsed = json.loads (result)

  responseKeyword = "{}response".format(command.lower())

  # print (json.dumps(resultParsed[responseKeyword], indent=2, sort_keys=True))

  for num, obj in enumerate(resultParsed[responseKeyword][keyword], start=1):
    print(obj['sent'])
    print(obj['name'])
    print(obj['description'])
    print('--------------------------------------')

    if (num == int(sys.argv[1])):
      break

  print("")


def getHealth_06 (keyword, command):
  print ('[{}]'.format(keyword))

  req = getReq (command)
  res = urllib2.urlopen(req)
  result = res.read()

  resultParsed = json.loads (result)

  responseKeyword = "{}response".format(command.lower())

  # print (json.dumps(resultParsed[responseKeyword], indent=2, sort_keys=True))

  for obj in resultParsed[responseKeyword][keyword]:
    if (obj['type'] == 'Routing'):

      cpuAvail = 100-float(obj['cpuused'][:-1])
      memTotalGB = float(obj['memorytotal']) / 1024 / 1024 / 1024
      memTotalTB = memTotalGB / 1024
      memAvailGB = (float(obj['memorytotal']) - float(obj['memoryused'])) / 1024 / 1024 / 1024
      memAvailTB = memAvailGB / 1024

      print("{}".format(obj['name']))
      print("CPU Avail: {:.2f} %".format(cpuAvail))
      print("Memory Total: {:.2f} GB / {:.2f} TB".format(memTotalGB, memTotalTB))
      print("Memory Avail: {:.2f} GB / {:.2f} TB".format(memAvailGB, memAvailTB))
      print('--------------------------------------')

  print("")


def getHealth_07 (keyword, command):
  print ('[{}]'.format(keyword))

  req = getReq (command)
  res = urllib2.urlopen(req)
  result = res.read()

  resultParsed = json.loads (result)

  responseKeyword = "{}response".format(command.lower())

  # print (json.dumps(resultParsed[responseKeyword], indent=2, sort_keys=True))

  for obj in resultParsed[responseKeyword][keyword]:
    print("{}".format(obj['tags']))
    print("Disk Total: {:.2f} TB".format(float(obj['disksizetotal']) / 1024 / 1024 / 1024 / 1024))
    print("Disk Avail: {:.2f} TB".format((float(obj['disksizetotal']) - float(obj['disksizeallocated'])) / 1024 / 1024 / 1024 / 1024 ))
    print('--------------------------------------')


def getHealth_08 (command):
  print ('secondary')

  req = getReq (command)
  res = urllib2.urlopen(req)
  result = res.read()

  resultParsed = json.loads (result)

  # print (json.dumps(resultParsed, indent=2, sort_keys=True))

  for obj in resultParsed['listcapacityresponse']['capacity']:
    if (obj['type'] == 6):
      print ("Name: {}".format(obj['name']))
      print ("Zone Name: {}".format(obj['zonename']))
      print ("Disk Total: {:.2f} GB".format(float(obj['capacitytotal']) /1024 / 1024 / 1024))
      print ("Disk Avail: {:.2f} GB".format( (float(obj['capacitytotal'] - float(obj['capacityused']))) / 1024 / 1024 / 1024))
      print('--------------------------------------')

  print ("")

getHealth_01 ('zone', 'listZones')
getHealth_01 ('pod', 'listPods')
getHealth_01 ('cluster', 'listClusters')
getHealth_01 ('storagepool', 'listStoragePools')
getHealth_01 ('systemvm', 'listSystemVms')
#getHealth_01 ('router', 'listRouters')
#getHealth_01 ('network', 'listNetworks')

getHealth_02 ('host', 'listHosts')
#getHealth_03 ('virtualmachine', 'listVirtualMachines')
getHealth_04 ('event', 'listEvents')
getHealth_05 ('alert', 'listAlerts')
getHealth_06 ('host', 'listHosts')
getHealth_07 ('storagepool', 'listStoragePools')
getHealth_08 ('listCapacity')


print("[Info]")
print("cloudstack main dashboard / vm status / network / vpc / virtualrouter / check needed!")
EOF

chmod +x checkcloudstack.py
```

위 스크립트 동작시 아래와 같이 cloudstack의 dashboard에서 확인 가능한 정보(zone, pod, cluster, systemvm, hosts, storagepool)들과 리소스 사용량을 확인할 수 있습니다.  
```bash
[root@cvm ~]# checkcloudstack.py 2
[zone]
ManageZone: Enabled
IntZone: Enabled
ExtZone: Enabled
DRZone: Enabled

[pod]
ManagePod: Enabled
IntPoD: Enabled
ExtPoD: Enabled
DR_PoD: Enabled

[cluster]
ManageCluster: Enabled
IntCluster: Enabled
ExtCluster: Enabled
DR_Cluster: Enabled

[storagepool]
MAN_SSD_DATA_POOL: Up
INT_SSD_DATA_POOL: Up
MAN-SATA-CVM-POOL: Up
DR-SATA-DATA2-POOL: Up
DR-SATA-DATA1-POOL: Up
DR-SSD-OS: Up
EXT-SAS-DATA2-POOL2: Up
EXT-SAS-DATA1-POOL2: Up
EXT-SAS-DATA2-POOL1: Up
EXT-SAS-DATA1-POOL1: Up
EXT-SSD-OS-POOL2: Up
EXT-SSD-OS-POOL1: Up
INT-SAS-DATA3-POOL2: Up
INT-SAS-DATA2-POOL2: Up
INT-SAS-DATA1-POOL2: Up
INT-SAS-DATA3-POOL1: Up
INT-SAS-DATA2-POOL1: Up
INT-SAS-DATA1-POOL1: Up
INT-SSD-OS-POOL2: Up
Manage-SATA-DATA1: Up
INT-SSD-OS-POOL1: Up
Manage-SSD-OS: Up

[systemvm]
v-4-VM: Running
s-17-VM: Running
v-153-VM: Running
s-240-VM: Running
v-682-VM: Running
s-683-VM: Running
s-693-VM: Running
v-694-VM: Running

[host]
s-693-VM: Up / Enabled
v-694-VM: Up / Enabled
s-683-VM: Up / Enabled
v-682-VM: Up / Enabled
s-240-VM: Up / Enabled
v-153-VM: Up / Enabled
DR-edge1: Up / Enabled
Man-edge2: Up / Enabled
EXT-edge4: Up / Enabled
INT-edge4: Up / Enabled
EXT-edge3: Up / Enabled
EXT-edge2: Up / Enabled
EXT-edge1: Up / Enabled
INT-edge3: Up / Enabled
INT-edge2: Up / Enabled
s-17-VM: Up / Enabled
INT-edge1: Up / Enabled
v-4-VM: Up / Enabled
Man-edge1: Up / Enabled

[event]
if level is not INFO, then print!
--------------------------------------

[alert]
2023-11-18T19:24:16+0900
ALERT.SERVICE.CONSOLEPROXY
Console proxy up in zone: DRZone, proxy: v-694-VM, public IP: x.x.x.x, private IP: x.x.x.x
--------------------------------------
2023-11-18T19:24:14+0900
ALERT.SERVICE.SSVM
Secondary Storage Vm up in zone: DRZone, secStorageVm: s-693-VM, public IP: x.x.x.x, private IP: x.x.x.x
--------------------------------------

[host]
kcardr-edge1
CPU Avail: 77.27 %
Memory Total: 381.66 GB / 0.37 TB
Memory Avail: 92.88 GB / 0.09 TB
--------------------------------------
kcarman-edge2
CPU Avail: 92.95 %
Memory Total: 254.66 GB / 0.25 TB
Memory Avail: 27.84 GB / 0.03 TB
--------------------------------------
kcarext-edge4
CPU Avail: 98.44 %
Memory Total: 254.66 GB / 0.25 TB
Memory Avail: 91.47 GB / 0.09 TB
--------------------------------------
kcarint-edge4
CPU Avail: 96.96 %
Memory Total: 254.66 GB / 0.25 TB
Memory Avail: 78.49 GB / 0.08 TB
--------------------------------------
kcarext-edge3
CPU Avail: 89.21 %
Memory Total: 254.66 GB / 0.25 TB
Memory Avail: 125.67 GB / 0.12 TB
--------------------------------------
kcarext-edge2
CPU Avail: 96.55 %
Memory Total: 254.66 GB / 0.25 TB
Memory Avail: 103.18 GB / 0.10 TB
--------------------------------------
kcarext-edge1
CPU Avail: 97.51 %
Memory Total: 254.66 GB / 0.25 TB
Memory Avail: 135.00 GB / 0.13 TB
--------------------------------------
kcarint-edge3
CPU Avail: 94.41 %
Memory Total: 254.66 GB / 0.25 TB
Memory Avail: 65.15 GB / 0.06 TB
--------------------------------------
kcarint-edge2
CPU Avail: 89.09 %
Memory Total: 254.66 GB / 0.25 TB
Memory Avail: 101.60 GB / 0.10 TB
--------------------------------------
kcarint-edge1
CPU Avail: 95.13 %
Memory Total: 254.66 GB / 0.25 TB
Memory Avail: 93.93 GB / 0.09 TB
--------------------------------------
kcarman-edge1
CPU Avail: 75.01 %
Memory Total: 254.66 GB / 0.25 TB
Memory Avail: 72.83 GB / 0.07 TB
--------------------------------------

[storagepool]
MAN_SSD_DATA_POOL
Disk Total: 14.24 TB
Disk Avail: 5.99 TB
--------------------------------------
INT_SSD_DATA_POOL
Disk Total: 6.70 TB
Disk Avail: 0.36 TB
--------------------------------------
MAN-SATA-CVM-POOL
Disk Total: 7.27 TB
Disk Avail: 7.08 TB
--------------------------------------
DR-SATA-DATA2-POOL
Disk Total: 14.55 TB
Disk Avail: 10.06 TB
--------------------------------------
DR-SATA-DATA1-POOL
Disk Total: 52.75 TB
Disk Avail: 36.36 TB
--------------------------------------
DR-SSD-OS
Disk Total: 1.73 TB
Disk Avail: 0.36 TB
--------------------------------------
EXT-SAS-DATA2-POOL2
Disk Total: 7.27 TB
Disk Avail: 6.05 TB
--------------------------------------
EXT-SAS-DATA1-POOL2
Disk Total: 1.82 TB
Disk Avail: 0.79 TB
--------------------------------------
EXT-SAS-DATA2-POOL1
Disk Total: 7.27 TB
Disk Avail: 3.31 TB
--------------------------------------
EXT-SAS-DATA1-POOL1
Disk Total: 1.82 TB
Disk Avail: 0.35 TB
--------------------------------------
EXT-SSD-OS-POOL2
Disk Total: 3.46 TB
Disk Avail: 1.22 TB
--------------------------------------
EXT-SSD-OS-POOL1
Disk Total: 3.46 TB
Disk Avail: 1.51 TB
--------------------------------------
INT-SAS-DATA3-POOL2
Disk Total: 10.91 TB
Disk Avail: 4.26 TB
--------------------------------------
INT-SAS-DATA2-POOL2
Disk Total: 1.82 TB
Disk Avail: 0.72 TB
--------------------------------------
INT-SAS-DATA1-POOL2
Disk Total: 1.82 TB
Disk Avail: 0.09 TB
--------------------------------------
INT-SAS-DATA3-POOL1
Disk Total: 10.91 TB
Disk Avail: 7.54 TB
--------------------------------------
INT-SAS-DATA2-POOL1
Disk Total: 1.82 TB
Disk Avail: 1.82 TB
--------------------------------------
INT-SAS-DATA1-POOL1
Disk Total: 1.82 TB
Disk Avail: 0.94 TB
--------------------------------------
INT-SSD-OS-POOL2
Disk Total: 3.46 TB
Disk Avail: 2.48 TB
--------------------------------------
Manage-SATA-DATA
Disk Total: 43.65 TB
Disk Avail: 22.51 TB
--------------------------------------
INT-SSD-OS-POOL1
Disk Total: 3.46 TB
Disk Avail: -0.60 TB
--------------------------------------
Manage-SSD-OS
Disk Total: 1.73 TB
Disk Avail: 0.21 TB
--------------------------------------
secondary
Name: SECONDARY_STORAGE
Zone Name: DRZone
Disk Total: 999.51 GB
Disk Avail: 970.69 GB
--------------------------------------
Name: SECONDARY_STORAGE
Zone Name: ExtZone
Disk Total: 999.51 GB
Disk Avail: 704.47 GB
--------------------------------------
Name: SECONDARY_STORAGE
Zone Name: IntZone
Disk Total: 999.51 GB
Disk Avail: 807.63 GB
--------------------------------------
Name: SECONDARY_STORAGE
Zone Name: ManageZone
Disk Total: 999.51 GB
Disk Avail: 762.05 GB
--------------------------------------

[Info]
cloudstack main dashboard / vm status / network / vpc / virtualrouter / check needed!
```