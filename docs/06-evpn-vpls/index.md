# 06. EVPN VPLS

## 6.1 CE向けインタフェースの設定

!!! abstract "ゴール"

    PEルータにCE向けインタフェースの設定を行うこと

今回のEVPN-VPLSシナリオでは、下記のように全てのPE-CE間がシングルホーミング構成であることを想定します。

![](images/image01.png){ style="width:100%" }

まず、全てのPEルータ（C8102-1〜C8102-3）のCE（Windows or Ubuntu）向けインタフェースの設定を下記のように行います。

<pre class="cli"><code>interface HundredGigE0/0/0/2
 l2transport
 !
!</code></pre>

## 6.2 ブリッジドメイン設定

!!! abstract "ゴール"

    PEルータ（C8102-1〜C8102-3）にてブリッジドメインの設定を行うこと

全てのPEルータ（C8102-1〜C8102-3）に下記設定を行います。

<pre class="cli"><code>l2vpn
 bridge group BRIDGE
  bridge-domain BRIDGE_evi_100
   interface HundredGigE0/0/0/2
   !
   evi 100
   !</code></pre>

ブリッジドメインのstateが上がっていることを確認します。（C8102-1 ~ C8102-3）

<pre class="cli"><code>RP/0/RP0/CPU0:c8102-1#<strong>show l2vpn bridge-domain bd-name BRIDGE_evi_100 </strong>
Thu Dec 25 01:38:07.771 UTC
Legend: pp = Partially Programmed.
Bridge group: BRIDGE, bridge-domain: BRIDGE_evi_100, id: 0, <mark>state: up</mark>, ShgId: 0, MSTi: 0
  Aging: 300 s, MAC limit: 131072, Action: none, Notification: syslog
  Filter MAC addresses: 0
  <mark>ACs: 1 (1 up)</mark>, VFIs: 0, PWs: 0 (0 up), PBBs: 0 (0 up), VNIs: 0 (0 up)
  List of EVPNs:
    EVPN, state: up
  List of ACs:
    Hu0/0/0/2, state: up, Static MAC addresses: 0, MSTi: 5
  List of Access PWs:
  List of VFIs:
  List of Access VFIs:
RP/0/RP0/CPU0:c8102-1#</code></pre>

## 6.3 MP-BGP 設定

!!! abstract "ゴール"

    リモートPE間での BGP セッションの確立を確認すること

先ほどのVPWSとコントロールプレーン設定は共通となります。evpn-vpws ですでに設定済みのため、追加の設定は不要です。

PE ルータにてBGP 設定（C8102-1 ~ C8102-3）を確認します。

<pre class="cli"><code>RP/0/RP0/CPU0:c8102-1#<strong>show run router bgp</strong>
Tue Jan  6 01:56:45.996 UTC
router bgp 65000
 bgp router-id 1.1.1.1
 <mark>address-family l2vpn evpn</mark>
 !
 neighbor-group PEs
  remote-as 65000
  update-source Loopback0
  <mark>address-family l2vpn evpn</mark>
  !
 !
 neighbor 2.2.2.2
  use neighbor-group PEs
 !
 neighbor 3.3.3.3
  use neighbor-group PEs
 !
!

RP/0/RP0/CPU0:c8102-1#</code></pre>

PE間でBGPが上がっていることを確認します。（C8102-1 ~ C8102-3）

<pre class="cli"><code>RP/0/RP0/CPU0:c8102-1#<strong>show bgp l2vpn evpn neighbors brief</strong>
Wed Dec 24 08:08:32.313 UTC

Neighbor         Spk    AS  Description                         Up/Down  NBRState
2.2.2.2           0 65000                                      00:00:32 <mark>Established</mark> 
3.3.3.3           0 65000                                      00:00:11 <mark>Established</mark> 
RP/0/RP0/CPU0:c8102-1#</code></pre>

EVIの詳細を確認します。（C8102-1 ~ C8102-3）

Unicast/Multicast Labelを確認できます。また、今回はRDとRTは明示的に設定していないため、下記のパラメータが選択されていることがわかります。

- EVI BGP RD - $ROUTER-ID:$EVI
- EVI BGP RT - $AS:$EVI

Advertise MACを現在設定していないため、Noになっています。

<pre class="cli"><code>RP/0/RP0/CPU0:c8102-1#<strong>show evpn evi vpn-id 100 detail</strong>
Thu Dec 25 01:45:15.986 UTC

