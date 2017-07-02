# Traffic flow của mô hình Distributed Virtual Router (DVR)
# Mục lục
### [1. Đồ hình OpenStack cho mô hình DVR](#topo)
### [2. Kịch bản 1: Hai instances cùng network, nằm trên cùng Compute node](#sc1)
### [3. Kịch bản 2: Hai instances cùng network, nằm trên hai Compute node riêng biệt](#sc2)
### [4. Kịch bản 3: Hai instances khác network nhưng cùng chung router, trên cùng một Compute node](#sc3)
### [5. Kịch bản 4: Hai instances khác network nhưng cùng chung router, trên hai Compute node riêng biệt](#sc4)
### [6. Kịch bản 5: Instance trên Compute node truyền thông với internet sử dụng Floating IP](#sc5)
### [7. Kịch bản 6: Instance trên Compute node truyền thông với internet qua SNAT router trên Controller](#sc6)
### [8. Tham khảo](#ref)

---

### <a name="topo"></a> 1. Đồ hình OpenStack cho mô hình DVR


### <a name="sc1"></a> 2. Kịch bản 1: Hai instances cùng network, nằm trên cùng Compute node
- Trong ví dụ ở đây, hai instance __blue-1__ và __blue-2__ nằm trên cùng Compute Node 2 và cùng thuộc subnet __sub-private-net__ của network __private-net__.
- Trên các Compute node, tạo file openrc lưu giữ các biến môi trường cần cho việc xác thực và ủy quyền. Ví dụ ở đây sử dụng tạo file openrc trên thư mục người dùng root là: `/root`
- Trên Compute node 2, thiết lập biến môi trường: `source ~/admin-openrc`
    - Lấy thông tin các interface của các instances __blue-1__ và __blue-2__:

    ```sh
    nova interface-list blue-1
    +------------+--------------------------------------+--------------------------------------+--------------+-------------------+
    | Port State | Port ID                              | Net ID                               | IP addresses | MAC Addr          |
    +------------+--------------------------------------+--------------------------------------+--------------+-------------------+
    | ACTIVE     | 20353db4-a6c8-4c2d-8e23-4774cd17f5e8 | 639fe324-0f1d-492e-bf48-8c44147874d4 | 172.17.0.10  | fa:16:3e:02:c1:05 |
    +------------+--------------------------------------+--------------------------------------+--------------+-------------------+
    nova interface-list blue-2
    +------------+--------------------------------------+--------------------------------------+--------------+-------------------+
    | Port State | Port ID                              | Net ID                               | IP addresses | MAC Addr          |
    +------------+--------------------------------------+--------------------------------------+--------------+-------------------+
    | ACTIVE     | cb7ca0f0-7f55-450f-a771-33912633e51c | 639fe324-0f1d-492e-bf48-8c44147874d4 | 172.17.0.12  | fa:16:3e:3b:46:62 |
    +------------+--------------------------------------+--------------------------------------+--------------+-------------------+
    ```

    - Liệt kê các Linux bridge trên Compute Node 2:

    ```sh
    brctl show
    bridge name     bridge id               STP enabled     interfaces
    qbr05fd4103-3f          8000.82203d3e4f90       no              qvb05fd4103-3f
    qbr20353db4-a6          8000.aaa5d0a40d63       no              qvb20353db4-a6
                                                            tap20353db4-a6
    qbrcb7ca0f0-7f          8000.2afaefcc9cfa       no              qvbcb7ca0f0-7f
                                                            tapcb7ca0f0-7f
    virbr0          8000.525400d0ba3c       yes             virbr0-nic
    ```

    - Từ kết quả các câu lệnh trên có thể thấy để ý thấy rằng: 
        - Intance __blue-1__ kết nối với bridge __qbr20353db4-a6__ (chú ý prefix của interface ID của các instance sử dụng làm postfix trong tên của bridge kết nối với instance đó)
        - Intance __blue-2__ kết nối với bridge __qbrcb7ca0f0-7f__
    - Để ý thấy rằng mỗi bridge đều có __tap__ interface kết nối virtual NIC của instance tương ứng, còn __qvb__ interface của linux bridge cùng với __qvo__ của integration bridge __br-int__ tạo thành cặp __veth-pair__ sử dụng để chuyển tiếp gói tin từ instance qua linux bridge tới br-int.
    - Các linux bridge trên được neutron sử dụng để thiết lập __iptables rules__ lên lưu lượng ra vào instance theo security group đã thiết lập lúc khởi tạo instance. Các rules này được áp dụng lên __tap__ interface. Về mặt lý tưởng thì instance hoàn toàn có thể kết nối với integration bridge là __br-int__. Tuy nhiên mô hình DVR ở đây sử dụng Open vSwitch agent, mà __iptables rules__ không áp dụng được với các tap interface trên Open vSwitch bridge nên buộc phải sử dụng một linux bridge trung gian rồi áp dụng iptables rule lên tap interface của bridge đó.
    - Ta thực hiện ping từ __blue-1__ (172.17.0.10) sang __blue-2__ (172.17.0.12). Trong khi đó trên Compute node 2 thực hiện bắt gói tin qua __tap20353db4-a6__ interface kết nối với __blue-1__:

    ```sh
    tcpdump -i tap20353db4-a6 -n -e icmp 
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on tap20353db4-a6, link-type EN10MB (Ethernet), capture size 262144 bytes
    08:53:13.642012 fa:16:3e:02:c1:05 > fa:16:3e:3b:46:62, ethertype IPv4 (0x0800), length 98: 172.17.0.10 > 172.17.0.12: ICMP echo request, id 62464, seq 346, length 64
    08:53:13.643496 fa:16:3e:3b:46:62 > fa:16:3e:02:c1:05, ethertype IPv4 (0x0800), length 98: 172.17.0.12 > 172.17.0.10: ICMP echo reply, id 62464, seq 346, length 64
    08:53:14.642851 fa:16:3e:02:c1:05 > fa:16:3e:3b:46:62, ethertype IPv4 (0x0800), length 98: 172.17.0.10 > 172.17.0.12: ICMP echo request, id 62464, seq 347, length 64
    08:53:14.643906 fa:16:3e:3b:46:62 > fa:16:3e:02:c1:05, ethertype IPv4 (0x0800), length 98: 172.17.0.12 > 172.17.0.10: ICMP echo reply, id 62464, seq 347, length 64
    08:53:15.643785 fa:16:3e:02:c1:05 > fa:16:3e:3b:46:62, ethertype IPv4 (0x0800), length 98: 172.17.0.10 > 172.17.0.12: ICMP echo request, id 62464, seq 348, length 64
    08:53:15.644613 fa:16:3e:3b:46:62 > fa:16:3e:02:c1:05, ethertype IPv4 (0x0800), length 98: 172.17.0.12 > 172.17.0.10: ICMP echo reply, id 62464, seq 348, length 64
    08:53:16.644727 fa:16:3e:02:c1:05 > fa:16:3e:3b:46:62, ethertype IPv4 (0x0800), length 98: 172.17.0.10 > 172.17.0.12: ICMP echo request, id 62464, seq 349, length 64
    08:53:16.645764 fa:16:3e:3b:46:62 > fa:16:3e:02:c1:05, ethertype IPv4 (0x0800), length 98: 172.17.0.12 > 172.17.0.10: ICMP echo reply, id 62464, seq 349, length 64
    ```

    - Kiểm tra port __qvo__ của br-int kết nối với port __qvb__ của linux bridge kết nối với instance __blue-1__:

    ```sh
    root@compute2:~# ovs-vsctl show | grep 20353db4-a6
            Port "qvo20353db4-a6"
                Interface "qvo20353db4-a6"
    ```
    
    - Thực hiện theo dõi các flow entries của các table trên pipeline của bridge __br-int__, trước hết là table 0 - bảng đầu tiên trên pipeline mà gói tin phải đi qua sau khi được gửi qua __veth-pair__ giữa linux brige __qbr20353db4-a6__ và __br-int__:

    ```sh
    watch -n.5 "ovs-ofctl dump-flows br-int table=0 | grep -v n_packets=0"
    Every 0.5s: ovs-ofctl dump-flows br-int table=0 | grep -v n_packets=0                                                                                                                                             Sat Jul  1 09:05:31 2017

    NXST_FLOW reply (xid=0x4):
     cookie=0xa1560e735d41e2d1, duration=228247.683s, table=0, n_packets=74, n_bytes=3108, idle_age=8, hard_age=65534, priority=10,arp,in_port=2 actions=resubmit(,24)
     cookie=0xa1560e735d41e2d1, duration=3411.323s, table=0, n_packets=3, n_bytes=126, idle_age=2812, priority=10,arp,in_port=8 actions=resubmit(,24)
     cookie=0xa1560e735d41e2d1, duration=2277.460s, table=0, n_packets=71, n_bytes=2982, idle_age=8, priority=10,arp,in_port=9 actions=resubmit(,24)
     cookie=0xa1560e735d41e2d1, duration=228337.625s, table=0, n_packets=194, n_bytes=11936, idle_age=65534, hard_age=65534, priority=2,in_port=5 actions=drop
     cookie=0xa1560e735d41e2d1, duration=228247.688s, table=0, n_packets=2341, n_bytes=308542, idle_age=0, hard_age=65534, priority=9,in_port=2 actions=resubmit(,25)
     cookie=0xa1560e735d41e2d1, duration=3411.327s, table=0, n_packets=62, n_bytes=7857, idle_age=2812, priority=9,in_port=8 actions=resubmit(,25)
     cookie=0xa1560e735d41e2d1, duration=2277.464s, table=0, n_packets=1161, n_bytes=117080, idle_age=0, priority=9,in_port=9 actions=resubmit(,25)
     cookie=0xa1560e735d41e2d1, duration=228181.986s, table=0, n_packets=233790, n_bytes=14227969, idle_age=0, hard_age=65534, priority=3,in_port=5,vlan_tci=0x0000/0x1fff actions=mod_vlan_vid:3,NORMAL
     cookie=0xa1560e735d41e2d1, duration=228338.062s, table=0, n_packets=3181, n_bytes=361247, idle_age=0, hard_age=65534, priority=0 actions=NORMAL
    ```

    Để ý flow entries cuối cùng của table 0, ta thấy action được áp dụng với gói tin là __NORMAL__, nghĩa là bridge br-int sẽ thực hiện chuyển tiếp frame lớp 2 bình thường 
 
    - Kiểm tra bảng chuyển tiếp trên __br-int__ để kiểm tra mac nguồn và mac đích trong kết quả bắt gói của tcpdump tương ứng với cổng nào của bridge __br-int__:

    ```sh
    ovs-appctl fdb/show br-int
     port  VLAN  MAC                Age
       5     3  1c:98:ec:20:96:7e   72
       4     1  fa:16:3e:63:e4:9b    1
       7     3  fa:16:3e:13:78:88    1
       9     1  fa:16:3e:3b:46:62    1 
       2     1  fa:16:3e:02:c1:05    1
       5     3  52:e1:fe:f4:5e:47    1
       5     3  52:54:00:a0:f5:9e    1
    ```

    Như vậy MAC __fa:16:3e:02:c1:05__ là địa chỉ MAC của instance __blue-1__ tương ứng kết nối với port 2 của __br-int__. Trong khi đó __fa:16:3e:3b:46:62__ là địa chỉ MAC của __blue-2__ tương ứng với port 9, nghĩa là gói tin ICMP từ __blue-1__ sẽ đi ra khỏi openflow port 9 trên __br-int__.

    - Kiểm tra số hiệu port tương ứng với port name trên bridge __br-int__:

    ```sh
    ovs-ofctl show br-int 
    OFPT_FEATURES_REPLY (xid=0x2): dpid:00003e40b4fbfe43
    n_tables:254, n_buffers:256
    capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
    actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
     1(patch-tun): addr:e2:6b:31:fd:44:3b
         config:     0
         state:      0
         speed: 0 Mbps now, 0 Mbps max
     2(qvo20353db4-a6): addr:a2:e5:75:e2:ca:10
         config:     0
         state:      0
         current:    10GB-FD COPPER
         speed: 10000 Mbps now, 0 Mbps max
     3(tap21d166d0-57): addr:00:00:00:00:0e:00
         config:     PORT_DOWN
         state:      LINK_DOWN
         speed: 0 Mbps now, 0 Mbps max
     4(qr-069f3138-b4): addr:00:00:00:00:0e:00
         config:     PORT_DOWN
         state:      LINK_DOWN
         speed: 0 Mbps now, 0 Mbps max
     5(int-br-ex): addr:0e:27:db:6f:90:d0
         config:     0
         state:      0
         speed: 0 Mbps now, 0 Mbps max
     6(qr-6e4c9476-70): addr:00:00:00:00:0e:00
         config:     PORT_DOWN
         state:      LINK_DOWN
         speed: 0 Mbps now, 0 Mbps max
     7(fg-16b8adc7-7e): addr:00:00:00:00:0e:00
         config:     PORT_DOWN
         state:      LINK_DOWN
         speed: 0 Mbps now, 0 Mbps max
     8(qvo05fd4103-3f): addr:ce:23:b8:df:09:cc
         config:     0
         state:      0
         current:    10GB-FD COPPER
         speed: 10000 Mbps now, 0 Mbps max
     9(qvocb7ca0f0-7f): addr:a2:77:a2:77:0b:0d <---- pot 9 map voi qvocb7ca0f0-7f, port nay lien ket toi VM blue-2
         config:     0
         state:      0
         current:    10GB-FD COPPER
         speed: 10000 Mbps now, 0 Mbps max
     LOCAL(br-int): addr:3e:40:b4:fb:fe:43
         config:     PORT_DOWN
         state:      LINK_DOWN
         speed: 0 Mbps now, 0 Mbps max
    OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
    ```

    Như vậy port 9 tương ứng với __qvocb7ca0f0-7f__, port này liên kết với instance __blue-2__. 

    - Port __qvb__ của linux bridge __qbr__ dưới đây là một cặp __veth-pair với __qvocb7ca0f0-7f__, trong khi tap port bên dưới là port kết nối linux bridge với virtual NIC của máy ảo blue-2:

    ```sh
    qbrcb7ca0f0-7f          8000.2afaefcc9cfa       no              qvbcb7ca0f0-7f
                                                            tapcb7ca0f0-7f
    ```

