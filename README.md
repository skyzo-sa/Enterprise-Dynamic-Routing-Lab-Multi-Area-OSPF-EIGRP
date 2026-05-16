## Enterprise Dynamic Routing Lab – Multi-Area OSPF & EIGRP

### Obejective

Mastering OSPF and EIGRP configuration, troubleshooting, and redistribution.

### Technical Skills Gained

- OSPF areas
- EIGRP metrics
- Route redistribution
- Path selection

##  Tools and Technologies Used

- Network Simulation Tool (EVE-NG) used to simulate enterprise networking environments virtually.
- Cisco Networking Technologies
- 5x Cisco IOSv routers (R1-R5)
- 3x Cisco IOSvL2 switches (SW1-SW3)
- 3x Linux containers (PC1-PC3)
- Cisco IOS CLI

**Ref: Network Diagram**
<img width="1810" height="909" alt="image" src="https://github.com/user-attachments/assets/e0e8964f-eae9-413d-9e4c-b7e48dc743a8" />

### IP Addressing Scheme
```
OSPF Backbone (Area 0):
R1 G0/0: 10.0.12.1/30 ↔ R2 G0/0: 10.0.12.2/30
R2 G0/1: 10.0.23.1/30 ↔ R3 G0/0: 10.0.23.2/30

Area 1 (Stub):
R1 Loopback0: 1.1.1.1/32
R1 Loopback1: 192.168.1.1/24

Area 2 (NSSA):
R2 Loopback0: 2.2.2.2/32
R2 Loopback1: 192.168.2.1/24

EIGRP AS 100:
R3 G0/1: 10.0.34.1/30 ↔ R4 G0/0: 10.0.34.2/30
R4 G0/1: 10.0.45.1/30 ↔ R5 G0/0: 10.0.45.2/30

User Networks:
PC1: 192.168.10.10/24 via R1
PC2: 192.168.20.10/24 via R2
PC3: 192.168.30.10/24 via R4
```

## CONFIGURATION STEPS

### **Step 1: EVE-NG Lab Creation**

1. Create new lab: `IP-Connectivity-Lab`
2. Add 5x Cisco IOSv routers (R1-R5)
3. Add 3x Cisco IOSvL2 switches (SW1-SW3)
4. Add 3x Linux containers (PC1-PC3)
5. Connect as per topology diagram

### **Step 2: Basic Router Configuration**

**On R1:**
```
enable
configure terminal
hostname R1
no ip domain-lookup
ip cef
!
! Configure interfaces
interface GigabitEthernet0/1
 description Link to R2
 ip address 10.0.12.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0
 description Link to SW1
 ip address 192.168.10.1 255.255.255.0
 no shutdown
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
!
interface Loopback1
 ip address 192.168.1.1 255.255.255.0
!
line console 0
 logging synchronous
 exec-timeout 0 0
end
write memory
```
**On R2:**
```
hostname R2
!
interface GigabitEthernet0/1
 description Link to R1
 ip address 10.0.12.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/2
 description Link to R3
 ip address 10.0.23.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/0
 description Link to SW2
 ip address 192.168.20.1 255.255.255.0
 no shutdown
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
!
interface Loopback1
 ip address 192.168.2.1 255.255.255.0
end
write memory
```
**On R3:**
```
hostname R3
!
interface GigabitEthernet0/2
 description Link to R2
 ip address 10.0.23.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description Link to R4
 ip address 10.0.34.1 255.255.255.252
 no shutdown
!

interface GigabitEthernet0/0
 description Link to R5
 ip address 10.0.45.1 255.255.255.252
 no shutdown
 !
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
end
write memory
```
**On R4:**
```
hostname R4
!
interface GigabitEthernet0/0
 description Link to R3
 ip address 10.0.34.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description Link to SW3
 ip address 192.168.30.1 255.255.255.0
 no shutdown
!
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
end
write memory
```
**On R5:**
```
hostname R5
!
interface GigabitEthernet0/0
 description Link to R3
 ip address 10.0.45.2 255.255.255.252
 no shutdown
!
interface Loopback0
 ip address 5.5.5.5 255.255.255.255
!
interface Loopback1
 ip address 192.168.100.1 255.255.255.0
end
write memory
```
### **Step 3: Switch and PC Configuration**