VPN-ID     Encap      Bridge Domain                Type               
---------- ---------- ---------------------------- -------------------
100        MPLS       BRIDGE_evi_100               EVPN               
   Stitching: Regular
   <mark>Unicast Label  : 24003</mark>
   <mark>Multicast Label: 24004</mark>
   Reroute Label: 0
   Flow Label: N
   Dynamic Flow Label: No
   Control-Word: Enabled
   E-Tree: Root
   Forward-class: 0
   <mark>Advertise MACs: No</mark>
   Advertise BVI MACs: No
   Aliasing: Enabled
   UUF: Enabled
   Re-origination: Enabled
   Multicast:
     IGMP-Snooping Proxy: No
     MLD-Snooping Proxy : No
   BGP Implicit Import: Enabled
   VRF Name: 
   Preferred Nexthop Mode: Off
   BVI Coupled Mode: No
   BVI Subnet Withheld: ipv4 No, ipv6 No
   L3VRF Label Mode: Per-VRF
   RD Config: none
   <mark>RD Auto  : (auto) 1.1.1.1:100</mark>
   <mark>RT Auto  : 65000:100</mark>
   Route Targets in Use           Type                 
   ------------------------------ ---------------------
   65000:100                      Import               
   65000:100                      Export               

RP/0/RP0/CPU0:c8102-1#</code></pre>

LFIBテーブルにもUnicast/Multicast Labelが存在していることを確認します。

（C8102-1 ~ C8102-3）

<pre class="cli"><code>RP/0/RP0/CPU0:c8102-1#<strong>show mpls forwarding</strong> 
Thu Dec 25 01:40:44.852 UTC
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    
------ ----------- ------------------ ------------ --------------- ------------
16002  16002       SR Pfx (idx 2)     Hu0/0/0/0    10.14.0.4       435470      
16003  16003       SR Pfx (idx 3)     Hu0/0/0/1    10.15.0.5       22886       
16004  Pop         SR Pfx (idx 4)     Hu0/0/0/0    10.14.0.4       0           
16005  Pop         SR Pfx (idx 5)     Hu0/0/0/1    10.15.0.5       0           
24000  Pop         SR Adj (idx 0)     Hu0/0/0/0    10.14.0.4       0           
24001  Pop         SR Adj (idx 0)     Hu0/0/0/1    10.15.0.5       0           
<mark>24003  Pop         EVPN:100 U         BD=0 E       point2point     0           </mark>
<mark>24004  Pop         EVPN:100 M         BD=0 EIM     point2point     0</mark>           
RP/0/RP0/CPU0:c8102-1#</code></pre>

しかし、EVPNによるMAC アドバタイズメントを有効にしていないため、下記のようにMAC学習がされていないことがわかります。

<pre class="cli"><code>RP/0/RP0/CPU0:c8102-1#<strong>show evpn evi mac</strong>
Thu Dec 25 01:52:41.944 UTC

VPN-ID     Encap      MAC address    IP address                               Nexthop                                 Label   
---------- ---------- -------------- ---------------------------------------- --------------------------------------- --------
RP/0/RP0/CPU0:c8102-1#</code></pre>

## 6.4 MAC Advertisement設定

!!! abstract "ゴール"

    MACアドバタイズメントを有効にし、相互にMAC学習をすること

PE ルータにてMACアドバタイズメントを有効化（C8102-1 ~ C8102-3）

<pre class="cli"><code>evpn
 evi 100
  advertise-mac
  !
 !
!</code></pre>

この設定により、RT-2のMAC advertisementのやりとりが開始され、学習したローカルおよびリモートの MAC アドレス及び対応するラベルの確認ができるようになります。

<pre class="cli"><code>RP/0/RP0/CPU0:c8102-1#show evpn evi mac
Tue Jan  6 05:27:41.011 UTC

VPN-ID     Encap      MAC address    IP address     Nexthop                 Label   
---------- ---------- -------------- -------------- ----------------------- --------
100        MPLS       0050.56b2.962a ::             2.2.2.2                 24003   
100        MPLS       0050.56b2.b9ca ::             HundredGigE0/0/0/2      24003   
100        MPLS       0050.56b2.ca86 ::             3.3.3.3                 24003   
RP/0/RP0/CPU0:c8102-1#</code></pre>

PE間でやりとりされるBGP EVPNルートは ” show bgp l2vpn evpn” コマンドで確認できます。（オプション）

<pre class="cli"><code>RP/0/RP0/CPU0:c8102-1#<strong>show bgp l2vpn evpn</strong>
Thu Dec 25 02:16:11.101 UTC
BGP router identifier 1.1.1.1, local AS number 65000
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0x0
BGP table nexthop route policy: 
BGP main routing table version 16
BGP NSR Initial initsync version 1 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0
BGP scan interval 60 secs

