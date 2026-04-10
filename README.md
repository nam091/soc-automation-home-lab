# SOC Automation Lab - Note Home Lab Của Mình

**Uruc Tarim** (mình fork về customize theo lab này)

**Lời cảm ơn:** Note này mình học theo rất nhiều từ kênh YouTube DFIR (mydfir). Video của anh ấy chi tiết và thực tế lắm, giúp mình hiểu rõ cách triển khai từng thành phần trong SOC Automation Lab. Anh em nên xem hết series đó trước khi làm theo note này nhé.

## 1. Giới thiệu

### 1.1 Tổng quan
Anh em ơi, project SOC Automation này mình làm để xây dựng một workflow tự động cho Security Operations Center (SOC). Nó giúp streamline việc monitor event, alerting và incident response luôn. Mình dùng bộ công cụ open-source mạnh như Wazuh, Shuffle, và TheHive. Cụ thể là Windows 10 client chạy Sysmon generate event chi tiết, Wazuh thu thập + phân tích + alert, Shuffle tự động hóa workflow, TheHive quản lý case và response.

![SOC Automation Diagram](./SOC_Automation_Diagram.png)

### 1.2 Mục đích và những gì mình học được
- **Tự động thu thập & phân tích event:** Mình setup để event security được collect real-time, giảm tối đa việc làm tay. Mẹo là dùng Sysmon + Wazuh thì detect threat nhanh và chủ động hơn nhiều.
- **Tối ưu alerting:** Tự động sinh alert và forward luôn, giảm thời gian response, tránh miss incident quan trọng. Mình thấy cái này giảm workload SOC analyst đi đáng kể.
- **Tăng khả năng incident response:** Tự động trigger action khi có incident, phản ứng nhanh, nhất quán và hiệu quả hơn.
- **Làm SOC hiệu quả hơn:** Automate mấy task routine để anh em tập trung vào phân tích sâu và cải tiến quy trình.

## 2. Những thứ cần chuẩn bị

### 2.1 Yêu cầu phần cứng
- Máy host mạnh đủ để chạy nhiều VM cùng lúc (mình recommend ít nhất 16GB RAM, CPU 6-8 cores, SSD nhanh).
- Đủ tài nguyên cho các VM chạy mượt, đặc biệt Wazuh và TheHive hay ngốn RAM.

### 2.2 Phần mềm cần có
- **VMware Workstation/Fusion:** Mình dùng cái này vì ổn định, dễ snapshot để test.
- **Windows 10:** Làm client generate event thực tế.
- **Ubuntu 22.04:** Mình chọn bản này vì ổn định, dễ deploy Wazuh và TheHive.
- **Sysmon:** Tool hay nhất để log chi tiết trên Windows, mình không thể thiếu.

### 2.3 Công cụ & Platform
- **Wazuh:** Trung tâm thu thập, phân tích, alerting. Enterprise-grade mà miễn phí.
- **Shuffle:** SOAR open-source, linh hoạt, mình dùng để automate workflow.
- **TheHive:** Quản lý case, incident response platform rất mạnh cho SOC.
- **VirusTotal:** Check hash nhanh, mình integrate vào workflow luôn.
- **Cloud hoặc VM thêm:** Mình deploy Wazuh và TheHive lên DigitalOcean Droplet cho nhanh, anh em có thể dùng VM local cũng được.

### 2.4 Kiến thức nền
- Biết cơ bản VM (snapshot, networking).
- Linux command line (apt, systemctl, nano...).
- Hiểu khái niệm SOC, event logging, incident response. Nếu chưa thì xem video DFIR trước.

## 3. Setup (Mình làm theo thứ tự này)

### 3.1 Bước 1: Cài Windows 10 + Sysmon
**3.1.1 Cài Windows 10 trên VMware:**
Mình tạo VM Windows 10 trước. 

![Windows 10 Installation](./images/Pasted image 20240603131110.png)

