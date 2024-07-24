---
title: 'K8s Security - Bảo mật Microservice trên K8s !!??'
date: 2024-05-19T22:53:05+07:00
draft: false
tags: ["k8s", "security", "DevSecOps"]
categories: ["security"]
---

#### Một cái tiêu đề rõ ràng quá đúng không :D Nhưng mà mình không nghĩ ra được cái tiêu đề nào nó vui hơn hay ít ra là ít học thuật hơn cái này nên là ... thôi, just go 4 it vậy :3

![meme: meme](/k8s-sec-img/meme.jpg)

Tại sao lại là k8s và tại sao lại là Microservice, mình nghĩ đây đáng lý phải là một chủ đề đáng để bàn luận chứ nhỉ. Người ta thường nói k8s là một nền tảng để quản lý container tập chung, và cũng không khó để tìm quanh Internet và đến 90% người ta đều nói vậy, với mình như vậy cũng đúng, nhưng ... chưa đủ lắm :D Để mà nói về sự thông minh và cái triết lý, core concept đỉnh chóp của k8s mình nghĩ nên đặt nó dưới góc nhìn của . . . một nền tảng quản lý APIs. 

Sorry khi bưng ra một cái góc nhìn ảo ma như vậy =))) nhưng mà tin mình đi, kubernetes thể hiện cả đống nguyên tắc của một APIs hoàn hảo. Hồi lâu rồi mình có kêu với 1 ô a ở cty là "e nghĩ e code lại nguyên được cái kubectl bằng curl với bash được đấy", ổng không tin,... vài câu chuyện về việc " thế thì chú đi làm cho GG mẹ đi" ... và vài câu bàn tán trao đổi qua lại hay ho nữa :D nhưng mình cười :3 Và ...cái trò con bò đấy đơn giản thật, thật đấy .

Cái chủ đề về Microservice đã đủ để đưa ra bàn tán đấm đá cả năm r, thêm vào cả cái nền tảng nhiều tính năng như k8s nữa =))) oh men, câu chuyện nghe có vẻ sẽ đi xa.. xa.... và, xa lắm.
Nào là Single Responsibility Principle, Decentralized Data Management, API-Driven, Stateless,Loose Cupling, Auto scaling, haha cái vũ trụ Micro thôi mà nghe vẻ nhiều vấn đề phết.

Nhìn lại k8s mà xem, mỗi một cái resource ở trên nó đều có group, vesion, kind, thậm chí còn thêm vào được cả external metadata :v. Hồi đầu sử dụng helm mình còn thắc mắc "Ơ đù mé, database của thằng helm đâu nhỉ, server của nó nằm chỗ mô, sao vẫn quản lý version ầm ầm ta :D sau này mới rõ, tất cả chỉ là metadata nhét thêm vào". Ơ thế bạn muốn cái API lý tưởng của bạn cần có thêm cái gì nữa. Thậm chí nó còn có CRDs để mở rộng thêm cho k8s nữa cơ mà :3.

Cái phần này sẽ có vẻ có giá trị hơn phần mình sẽ viết ở bên dưới đây, thật đấy, mấy cái về security bên dưới mang nặng tính lý thuyết chỉ là mình túm túm lại thôi. Còn nói thật =))) bạn muốn Microservice của bạn bảo mật :v làm hệt như những gì K8S làm với APIServer, Controller Manager, Scheduler của kubernetes ấy :3 Hehe -  Bắt đầu nhé!

#### K8S Sec

Để có cái nhìn tổng thể hơn về những vấn đề an toàn bảo mật của k8s ta không thể không nhắc đến mô hình Security 4C của các hệ thống Cloud Native. Mô hình 4C này chia hệ thống thành 4 lớp khác nhau đó chính là : Cloud, Clusters, Containers, và Code. Cách tiếp cận này sẽ giúp ta tăng cường bảo vệ theo chiều sâu (Defend in Depth) và cũng được nhiều người coi là phương thức chuẩn nhất, hiệu quả nhất để có thể bảo mật cho hệ thống phần mềm.

![4c: 4c](/k8s-sec-img/4c-model.jpg)

Mỗi lớp của mô hình này sẽ được xây dựng dựa trên lớp ngoài kế tiếp nó. Lớp Code được hưởng lợi từ việc bảo vệ tốt các lớp trước nó như Container, Cluster, Cloud. Nhưng Code layer cũng là lớp cuối cùng trong mô hình bảo mật Cloud Native. Nếu ta chỉ giải quyết các vấn đề bảo mật ở lớp Code, mà các lớp trên không đáp ứng được các tiêu chuẩn bảo mật thì cũng không đảm bảo độ an toàn của hệ thống. (Cloud trong mô hình này là nhắc đến các nhà cung cấp dịch vụ Cloud, nhưng cũng có thể hiểu ở đây là Datacenter, là các máy chủ chạy cụm K8S chúng ta). Ta sẽ đi vào phân tích từng phần cụ thể.

Container là một công nghệ ảo hóa nhẹ, cho phép chạy các ứng dụng độc lập với nhau trên cùng một máy chủ. Container có thể được triển khai nhanh chóng, dễ dàng và linh hoạt, giúp tăng hiệu quả và tiết kiệm chi phí. Tuy nhiên, container cũng đặt ra những thách thức về bảo mật, đòi hỏi sự chú ý đến các lớp bảo mật khác nhau, từ ứng dụng, container engine, network, storage cho đến host OS.

È ... mà thôi lười viết ra đây quá. 
Chi tiết nếu bạn muốn đọc thì mình có cắt một đoạn ở luận văn của mình ra đêy nhé :3
Yên tâm, legit và không có e bé :D
[Download file](/k8s-sec-file/k8s-sec.pdf)

À về cơ bản thì bạn cũng nên biết và hiểu về security context ở 3 cái role sẽ đi xuyên suốt quá trình làm việc cùng với k8s là Developers, Project Admins, và Administrators ở đây : [ https://github.com/kubernetes/design-proposals-archive/blob/main/auth/security.md ]
Hay lắm, đọc đi!