Status codes: s suppressed, d damped, h history, * valid, &gt; best
              i - internal, r RIB-failure, S stale, N Nexthop-discard
Origin codes: i - IGP, e - EGP, ? - incomplete
   Network            Next Hop            Metric LocPrf Weight Path
<mark>Route Distinguisher: 1.1.1.1:100</mark> (default for vrf BRIDGE_evi_100)
Route Distinguisher Version: 16
<mark>*&gt;i[2][0][48][0050.56aa.7173][0]/104</mark>
                      3.3.3.3                       100      0 i
<mark>*&gt;i[2][0][48][0050.56aa.91c5][0]/104</mark>
                      2.2.2.2                       100      0 i
<mark>*&gt; [2][0][48][0050.56aa.ba57][0]/104</mark>
                      0.0.0.0                                0 i
*&gt;i[2][0][48][3e0a.9933.6ea1][0]/104
                      2.2.2.2                       100      0 i
*&gt;i[2][0][48][aa5e.2697.43aa][0]/104
                      3.3.3.3                       100      0 i
*&gt; [2][0][48][ae29.15e2.d0f6][0]/104
                      0.0.0.0                                0 i
<mark>*&gt; [3][0][32][1.1.1.1]/80</mark>
                      0.0.0.0                                0 i
<mark>*&gt;i[3][0][32][2.2.2.2]/80</mark>
                      2.2.2.2                       100      0 i
<mark>*&gt;i[3][0][32][3.3.3.3]/80</mark>
                      3.3.3.3                       100      0 i
Route Distinguisher: 2.2.2.2:100
Route Distinguisher Version: 13
*&gt;i[2][0][48][0050.56aa.91c5][0]/104
                      2.2.2.2                       100      0 i
*&gt;i[2][0][48][3e0a.9933.6ea1][0]/104
                      2.2.2.2                       100      0 i
*&gt;i[3][0][32][2.2.2.2]/80
                      2.2.2.2                       100      0 i
Route Distinguisher: 3.3.3.3:100
Route Distinguisher Version: 15
*&gt;i[2][0][48][0050.56aa.7173][0]/104
                      3.3.3.3                       100      0 i
*&gt;i[2][0][48][aa5e.2697.43aa][0]/104
                      3.3.3.3                       100      0 i
*&gt;i[3][0][32][3.3.3.3]/80
                      3.3.3.3                       100      0 i

Processed 15 prefixes, 15 paths
RP/0/RP0/CPU0:c8102-1#</code></pre>

下記のように、RT-2 アドバタイズメントの詳細を確認することもできます。MACアドレス0050.56aa.7173はC8102-3（3.3.3.3）から受け取っており、Unicast Label 24003で転送されることがわかります。（オプション）

<pre class="cli"><code>RP/0/RP0/CPU0:c8102-1#<strong>show bgp l2vpn evpn rd 3.3.3.3:100 [2][0][48][0050.56aa.7173][0]/104</strong>
Thu Dec 25 02:20:00.634 UTC
BGP routing table entry for [2][0][48][0050.56aa.7173][0]/104, Route Distinguisher: 3.3.3.3:100
Versions:
  Process           bRIB/RIB   SendTblVer
  Speaker                 11           11