**On SW1:**
```
hostname SW1
vlan 10
 name USERS
!
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
!
interface GigabitEthernet0/0
 no switchport
 ip address 192.168.10.2 255.255.255.0
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 192.168.10.1
end
```
**On PC1 (Linux container):**
```
ip addr add 192.168.10.10/24 dev eth0
ip route add default via 192.168.10.1
```
**Configure SW2/PC2 and SW3/PC3 similarly with their respective subnets.**

---

## **Part 2: OSPF Configuration**

### **Step 1: OSPF Basic Configuration on R1**
```
! On R1
configure terminal
router ospf 10
 router-id 1.1.1.1
!
! Configure Area 0 (Backbone)
 network 10.0.12.0 0.0.0.3 area 0
!
! Configure Area 1 as Stub
 network 192.168.1.0 0.0.0.255 area 1
 network 1.1.1.1 0.0.0.0 area 1
 network 192.168.10.0 0.0.0.255 area 1
!
! Configure Area 1 as Stub area
 area 1 stub
!
! Configure OSPF timers (optional tuning)
 auto-cost reference-bandwidth 1000
 timers throttle spf 10 100 5000
!
passive-interface GigabitEthernet0/0
default-information originate
end
```
**Step 2: OSPF Configuration on R2**
```
! On R2
configure terminal
router ospf 10
 router-id 2.2.2.2
!
! Configure Area 0 (Backbone)
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
!
! Configure Area 2 as NSSA
 network 192.168.2.0 0.0.0.255 area 2
 network 2.2.2.2 0.0.0.0 area 2
 network 192.168.20.0 0.0.0.255 area 2
!
! Configure Area 2 as NSSA
 area 2 nssa
!
	passive-interface GigabitEthernet0/0
end
```
**Step 3: OSPF Configuration on R3 (OSPF Area 0)**
```
! On R3
configure terminal
router ospf 10
 router-id 3.3.3.3
!
! Configure only in Area 0
 network 10.0.23.0 0.0.0.3 area 0
 network 3.3.3.3 0.0.0.0 area 0
!
! Configure summarization (optional)
 area 0 range 10.0.0.0 255.255.0.0
!
end
```
### **Step 4: Verify OSPF Configuration**

**On R1:**
```
show ip ospf neighbor
show ip ospf interface brief
show ip ospf database
show ip route ospf
show ip protocols
```
**On R2:**
```
show ip ospf neighbor detail
show ip ospf database nssa-external
show ip route
```
**On R3:**
```
show ip ospf border-routers
show ip ospf virtual-links
```
## **Part 3: EIGRP Configuration**