**3.1.2 Tải Sysmon:**
Tải từ Microsoft Sysinternals.

![Sysmon Installation](./images/Pasted image 20240603131150.png)

**3.1.3 Tải config Sysmon Modular từ GitHub của olafhartong:**
Config này hay lắm, detect được nhiều threat.

![Sysmon Modular Config](./images/Pasted image 20240603131815.png)
![Sysmon Modular Config Files](./images/Pasted image 20240603132002.png)

**3.1.4 Giải nén Sysmon và mở PowerShell Administrator, cd vào thư mục:**
Lưu ý quyền admin rất quan trọng.

![Extract Sysmon Zip](./images/Pasted image 20240603133020.png)

**3.1.5 Copy file config sysmonconfig.xml vào cùng thư mục.**

**3.1.6 Check xem Sysmon đã cài chưa (Services + Event Viewer):**

![Check Sysmon Installation](./images/Pasted image 20240603133433.png)

**3.1.7 Chưa có thì cài bằng lệnh:**
```
.\Sysmon64.exe -i .\sysmonconfig.xml
```

![Install Sysmon](./images/Pasted image 20240603154817.png)

**3.1.8 Verify lại sau khi cài:**

![Verify Sysmon Installation](./images/Pasted image 20240603154953.png)

Xong bước này Windows client của mình sẵn sàng. Tiếp theo setup Wazuh.

### 3.2 Bước 2: Setup Wazuh Server
**3.2.1 Tạo Droplet trên DigitalOcean (hoặc VM local cũng được):**
Mình dùng cloud cho nhanh.

![Create Droplet](./images/Pasted image 20240603215218.png)

Chọn Ubuntu 22.04:

![Select Ubuntu](./images/Pasted image 20240603220120.png)

Đặt tên "Wazuh", tạo root password:

![Create Wazuh Droplet](./images/Pasted image 20240603220521.png)

**3.2.2 Setup Firewall:**
Quan trọng lắm anh em, chỉ allow IP của mình thôi để tránh scan.

![Create Firewall](./images/Pasted image 20240603220742.png)
![Set Inbound Rules](./images/Pasted image 20240603220920.png)
![Apply Firewall](./images/Pasted image 20240603221926.png)
![Firewall Protection](./images/Pasted image 20240603222113.png)

**3.2.3 SSH vào server:**
Dùng Droplet Console hoặc SSH client.

![Launch Droplet Console](./images/Pasted image 20240603223020.png)

**3.2.4 Update system:**
```
sudo apt-get update && sudo apt-get upgrade
```

