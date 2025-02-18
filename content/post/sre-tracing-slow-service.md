---
title: 'Tracing dịch vụ bị chậm trên môi trường Kubernetes Production'
date: 2025-02-18T23:41:42+07:00
draft: false
tags: ["k8s", "Containerd", "Cgroups", "tracing", "sự cố", "SRE"]
categories: ["SRE"]
---

Nhét lại bộ bài trong hộc tủ vào túi đeo chéo (hồi còn ở cty tôi hay lòe mấy đứa bằng mấy cái tricks bài lỏ lỏd ), gấp lại quyển sổ tay đã 5 năm nằm trên cái bộ máy tính từ hồi tôi vào công ty. Mấy câu chào tạm biệt với anh em, cái bắt tay vội vã với sếp ngoài hành lang vắng người, ánh mắt hơi đỏ, nhưng cũng đến lúc phải rời khỏi cái “safe zone” ấy rồi.
Đã khá lâu kể từ ngày mà tôi quyết định chuyển sang 1 nơi làm việc khác, đôi khi nhớ lại, cũng thấy có vài thứ hay ho mà tôi đã làm được. Âu cũng là chút may mắn của bản thân khi được làm việc trực tiếp vs quản lý cũng chính là CTO của 1 cty Product lớn và lâu đời ở VN này. Hôm nay,  tình cờ xem lại vài cái note xử lý sự cố, tôi quyết định ghi lại một chút hồi tưởng về những gì đã trải qua. Biết đâu, nó sẽ là cảm hứng, hay là kiến thức cho chính mình sau này thì sao.

Bắt đầu thôi nhỉ ...

Hồi đó, khi còn làm SRE tại công ty cũ, vào một ngày không đẹp trời lắm đội AI (ngồi ngay dãy bàn bên cạnh) gửi đến cho tôi một request: " Anh Q xem giúp e xử lý một số dịch vụ trong dự án AI bị chậm hơn khi sử dụng trên môi trường Production với 😢". Điểm kỳ lạ là không có lỗi hay exception nào xuất hiện, và điều này khiến mọi thứ càng thêm khó hiểu mặc dù đã đưa dịch vụ ra các worker dùng riêng của dự án. Nếu nói về tài nguyên, Production luôn vượt trội hơn môi trường Test, R&D, nhưng thực tế thì các dịch vụ chạy trên Test lại nhanh hơn đến 10-20 lần.
Tôi nhớ mình đã mở biểu đồ Querry latency trên ELK và đứng hình một lúc. Mấy cái thông số xử lý như thể bị kéo dài một cách vô lý: một request đơn giản vào dịch vụ spell correction với input chỉ rất ngắn mà lại mất tới 4.2 giây để xử lý. Với chút ít kinh nghiệm, và một chút linh cảm trong đầu tôi biết đây không chỉ là vấn đề tài nguyên, mà là một câu đố phức tạp hơn đang chờ đợi mình.

*Nói qua sơ bộ thì dịch vụ này là để kiểm tra chính tả cho 1 đoạn văn bản, request sẽ chứa sentences đi vào Model AI xử lý và response sẽ trả lại xem lỗi chính tả ở đâu, sửa thế nào!*

**Bạn có thể tưởng tượng dịch vụ chậm như thế nào qua ảnh này:**
![Pasted image 20250112015156.png](/sre-tracing-slow-service-img/Pasted_image_20250112015156.png)

---
### Phân tích vấn đề

Khi đối mặt với sự cố hiệu suất, ta phải có một số giả thiết về nó, việc đầu tiên là chia nhỏ hệ thống thành từng phần rõ ràng. Với tôi, một request sẽ phải qua rất nhiều lớp để có thể đi đến được dịch vụ bên dưới của dự án, và tôi sẽ chia nó thành ba lớp chính:

1. **Lớp điều hướng request**: Từ Domain, Firewall, Load Balancer, Ingress đến Endpoint.
2. **Lớp xử lý logic**: Các container, pod, và luồng logic trong ứng dụng.
3. **Lớp lưu trữ**: Database, file storage, hoặc các config liên quan.