### **Step 1: EIGRP Configuration on R3**
```
! On R3
configure terminal
router eigrp 100
 !
 ! Configure EIGRP with named mode (modern approach)
 address-family ipv4 unicast autonomous-system 100
  !
  ! Configure networks
  network 10.0.34.0 0.0.0.3
 network 10.0.45.0 0.0.0.3
  network 3.3.3.3 0.0.0.0
  !
  ! Configure EIGRP parameters
  metric weights 0 1 1 1 1 0
  maximum-paths 4
  variance 2
  !
  ! Configure passive interfaces
  passive-interface default
  no passive-interface GigabitEthernet0/2
  !
  ! Configure summarization
  af-interface GigabitEthernet0/1
   summary-address 10.0.0.0 255.255.0.0
  exit-af-interface
  !
 exit-address-family
!
! Classic EIGRP configuration (alternative)
! router eigrp 100
!  network 10.0.34.0 0.0.0.3
  network 10.0.45.0 0.0.0.3
!  network 3.3.3.3 0.0.0.0
!  no auto-summary
!
end
```
### **Step 2: EIGRP Configuration on R4**
```
! On R4
configure terminal
router eigrp 100
 address-family ipv4 unicast autonomous-system 100
  !
  network 10.0.34.0 0.0.0.3
  network 192.168.30.0 0.0.0.255
  network 4.4.4.4 0.0.0.0
  !
  metric weights 0 1 1 1 1 0
  maximum-paths 4
  !
  passive-interface default
  no passive-interface GigabitEthernet0/0
  no passive-interface GigabitEthernet0/1
  !
  af-interface GigabitEthernet0/2
   summary-address 192.168.0.0 255.255.0.0
  exit-af-interface
  !
 exit-address-family
end
```
### **Step 3: EIGRP Configuration on R5**
```
! On R5
configure terminal
router eigrp 100
 address-family ipv4 unicast autonomous-system 100
  !
  network 10.0.45.0 0.0.0.3
  network 5.5.5.5 0.0.0.0
  network 192.168.100.0 0.0.0.255
  !
  metric weights 0 1 1 1 1 0
  !
  passive-interface default
  no passive-interface GigabitEthernet0/0
  !
 exit-address-family
end
```
### **Step 4: Verify EIGRP Configuration**

**On R3:**
```
show ip eigrp neighbors
show ip eigrp topology
show ip eigrp interfaces
show ip route eigrp
```
**On R4:**
```
show ip eigrp neighbors detail
show ip eigrp topology all-links
show ip protocols | section eigrp
```
**On R5:**
```
show ip eigrp traffic
show ip route 192.168.30.0
```
## **Part 4: Route Redistribution**

### **Step 1: Configure Redistribution on R3**
```
! On R3 - Redistribute between OSPF and EIGRP
configure terminal
!
! Redistribute EIGRP into OSPF
router ospf 10
 redistribute eigrp 100 subnets metric 50 metric-type 1
!
! Redistribute OSPF into EIGRP
router eigrp 100
 address-family ipv4 unicast autonomous-system 100
  redistribute ospf 10 metric 10000 100 255 1 1500
 exit-address-family
!
! Configure route tags for tracking
route-map OSPF_TO_EIGRP permit 10
 match tag 100
 set metric 10000 100 255 1 1500
!
route-map EIGRP_TO_OSPF permit 10
 match tag 200
 set metric 50
 set metric-type type-1
!
end
```
### **Step 2: Configure Default Route Propagation**
```
! On R5 - Create default route and propagate via EIGRP
ip route 0.0.0.0 0.0.0.0 Null0
!
router eigrp 100
 address-family ipv4 unicast autonomous-system 100
  network 0.0.0.0
 exit-address-family
 ```
### **Step 3: Verify Redistribution**
```
show ip route
show ip route ospf
show ip route eigrp
show ip protocols
show route-map
```
## **Part 5: Advanced OSPF Features**

### **Step 1: Configure OSPF Stub Areas**
```
! Already configured Area 1 as stub on R1
! Configure totally stubby area
router ospf 10
 area 1 stub no-summary
 ```
## **Part 6: Advanced EIGRP Features**

### **Step 1: Configure EIGRP Stub**
```
! On R5 - Configure as EIGRP stub router
router eigrp 100
 address-family ipv4 unicast autonomous-system 100
  eigrp stub connected summary
 exit-address-family
 ```
### **Step 2: Configure EIGRP Load Balancing**
```
! On R4 - Configure unequal cost load balancing
router eigrp 100
 address-family ipv4 unicast autonomous-system 100
  variance 3
  maximum-paths 6
 exit-address-family
 ```
## **Part 7: Verification and Troubleshooting**

### **Comprehensive Verification Commands**