Last Modified: Dec 25 01:56:52.198 for 00:23:08
Paths: (1 available, best #1)
  Not advertised to any peer
  Path #1: Received by speaker 0
  Not advertised to any peer
  Local
    <mark>3.3.3.3 (metric 3) from 3.3.3.3 (3.3.3.3)</mark>
      <mark>Received Label 24003</mark> 
      Origin IGP, localpref 100, valid, internal, best, group-best, import-candidate, not-in-vrf
      Received Path ID 0, Local Path ID 1, version 11
      Extended community: Flags 0x10: SoO:3.3.3.3:100 RT:65000:100 
      EVPN ESI: 0000.0000.0000.0000.0000
RP/0/RP0/CPU0:c8102-1#</code></pre>

また、RT-3でやりとりされるBUMトラフィック用のラベル 24004が確認できます。

<pre class="cli"><code>RP/0/RP0/CPU0:c8102-1#<strong>show bgp l2vpn evpn rd 1.1.1.1:100 [3][0][32][3.3.3.3]/80</strong>
Thu Dec 25 03:37:51.754 UTC
BGP routing table entry for [3][0][32][3.3.3.3]/80, Route Distinguisher: 1.1.1.1:100
Versions:
  Process           bRIB/RIB   SendTblVer
  Speaker                  6            6
Last Modified: Dec 25 01:45:43.198 for 01:52:08
Paths: (1 available, best #1)
  Not advertised to any peer
  Path #1: Received by speaker 0
  Not advertised to any peer
  Local
    <mark>3.3.3.3 (metric 3) from 3.3.3.3 (3.3.3.3)</mark>
      Origin IGP, localpref 100, valid, internal, best, group-best, import-candidate, imported
      Received Path ID 0, Local Path ID 1, version 6
      Extended community: EVPN L2 ATTRS:0x04:0 RT:65000:100 
      PMSI: flags 0x00, type 6, <mark>label 24004</mark>, ID 0x03030303
      Source AFI: L2VPN EVPN, Source VRF: default, Source Route Distinguisher: 3.3.3.3:100
RP/0/RP0/CPU0:c8102-1#</code></pre>

## 6.5 疎通確認

!!! abstract "ゴール"

    WindowsとUbuntu間で相互に疎通確認を行うこと

![](images/image02.png){ style="width:100%" }

UbuntuからWindows（198.18.10.100）へのPing確認

![](images/image03.png){ style="width:100%" }

WindowsからUbuntu（198.18.10.127, 198.18.10.227）へのPing確認

![](images/image04.png){ style="width:100%" }

WindowsからUbuntu（198.18.10.127, 198.18.10.227）へのSSH

![](images/image05.png){ style="width:100%" }

## 6.6 パケットキャプチャ

!!! abstract "ゴール"

    EVPN VPLSのラベル転送の動作を確認すること

WindowsからUbuntu（198.18.10.127）へ継続的にPingを実施

![](images/image06.png){ style="width:100%" }

CML にて、C8102-1とC8102-４の間のリンクを右クリックし、Packet Captureを選択します。Start をクリックするとパケットキャプチャを開始します。Stop で停止します。

Source: 198.18.10.100, Destination: 198.18.10.127, Protocol: ICMP のパケットをクリックすると、詳細が表示されます。C8201-1からC8201-4へLabel: 24003 と 16002 が付与された上で転送されていることがわかります。

![](images/image07.png){ style="width:100%" }

次に、C8201-4からC8201-2間のリンクを右クリックし、Packet Captureを選択します。Start でパケットキャプチャを開始し、Source: 198.18.10.100, Destination: 198.18.10.127, Protocol: ICMP のパケットをクリックします。PHPがデフォルトで有効になっているため、宛先の一つ手前でTransport Label (16002) が外されて、C8201-4からC8201-2へ転送されていることがわかります。

![](images/image08.png){ style="width:100%" }

また、例えば、C8201-5からC8201-3間のリンクをパケットキャプチャしてみると、このリンクには198.18.10.127宛のトラフィックは流れないことがわかります。

![](images/image09.png){ style="width:100%" }

Ctrl+CでPingを止めます。

![](images/image10.png){ style="width:100%" }

ここで、BUMパケットがどのように転送されるのかを見てみます。存在しないIPアドレス（198.18.10.27）宛にWindowsからPingを行います。

![](images/image11.png){ style="width:100%" }

CML にて、C8102-1とC8102-４の間のリンクを右クリックし、Packet Captureを選択します。Start をクリックするとパケットキャプチャを開始します。Stop で停止します。

Protocol: ARP、Info: Who has 198.18.10.27? Tell 198.18.10.100のパケットをクリックすると、詳細が表示されます。C8201-1からC8201-4へ<strong>Multicast Label: 24004</strong>と Transport Label (C8102-2のNode SID) 16002 が付与された上で転送されていることがわかります。

![](images/image12.png){ style="width:100%" }

同様に、CML にて、C8102-1とC8102-５の間のリンクをキャプチャし、Protocol: ARP、Info: Who has 198.18.10.27? Tell 198.18.10.100のパケットをクリックすると、C8201-1からC8201-５へ<strong>Multicast Label: 24004</strong>と Transport Label (C8102-3のNode SID) 16003 が付与された上で転送されていることがわかります。

![](images/image13.png){ style="width:100%" }

Ctrl+CでPingを止めます。

![](images/image14.png){ style="width:100%" }

## 6.7 VPLS 設定の削除

!!! abstract "ゴール"

    VPWS設定を削除すること

全てのPEにてCE向けインタフェース、EVPN設定を削除します（C8102-1 ~ C8102-3）

<pre class="cli"><code>no interface HundredGigE0/0/0/2
no evpn
no l2vpn
no router bgp</code></pre>

EVPN-VPLSのシナリオは以上です。