Sự chậm trễ có thể xuất phát từ một hoặc nhiều thành phần trong các lớp này. Vì vậy, bước tiếp theo là tiến hành kiểm tra từng lớp, phân tích và loại suy để xác định thành phần nào hoạt động ổn định và nhanh chóng, và thành phần nào gặp vấn đề hoặc bị chậm.

---
### Loại suy: Điều hướng request

Công cụ đầu tiên tôi dùng là `curl`, một công cụ trung thành và quen thuộc của mọi SRE. Tôi viết một lệnh curl có thể đo thời gian cho từng bước: từ phân giải DNS đến kết nối server, từ bắt đầu truyền tải đến khi hoàn tất. Cấu hình như sau:

```Bash
curl -w "@curltime" -X 'POST' \
  'https://aiservice.xxx.xx/api/v1/correction-spell?use_g=false' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
    "sentences": "Đối với tàu quân sự...",
    "num_suggestions": 3,
    "beam_width": 5,
    "boas": true,
    "eosa": true
  }'

```

File `curltime` để đo từng thông số:

```Bash
     time_namelookup:  %{time_namelookup}s\n
        time_connect:  %{time_connect}s\n
     time_appconnect:  %{time_appconnect}s\n
    time_pretransfer:  %{time_pretransfer}s\n
       time_redirect:  %{time_redirect}s\n
  time_starttransfer:  %{time_starttransfer}s\n
                     ----------\n
          time_total:  %{time_total}s\n
```

Qua nhiều thử nghiệm, từ mạng nội bộ công ty đến mạng bên ngoài, thay đổi cả domain trong file hosts để test ingress, tôi nhận ra request được chuyển đến dịch vụ Uvicorn (ASGI server của dự án) rất nhanh. Điều này loại bỏ khả năng vấn đề nằm ở tầng điều hướng. Loại suy được một thành phần lớn. Tiếp theo, tôi  sẽ tập trung vào lớp xử lý logic.

---
### Lớp xử lý logic

Khi xử lý logic ta cũng sẽ có một số nguyên nhân dẫn đến việc chậm trễ, một số lý do có thể kể đến như: Thiếu resource về RAM, thiếu CPU, việc đọc ghi I/O disk bên dưới bị chậm, xử lý logic code có vấn đề (CodeSmell, vòng lặp xử lý quá tải, Recursive function,...) . Việc cấp phát tài nguyên máy chủ trên các môi trường Test và Production tương đối giống nhau, đều sử dụng Ubuntu 20.04 và các thành phần setting trong kernel như sysctl.conf cũng không khác biệt.

Tôi chuyển sang kiểm tra lần lượt các nguyên nhân tiềm ẩn trong khi logic xử lý:
- Thiếu tài nguyên (RAM, CPU).
- I/O disk chậm.
- Vấn đề trong code logic (vòng lặp quá tải, đệ quy...).

Bất ngờ thay, tài nguyên hệ thống dường như dư dả:
- CPU trên node worker chỉ chạm ngưỡng 40-50%.
- RAM cũng chỉ sử dụng khoảng 70-80%.
- Các dịch vụ của dự án AI đã được tách ra các node worker riêng.

Khi đối diện với các thành phần chậm treo trong hệ thống, ta thường có xu hướng nghĩ ngay đến việc tăng tài nguyên cho Container, POD, đây là một giải pháp nhanh chóng để giải quyết vấn đề. Đặc biệt, trong trường hợp này, các dịch vụ AI cũng không nằm ngoài xu hướng đó. Tài nguyên của POD đã được tăng gấp đôi, thậm chí gấp ba. Nhưng ngay cả sau khi áp dụng thay đổi này, vấn đề vẫn không được giải quyết. Tuy nhiên, qua quá trình này, tôi cũng thu thập được một số thông tin quan trọng, giúp tôi có nhiều insight hơn về nguyên nhân gốc rễ của sự cố.

