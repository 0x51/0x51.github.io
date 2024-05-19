---
title: 'Điện VN Analysis'
date: 2023-03-02T10:59:46+07:00
draft: false
tags: ["issuers", "tail risk", "poker"]
categories: ["finance"]
---

####  Cảnh báo - nontech =)))) 

_________________________
Dạo này mình có nổi hứng đọc báo, vớ được vài bài báo về điện gió điện mặt trời :
 - https://vnexpress.net/nhieu-chu-dau-tu-dien-tai-tao-chay...
 - https://vnexpress.net/nha-dau-tu-dien-gio-mat-troi-chuyen...

Hôm nay skip 1 buổi Poker với hội bạn :v , tiện cũng đang có hứng phân tích tí tẹo để đỡ quên mấy cái kiến thức cơ bản, thì mạn phép được bàn về cái khái niệm mà chắc là ai hay bất kỳ doanh nghiệp nào cũng phải đụng đến khi chơi mấy cái Game tài chính này, đấy chính là rủi ro khi đầu tư [Tail Risk]. Cái khái niệm này khá hay và có thể áp dụng để chơi Poker hay tính toán cái gì đó cũng được :v không có dùng tí số tí má nào đâu nên cũng đáng để ngồi ngẫm đấy chứ :3

Về mấy bài báo trên vnexpress nói về điện gió và điện mặt trời người ta hay chỉ ra mấy cái nguyên nhân chính dẫn đến vấn đề nan giải của ngành điện hiện nay (Gọi nó là dilemma của ngành điện cũng chẳng sai). 
 1. Dùng nợ vay lớn
 2. Giá mua điện mới thấp hơn giá ưu đãi trước đây (FIT) 20-30%

Vậy thì do lỗi chính sách hay lỗi của doanh nghiệp đây? Đến phần security analysis rồi.

![Analysis: Analysis](/evn-dilemma-img/analysis.jpg)

Từ doanh thu cho đến lợi nhuận ta sẽ phải trải qua 3 cái rủi ro:
 - **Sale risk (SR)**: Doanh thu bán hàng, lúc cao lúc thấp, trong Poker thì sẽ giống kiểu Bet sau mỗi turn để hy vọng mình kiếm được profit nhiều nhất có thể
 - **Operating risk (OR)** : Là những cái chi phí cố định, bị fix sẵn, có bán hàng bao nhiêu thì nó cũng nhiêu đó. VD như tiền thuê mặt bằng, khấu hao, nhân công, ... trong Poker sẽ như Small Blind vs Big Blind vậy (chi phí mù, không làm gì cũng phải mất). Cái này sẽ hút tiền vốn của doanh nghiệp và hút dần mòn chip của người chơi =)))
 - **Money Loan/ Financial Risk (FR)** : Tiền nợ vay, cái này cũng bị fix như Operating risk. Poker không có cái này :v nhưng dạo này hội bạn mình có kiểu hùn vốn ở River để đấm nhau với mình thì đại khái cũng na ná thế =)))) móe .. cay thặcc
OK! Phân tích này.

![poker: poker](/evn-dilemma-img/poker.jpg)

Trong rủi ro thì OR sẽ là cái hơi khách quan, thuộc về từng ngành và ở đâu cũng vậy. Thế nên doanh nghiệp hay người chơi sẽ chủ động quyết định FR, tức là nợ vay phải trả nhiều hay ít tùy vào đánh giá SR. Mấy dự án năng lượng điện mặt trời và điện gió đều có nhà máy, thiết bị đắt tiền, tức OR lớn. Và gần như tất cả đều vay nhiều, tức FR lớn. Vì sao nên nỗi?

Nghĩ mà xem, ngành điện luôn được mặc định là doanh thu - dòng tiền ổn định. Cái SR của nó nhỏ.
Vì vậy doanh nghiệp có SR nhỏ sẽ dồn năng lực để đấm OR và FR. Đấy là còn chưa kể nếu phân tích sâu hơn tí, chi phí hoạt động cố định của dự án điện gió, điện mặt trời phần lớn là chi phí khấu hao. Ai tính tiền nắng tiền gió vào đấy mần chi đâu ¯\_(ツ)_/¯. Mà cái paper cost này tuy có thể làm âm lợi nhuận nhưng khó làm âm dòng tiền được. Vậy nên mấy dự án điện gió điện mặt trời này vay nhiều không phải nguy hiểm về mặt tài chính cho lắm... Nhưng, nó chỉ đúng nếu cái điều giả định SR thực sự nhỏ là đúng.

Và vấn đề ở đây là nó không đúng :v

Khi các cụ (Gov) đưa ra chính sách cơ chế giá FIT, giá tốt apply tận 20 năm, các doanh nghiệp kháo nhau đi đầu tư điện gió và điện mặt trời và khả năng cao cũng chỉ đơn giản nghĩ mình sẽ được hưởng giá FIT tốt, với một ít xác suất xấu là giá sau đó sẽ thấp hơn giá FIT tí tẽo. Wost case thì bao giờ cũng phải phân tích, nhưng chắc cũng ko bên nào đủ trầm cảm để nghĩ đến wost case lại là giá sau FIT thấp hơn tận 20-30%. 
Tail Risk ập đến khi những điều tồi tệ xảy ra với xác suất cao hơn dự báo ban đầu. Một bài học về Tail Risk . . . với giá hơi chát 😃

À nếu bạn có thắc mắc dăm ba cái đống kiến thức lày ở đâu :v Có real hay đáng tin ko thì có thể tham khảo cuốn lày...

![issuers: issuers](/evn-dilemma-img/issuers.jpg)