**1. OSPF Verification:**
```
! Verify neighbor relationships
show ip ospf neighbor
show ip ospf neighbor detail

! Verify OSPF interfaces
show ip ospf interface brief
show ip ospf interface GigabitEthernet0/0

! Verify OSPF database
show ip ospf database
show ip ospf database router
show ip ospf database external

! Verify OSPF routes
show ip route ospf
show ip route 192.168.10.0
```
**2. EIGRP Verification:**
```
! Verify EIGRP neighbors
show ip eigrp neighbors
show ip eigrp neighbors detail

! Verify EIGRP topology
show ip eigrp topology
show ip eigrp topology 192.168.30.0/24

! Verify EIGRP interfaces
show ip eigrp interfaces
show ip eigrp interfaces detail GigabitEthernet0/1

! Verify EIGRP traffic
show ip eigrp traffic
```
**3. Route Verification:**
```
! Check routing table
show ip route
show ip route connected
show ip route ospf
show ip route eigrp

! Check protocol status
show ip protocols
show ip protocols | begin ospf
show ip protocols | begin eigrp

! Check redistribution
show ip route redistributed
```
**4. Path Testing:**

cisco
```
! Traceroute tests
traceroute 192.168.30.10 source 192.168.10.1
traceroute 5.5.5.5
traceroute 1.1.1.1

! Ping tests with extended options
ping 192.168.20.1 source 192.168.10.1
ping 10.0.45.2 repeat 50 size 1500
```
### **Troubleshooting Exercises**

**Issue 1: OSPF Neighbors Not Forming**

cisco
```
! Debug OSPF adjacency
debug ip ospf adj
debug ip ospf hello
show ip ospf interface GigabitEthernet0/0
show run | section ospf

! Common issues:
! 1. Mismatched area numbers
! 2. Different subnet masks
! 3. Authentication mismatch
! 4. Hello/dead timer mismatch
```
**ssue 2: EIGRP Routes Missing**

cisco
```
! Debug EIGRP
debug eigrp packets
debug ip eigrp
show ip eigrp neighbors
show ip eigrp topology

! Common issues:
! 1. AS number mismatch
! 2. K-values mismatch
! 3. Network statements incorrect
! 4. Passive interface configured
```
**Issue 3: Redistribution Not Working**

cisco
```
show ip route
show ip protocols
debug ip routing
show route-map

! Check for:
! 1. Missing subnets keyword
! 2. Metric not configured
! 3. Route filters blocking routes
! 4. Administrative distance issues
```

## **Monitoring and Documentation**

### **Create Documentation Script**

cisco
```
! Script to document network configuration
enable
terminal length 0

! Create timestamp
show clock > network_documentation.txt

! Document OSPF
show ip ospf neighbor >> network_documentation.txt
show ip ospf interface brief >> network_documentation.txt
show ip ospf database summary >> network_documentation.txt

! Document EIGRP
show ip eigrp neighbors >> network_documentation.txt
show ip eigrp topology >> network_documentation.txt

! Document routing tables
show ip route >> network_documentation.txt
show ip route ospf >> network_documentation.txt
show ip route eigrp >> network_documentation.txt

! Document interfaces
show ip interface brief >> network_documentation.txt

terminal length 24
```
## **Lab Cleanup and Reset**

### **Reset Commands**

cisco
```
! Remove OSPF configuration
no router ospf 10

! Remove EIGRP configuration
no router eigrp 100

! Remove redistribution
no ip prefix-list DENY-LOOPBACKS
no route-map FILTER-EIGRP

! Reset interfaces
default interface GigabitEthernet0/0
default interface GigabitEthernet0/1

! Save cleared configuration
write memory
```
## **Learning Outcomes**

- ✅ **OSPF Configuration:** Multi-area OSPF with stub and NSSA areas

- ✅ **EIGRP Configuration:** Named mode EIGRP with advanced features

- ✅ **Route Redistribution:** Bi-directional redistribution with filtering

- ✅ **Path Selection:** Influence routing decisions with metrics and costs

- ✅ **Troubleshooting:** Comprehensive debugging of dynamic routing protocols

- ✅ **Verification:** Extensive show commands for protocol validation





