**Thông tin thu thập được từ hệ thống:**
1. **Pod không đạt ngưỡng limit của container**
2. **Dịch vụ Uvicorn:**
    - Dịch vụ nhận request nhanh và mượt trên tất cả các node.

Lúc này, tôi bắt đầu nghi ngờ rằng có một yếu tố khác đang can thiệp, đặc biệt là khi môi trường Production có nhiều dịch vụ chạy song song, trong khi môi trường test chỉ có một vài service cơ bản. Lý do tôi nghi ngờ là vì trên Prod có thêm 1 dịch vụ Captcha được lập lịch đứng cùng, dịch vụ này sử dụng model AI để xử lý hình ảnh. Chính vì vậy, tôi quyết định thử một số test case để làm rõ các yếu tố có thể ảnh hưởng đến hiệu suất. Trong quá trình này, kết hợp thêm việc sử dụng các công cụ và câu lệnh để đo đạc tài nguyên của hệ thống, đặc biệt là CPU, RAM và I/O Disk.

**Brief về các công cụ và câu lệnh được dùng:**

1. Lệnh `mpstat` là một lệnh trong Linux dùng để báo cáo về hiệu suất CPU. Cụ thể sử dụng lệnh `mpstat -P ALL 1` sẽ hiển thị thống kê CPU cho tất cả các bộ xử lý với tần suất một giây. Ta có thể để ý một số metrics:
	-  Hệ thống bình thường nếu:
		`%usr` , `%sys` và `%idle` có giá trị hợp lý, không quá cao hoặc quá thấp.
		`%idle` cao (ví dụ: 80-90%) nghĩa là CPU đang nhàn rỗi nhiều, không bị tải cao.
		`%usr` và `%sys` không quá cao (dưới 50%) cho thấy không có nhiều tác vụ user và hệ thống đang chạy.
	- Hệ thống có vấn đề khi:
		`%iowait` cao (trên 5%) cho thấy hệ thống đang chờ I/O nhiều, có thể do đĩa cứng chậm hoặc vấn đề mạng.
		`%steal` cao có thể cho thấy vấn đề về ảo hóa, CPU bị steal bởi hypervisor.
		`%usr` hoặc `%sys` quá cao (trên 70-80%) cho thấy CPU đang bị tải cao, có thể gây ra hiệu suất kém.

2. Lệnh `vmstat` (Virtual Memory Statistics) trong Linux cung cấp thông tin về hệ thống bộ nhớ, swap, I/O, và CPU. Khi chạy `vmstat 1`, hệ thống sẽ báo cáo các thống kê này mỗi giây.

3. Lệnh `iostat -x 1` trong Linux cung cấp thông tin về hoạt động của các thiết bị lưu trữ (disk) như tốc độ đọc/ghi, độ chậm (latency), và tình trạng sử dụng CPU. Một vài thông số ta cần để ý như:
	**%util cao**: Các thiết bị lưu trữ có %util cao cho thấy chúng đang gặp phải vấn đề tải.
	**Thời gian chờ I/O cao**: CPU đang phải chờ đợi nhiều để xử lý I/O.
	**Tốc độ xử lý I/O chậm**: `await` và `svctm` có giá trị cao, cho thấy thời gian chờ và thời gian dịch vụ cho các yêu cầu I/O đều cao.

**Kết quả từ các thử nghiệm:**
- **Case 1:** Service Spell Correction chạy một mình trên node (không có service captcha).  
    **Kết quả:** Dịch vụ chạy mượt mà, thời gian xử lý nhanh chóng.
    
- **Case 2:** Service Spell Correction chạy cùng 1-2 pod service captcha.  
    **Kết quả:** Thời gian xử lý tăng từ 1s lên 2.5s.
    
- **Case 3:** Service Spell Correction chạy cùng 4-5 pod service captcha.  
    **Kết quả:** Thời gian xử lý tăng lên đáng kể, từ 2.5s lên 9-10s.
    
- **Case 4:** Sau khi xóa hết pod captcha trên node worker (WK).  
    **Kết quả:** Thời gian xử lý giảm nhanh chóng, trở lại mức bình thường 1-2s.
    