- Như vậy traffic flow trong kịch bản này sẽ như sau: Gói tin từ __blue-1__ tới linux bridge tương ứng. Tại đây iptables rules tương ứng với security group thiết lập cho __blue-1__ khi khởi tạo sẽ được áp dụng để kiểm soát lưu lượng ra vào __blue-1__. Tiếp đó, lưu lượng được chuyển tiếp tới __br-int__ trên VLAN 1 và được chèn thêm VLAN header với VLAN ID 1. Gói tin này được chuyển tiếp tới port VLAN 1 tương ứng với instance blue-2 (standard l2 MAC forwarding). Gói tin sẽ được gỡ bỏ VLAN header trước đi khi qua port __qvo__ này tới linux bridge áp dụng iptables rules tương ứng với __blue-2__. Cuối cùng gói tin rời khỏi tap interface của linux bridge và tới virtual NIC của __blue-2__. Gói tin ICMP reply sẽ có tiến trình tương ứng nhưng ngược chiều.

### <a name="sc2"></a> 3. Kịch bản 2: Hai instances cùng network, nằm trên hai Compute node riêng biệt


### <a name="sc3"></a> 4. Kịch bản 3: Hai instances khác network nhưng cùng chung router, trên cùng một Compute node


### <a name="sc4"></a> 5. Kịch bản 4: Hai instances khác network nhưng cùng chung router, trên hai Compute node riêng biệt


### <a name="sc5"></a> 6. Kịch bản 5: Instance trên Compute node truyền thông với internet sử dụng Floating IP


### <a name="sc6"></a> 7. Kịch bản 6: Instance trên Compute node truyền thông với internet qua SNAT router trên Controller


### <a name="ref"></a> 8. Tham khảo