**3.2.5 Cài Wazuh:**
```
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

Quá trình cài:

![Wazuh Installation](./images/Pasted image 20240603224933.png)

Ghi lại password admin (rất quan trọng):

**3.2.6 Truy cập web interface:**
Dùng https://IP-cua-Wazuh, bypass cert warning.

![Wazuh Login](./images/Pasted image 20240603225355.png)
![Wazuh Login Continue](./images/Pasted image 20240603225424.png)
![Wazuh Dashboard](./images/Pasted image 20240603225621.png)

Xong Wazuh. Tiếp tục TheHive.

### 3.3 Bước 3: Cài TheHive
**3.3.1 Tạo Droplet mới cho TheHive:**
Tương tự, Ubuntu 22.04, attach firewall cũ.

![Create TheHive Droplet](./images/Pasted image 20240603230036.png)

**3.3.2 Cài dependencies:**
```
apt install wget gnupg apt-transport-https git ca-certificates ca-certificates-java curl software-properties-common python3-pip lsb-release
```

![Install Dependencies](./images/Pasted image 20240603230509.png)

**3.3.3 Cài Java 11 (Amazon Corretto):**
Mình paste lệnh đầy đủ ở note này luôn cho anh em copy.

**3.3.4 Cài Cassandra:**
Database cho TheHive.

**3.3.5 Cài Elasticsearch:**
Dùng version 7.x.

**3.3.6 (Optional) Tune JVM cho Elasticsearch:**
Tạo file jvm.options với -Xms2g -Xmx2g, tùy resource máy anh em.

**3.3.7 Cài TheHive:**
Lệnh official từ strangebee.

Default credential:
- Username: admin@thehive.local
- Password: secret

![TheHive Login](./images/Pasted image 20240603231319.png)

### 3.4 Bước 4: Configure TheHive + Wazuh
**3.4.1 Config Cassandra:**
Edit cassandra.yaml, set listen_address, rpc_address, seeds thành public IP của TheHive. 

![Cassandra Configuration](./images/Pasted image 20240603231723.png)
![Listen Address](./images/Pasted image 20240603232121.png)
![Seed Provider](./images/Pasted image 20240603232508.png)

Restart và clear data cũ:
```
systemctl stop cassandra.service
rm -rf /var/lib/cassandra/*
systemctl start cassandra.service
systemctl status cassandra.service
```

![Cassandra Service Status](./images/Pasted image 20240603232813.png)

**3.4.2 Config Elasticsearch:**
Edit elasticsearch.yml, set network.host = IP, cluster.initial_master_nodes...

![Elasticsearch Configuration](./images/Pasted image 20240603233522.png)
![Elasticsearch Service Status](./images/Pasted image 20240603233935.png)

**3.4.3 Config TheHive:**
Chown permission trước:
```
chown -R thehive:thehive /opt/thp
```

Edit /etc/thehive/application.conf, set IP, cluster.name, baseUrl...

![TheHive Configuration](./images/Pasted image 20240603234256.png)
![TheHive Directory Permissions](./images/Pasted image 20240603234450.png)
![Change TheHive Directory Permissions](./images/Pasted image 20240603235441.png)

Start service và check status.

![TheHive Service Status](./images/Pasted image 20240603235616.png)

**Lưu ý quan trọng:** Nếu vào không được thì check 3 service Cassandra, Elasticsearch, TheHive phải chạy hết. Mình hay bị quên cái này.

Truy cập http://IP:9000/login

![TheHive Login](./images/Pasted image 20240603235840.png)
![TheHive Dashboard](./images/Pasted image 20240604000101.png)

### 3.5 Bước 5: Add Windows Agent vào Wazuh
Vào Wazuh dashboard > Add agent > Windows > điền IP server.

![Add Wazuh Agent](./images/Pasted image 20240604000526.png)

Copy command chạy trên Windows client.

![Wazuh Agent Installation](./images/Pasted image 20240604002346.png)

Start service:

![Wazuh Agent Service](./images/Pasted image 20240604002550.png)
![Wazuh Agent Service Start](./images/Pasted image 20240604002624.png)

Check agent active:

![Wazuh Agent Connected](./images/Pasted image 20240604002744.png)
![Wazuh Agent Status](./images/Pasted image 20240604002929.png)

## 4. Generate Telemetry & Custom Alert

### 4.1 Forward Sysmon event sang Wazuh
Edit ossec.conf trên Windows, thêm localfile cho Sysmon event log.

![Ossec Configuration](./images/Pasted image 20240604144648.png)
![Sysmon Event Log](./images/Pasted image 20240604150516.png)
![Ossec Sysmon Configuration](./images/Pasted image 20240604150556.png)

Restart agent sau khi edit (mình hay quên bước này).

![Restart Wazuh Agent Service](./images/Pasted image 20240604151539.png)

Check event trong Wazuh:

![Search Sysmon Events](./images/Pasted image 20240604152334.png)

### 4.2 Generate Mimikatz telemetry
Tắt tạm Windows Defender để tải Mimikatz.

![Disable Windows Defender](./images/Pasted image 20240604152850.png)

Chạy Mimikatz.

![Start Mimikatz](./images/Pasted image 20240604153518.png)

Config Wazuh logall = yes trên server (để debug dễ), edit filebeat enable archives.

Tạo index wazuh-archives-* .

Check log bằng command:
```
cat /var/ossec/logs/archives/archives.log | grep -i mimikatz
```

![Check Mimikatz Logs](./images/Pasted image 20240604154130.png)
![Mimikatz Sysmon Event](./images/Pasted image 20240604154620.png)
![Mimikatz Logs Generated](./images/Pasted image 20240604154910.png)
![Mimikatz Logs](./images/Pasted image 20240604155026.png)

### 4.3 Tạo custom rule detect Mimikatz
Dùng field originalFileName để detect dù attacker đổi tên file (rất thực tế).

![Mimikatz Original Filename](./images/Pasted image 20240604155124.png)

Tạo rule level 15 trong local_rules.xml:

```xml
<rule id="100002" level="15">
  <if_group>sysmon_event1</if_group>
  <field name="win.eventdata.originalFileName" type="pcre2">(?i)\\mimikatz\.exe</field>
  <description>Mimikatz Usage Detected</description>
  <mitre>
    <id>T1003</id>
  </mitre>
</rule>
```

![Wazuh Rules](./images/Pasted image 20240604155204.png)
![Sysmon Rules](./images/Pasted image 20240604155940.png)
![Custom Mimikatz Rule](./images/Pasted image 20240604160828.png)
![Custom Mimikatz Rule Added](./images/Pasted image 20240604164746.png)

Test bằng cách rename Mimikatz rồi chạy lại → alert phải trigger.

![Renamed Mimikatz](./images/Pasted image 20240604165046.png)
![Mimikatz Started](./images/Pasted image 20240604165457.png)
![Mimikatz Alert Triggered](./images/Pasted image 20240604165717.png)

## 5. Automation với Shuffle + TheHive

### 5.1 Setup Shuffle
Tạo account trên shuffler.io, tạo workflow mới, add Webhook trigger.

![Shuffle Account](./images/Pasted image 20240604171432.png)
![Create Shuffle Workflow](./images/Pasted image 20240604171857.png)

Thêm integration vào ossec.conf của Wazuh manager, dùng rule_id 100002.

Test bằng cách chạy Mimikatz lại.

### 5.2 Xây dựng Mimikatz Workflow
Các bước mình làm:
1. Nhận alert từ Wazuh
2. Extract SHA256 hash (dùng regex `SHA256=([0-9A-Fa-f]{64})`)
3. Check VirusTotal
4. Tạo alert trong TheHive
5. Gửi email cho SOC analyst

Tạo API key VirusTotal, add app vào workflow.

Setup TheHive: tạo org, user, API key cho Shuffle.

Configure TheHive action trong Shuffle với JSON payload chi tiết.

Thêm Email notification.

**Mẹo:** Nếu TheHive không nhận alert thì check firewall allow port 9000 từ mọi nơi (hoặc IP Shuffle).

## 6. Kết luận
Mình đã setup xong SOC Automation Lab với Wazuh, TheHive, Shuffle. Workflow tự động detect Mimikatz → check VT → tạo case trong TheHive → gửi mail thông báo. Rất thực tế và hữu ích cho home lab.

Những gì mình đạt được:
1. Windows + Sysmon + Wazuh agent.
2. Wazuh server thu thập event.
3. TheHive quản lý incident.
4. Custom rule detect Mimikatz bằng originalFileName (khá hay).
5. Shuffle SOAR automate toàn bộ flow.
6. Integrate VirusTotal + email notification.

Anh em làm theo note này thì sẽ có nền tảng tốt để customize thêm. Mình khuyên nên tiếp tục thêm playbook, integrate thêm tool khác (Cortex, MISP...), và test với nhiều technique hơn.

Automation giúp SOC của chúng ta hiệu quả hơn rất nhiều. Cảm ơn anh em đã đọc note này. Có thắc mắc thì comment hoặc mở issue nhé!

## 7. References
- https://www.mydfir.com/
- https://www.youtube.com/watch?v=Lb_ukgtYK_U