- **Case 5:** Scale pod lên 5 trên worker.  
    **Kết quả:** Service Spell Correction xử lý nhanh và ổn định, chỉ mất khoảng 0.5-1.5s, khi không có service captcha trên worker.
    
- **Case 6:** Đưa pod Spell Correction sang node worker khác, không cho pod của captcha schedule trên đó.  
    **Kết quả:** Mặc dù dịch vụ Spell Correction chạy trên worker mới, thời gian xử lý vẫn chậm (9s), cho thấy có sự can thiệp từ các dịch vụ AI khác đang chạy trên worker.

**Điều chỉnh tài nguyên:**

Tôi thử thay đổi từ Bustable sang QOS Guaranteed, với request = limit và không đặt limit tài nguyên, nhưng không có sự thay đổi về hiệu suất. Các dịch vụ vẫn xử lý chậm như cũ.

---

### Tracing và Profiling:

Khi dùng Py-spy và lệnh `perf record -F 99 -a -g -- sleep 10` cho code python của dịch vụ AI , tôi phát hiện rằng thư viện `Libgomp-a34b3233.so` của PyTorch chiếm nhiều thời gian xử lý. Đây là thư viện cho phép thực thi song song trên nhiều lõi CPU, và rõ ràng CPU đang gặp vấn đề khi xử lý lượng lớn phép toán song song. Lúc này, tôi có thể suy luận rằng vấn đề chủ yếu nằm ở phần xử lý CPU, chứ không phải ở tài nguyên hệ thống khác.
Hơi tiếc là không còn ảnh nào chụp lại để show ra đây (￣へ￣)
### Kiểm tra từ bên trong container pod:

Tiến hành kiểm tra từ bên trong container qua câu lệnh `kubectl exec -it` để xác nhận tình trạng tài nguyên. Các thông số như `nr_periods`, `nr_throttled` và `throttled_time` cao trong **/sys/fs/cgroup/cpu/cpu.stat** cho thấy pod bị giới hạn mức xử lý CPU, đây chính là nguyên nhân làm chậm trễ quá trình xử lý.

### Confirm bằng Stress test:

Cuối cùng, tôi quyết định thực hiện một stress test để confirm lại giả thiết với câu lệnh `stress-ng --cpu 2 --cpu-method int32 --timeout 240s --verbose --metrics-brief`, thử nghiệm này giúp mô phỏng tác vụ tính toán nặng mà AI model cần phải xử lý. Lệnh này để Stress test tài nguyên tính toán CPU bằng các phép toán số nguyên 32bit. Tại sao lại chọn lệnh tính toán này, ngắn gọn là bởi vì Model AI sử dụng rất nhiều các tính toán ma trận. Trong các trường hợp khác ta có thể sử dụng phép toán khác. Kết quả là khi CPU bị dồn vào xử lý cùng lúc, thời gian xử lý tăng vọt, và mặc dù worker node vẫn còn nhiều core CPU, dịch vụ AI lại sử dụng nhiều CPU hơn (từ 600m CPU lên 1500m, khi có tính toán cùng lúc thì lên đến 2k-5k CPU nếu không đặt limit).

Đã có thể kết luận là việc "**sử dụng nhiều tài nguyên CPU tính toán cùng lúc**" chính là nguyên nhân chính gây ra sự chậm trễ. Dù đã  **Isolate tài nguyên CPU** giữa các dịch vụ, vấn đề vẫn không được giải quyết triệt để. Tuy nhiên, trong khoảng thời gian chưa tìm ra workaround tôi nhận thấy rằng khi tách riêng dịch vụ hoặc sử dụng GPU Container, hệ thống có thể hoạt động ổn định hơn.

---

### Research để xử lý vấn đề:

Vì muốn đào sâu hơn vấn đề để giải quyết triệt để, tôi đã ngồi đọc lại các tài liệu về ContainerD, Cgroups. Cgroups là một tính năng của Kernel Linux nhằm cấp phát giới hạn, tính toán và tách biệt việc sử dụng tài nguyên của một tập hợp các process. Containerd sử dụng tài nguyên này để khởi tạo các pod container. Nếu mọi người thắc mắc tại sao lại là containerd và cgroups có thể xem bức ảnh này:

![Pasted image 20250112023030.png](/sre-tracing-slow-service-img/Pasted_image_20250112023030.png)

Cgroups và CPU khi phân tách CPU trong K8S chịu sự quản lý của Linux Kernel, mặc định sẽ sử dụng cơ chế Completely Fair Scheduler (CFS) quota. [https://docs.kernel.org/scheduler/sched-design-CFS.html]. CFS chịu trách nhiệm phân chia thời gian CPU một cách công bằng giữa các process. Đảm bảo rằng không có process nào bị bỏ qua hoặc bị ưu tiên quá mức. Chính vì lý do này các process sử dụng nhiều CPU có thể đưa các workload của mình sang các core CPU khác.

Vậy thì có cách nào để Isolate CPU cho một tiến trình hoặc một container trên linux không? Có , trong cgroups v1 một cơ chế là Cpuset. CPUSet tập trung vào việc giới hạn và kiểm soát các tiến trình để chạy trên các CPU cụ thể, giúp quản lý tài nguyên, isolate CPU tốt hơn.
Sau khi research 2 cơ chế trên, ta có thể tìm thấy cpuset trong mã nguồn của kubelet. Nhưng cơ chế này không được mặc định bật mà phải có flag `cpu-manager-policy=static`.

![Pasted image 20250112023127.png](/sre-tracing-slow-service-img/Pasted_image_20250112023127.png)

Mặc định kubelet sẽ sử dụng policy None:  `kubelet/cm/cpumanager/policy_none.go` , sau khi set `static` kubelet sẽ bắt đầu sử dụng `pkg/kubelet/cm/cpumanager/policy_static.go` và có vẻ đây đúng là thứ mà chúng ta cần. 

![Pasted image 20250112023204.png](/sre-tracing-slow-service-img/Pasted_image_20250112023204.png)

Sau khi config static flag cho kubelet. Đưa dịch vụ về `QoS Class Guaranteed` tức Limit bằng với Request, dịch vụ đã hoạt động ổn định với đúng lượng tài nguyên tính toán của CPU được cấp riêng. Tài nguyên tính toán của dịch vụ được Isolate một cách triệt để và không còn bị ảnh hưởng bởi các tính toán của các service POD khác trong Node. Service AI chạy nhanh trở lại và không tốn tài nguyên CPU như trước do không có tài nguyên nào được lập lịch vào những Core CPU được cấp cho riêng dịch vụ AI. Thời gian xử lý từ 7-10s giảm xuống còn 0.5 -1s.

![Pasted image 20250112023302.png](/sre-tracing-slow-service-img/Pasted_image_20250112023302.png)

Về phần QoS bạn có thể đọc ở đây:
[https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/]

![Pasted image 20250112023336.png](/sre-tracing-slow-service-img/Pasted_image_20250112023336.png)

---
### Tóm lại thì:

- Dịch vụ (POD, Container) AI xử lý chậm khi có nhiều tài nguyên tính toán CPU chạy đồng thời trên một Node.
- Mặc định kubelet k8s không sử dụng việc  phân tách tài nguyên CPU từng core riêng biệt mà phải sử dụng flag `cpu-manager-policy=static`.
- Chỉ khi sử dụng flag trên, các dịch vụ sử dụng `QoS Guaranteed` mới được phân tách tài nguyên tính toán CPU một cách rõ ràng và không bị ảnh hưởng bởi các dịch vụ khác. (Lưu ý, khi sử dụng `BustAble` tức là lượng request nhỏ hơn Limit container cũng sẽ không được phân tách và đảm bảo tài nguyên tính toán trên mọi policy ).
- Sau khi được phân tách tài nguyên tính toán, dịch vụ sẽ hoạt động nhanh và ổn định trở lại.

