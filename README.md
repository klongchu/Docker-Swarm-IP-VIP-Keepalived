# Copilot Chat Conversation Export: Random joke generator using an API

**User:** @klongchu
**Thread URL:** https://github.com/copilot/c/6b7762fe-c2b1-4b67-9238-b77ef6f85b47

## @klongchu

docker swarm ทํา IP VIP + keepalived 
ทำแบบไหน

## Copilot

การทำ High Availability (HA) ให้กับ Docker Swarm โดยใช้ IP VIP (Virtual IP) คู่กับ Keepalived มีจุดประสงค์เพื่อให้เรามี IP กลาง (VIP) เพียงเบอร์เดียวที่ลูกค้าหรือ Service อื่นวิ่งเข้ามาหาได้ตลอดเวลา แม้ว่า Node หลักจะล่มไป IP นี้ก็จะย้ายไปเกาะที่ Node สำรองให้อัตโนมัติครับ

นี่คือขั้นตอนการทำแบบ Step-by-Step ครับ

### 1. เตรียมความพร้อม (Prerequisites)
*   เครื่อง Server (หรือ VM) **อย่างน้อย 3 เครื่อง** เพื่อให้ Swarm ทำงานสมบูรณ์ (Quorum) และป้องกัน Split-brain
*   OS: Linux (เช่น Ubuntu หรือ CentOS)
*   Network: อยู่ในวงเดียวกัน และต้องมี **IP ว่าง 1 เบอร์** เพื่อใช้เป็น VIP

สมมติสถานการณ์:
*   **Node 1:** 192.168.1.11 (Master)
*   **Node 2:** 192.168.1.12 (Backup)
*   **Node 3:** 192.168.1.13 (Backup)
*   **VIP (IP กลาง):** 192.168.1.99

---

### 2. ติดตั้ง Docker Swarm
ติดตั้ง Docker และสร้าง Cluster ให้เรียบร้อยก่อนครับ

1.  **ติดตั้ง Docker** ทุกเครื่อง
2.  **Init Swarm** ที่ Node 1:
    ```bash
    docker swarm init --advertise-addr 192.168.1.11
    ```
3.  **Join Swarm** ที่ Node 2 และ 3 (ใช้ token ที่ได้จากข้อบน):
    ```bash
    docker swarm join --token <TOKEN> 192.168.1.11:2377
    ```

---

### 3. ติดตั้งและตั้งค่า Keepalived
เราจะติดตั้ง Keepalived ลงบนเครื่อง Host โดยตรง (ไม่ใช่ใน Container) เพื่อความเสถียรที่สุดในการคุม Network Interface

1.  **ติดตั้ง Keepalived** (ทำทุกเครื่อง):
    ```bash
    sudo apt-get update
    sudo apt-get install keepalived -y
    ```

2.  **ตั้งค่า Config** (แก้ไขไฟล์ `/etc/keepalived/keepalived.conf`):

    **เครื่อง Master (Node 1)**
    *   `state`: MASTER
    *   `priority`: 101 (ค่าสูงกว่าจะเป็น Master)
    *   `interface`: ชื่อการ์ดแลนของเครื่อง (เช็คด้วยคำสั่ง `ip addr` เช่น eth0, ens160)

    ```conf name=/etc/keepalived/keepalived.conf
    global_defs {
        router_id LVS_DEVEL
    }

    vrrp_script chk_docker {
        script "/usr/bin/killall -0 dockerd"  # เช็คว่า Docker ยังทำงานอยู่ไหม
        interval 2
        weight 2
    }

    vrrp_instance VI_1 {
        state MASTER
        interface ens7           # <--- แก้เป็นชื่อ Interface ของคุณ
        virtual_router_id 51     # ต้องตรงกันทุกเครื่อง
        priority 101             # ค่าสูงที่สุด
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass mysecret   # ตั้งรหัสผ่านให้ตรงกัน
        }
        virtual_ipaddress {
            192.168.1.99         # <--- ใส่ IP VIP ที่ต้องการ
        }
        track_script {
            chk_docker
        }
    }
    ```

    **เครื่อง Backup (Node 2 และ 3)**
    *   `state`: BACKUP
    *   `priority`: 100 (ลดหลั่นลงมา)

    ```conf name=/etc/keepalived/keepalived.conf

    global_defs {
        router_id LVS_DEVEL
    }

    vrrp_script chk_docker {
        script "/usr/bin/killall -0 dockerd"  # เช็คว่า Docker ยังทำงานอยู่ไหม
        interval 2
        weight 2
    }

    vrrp_instance VI_1 {
        state BACKUP
        interface ens7           # <--- แก้เป็นชื่อ Interface ของคุณ
        virtual_router_id 51     # ต้องตรงกันทุกเครื่อง
        priority 100             # ค่าสูงที่สุด
        advert_int 1
        authentication {
            auth_type PASS
            auth_pass mysecret   # ตั้งรหัสผ่านให้ตรงกัน
        }
        virtual_ipaddress {
            192.168.1.99         # <--- ใส่ IP VIP ที่ต้องการ
        }
        track_script {
            chk_docker
        }
    }
    ```

3.  **Start Keepalived** (ทำทุกเครื่อง):
    ```bash
    sudo systemctl restart keepalived
    sudo systemctl enable keepalived
    ```

---

### 4. ทดสอบการทำงาน
1.  **เช็ค VIP:** ที่ Node 1 ลองพิมพ์ `ip addr show eth0` คุณควรจะเห็น IP `192.168.1.99` ปรากฏขึ้นมา
2.  **ทดสอบ Deploy App:**
    ```bash
    docker service create --name web --publish 80:80 nginx
    ```
    จากนั้นเข้าผ่าน Browser หรือ curl ไปที่ `http://192.168.1.99` (เข้าผ่าน VIP) ควรจะเจอหน้า Nginx
3.  **ทดสอบ Failover:**
    *   ลองปิดเครื่อง Node 1 หรือ stop service keepalived (`sudo systemctl stop keepalived`)
    *   ไปเช็คที่ Node 2 หรือ 3 ดูว่า IP `192.168.1.99` ย้ายไปเกาะที่เครื่องนั้นหรือไม่
    *   Service Nginx ยังควรเข้าได้อยู่ (แต่อาจจะกระตุกนิดนึงตอนสลับ)

### เทคนิคเพิ่มเติม
*   **Check Script:** ใน Config ข้างบนผมใส่ `vrrp_script chk_docker` ไว้ด้วย เพื่อให้ Keepalived ย้าย VIP หนีทันทีถ้า Docker Service ในเครื่องนั้นตาย (ไม่ใช่แค่เครื่องดับ)
*   **Firewall:** อย่าลืมเปิด Firewall ให้โปรโตคอล **VRRP** สื่อสารกันได้ (Multicast หรือ Unicast) ไม่งั้นเครื่อง Backup จะมองไม่เห็น Master และแย่งกันเป็น Master (Split-brain)

