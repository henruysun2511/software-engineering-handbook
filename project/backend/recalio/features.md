# Hệ thống Học tập RECALIO — Tổng quan Chức năng

## Giới thiệu

RECALIO là một nền tảng học tập giúp người dùng ghi nhớ từ vựng và kiến thức một cách hiệu quả thông qua phương pháp lặp lại ngắt quãng. Khác với cách học truyền thống chỉ ôn tập một lần rồi quên, RECALIO sử dụng các thuật toán thông minh để xác định thời điểm ôn tập tối ưu cho từng mục kiến thức, giúp trí nhớ được củng cố lâu dài với thời gian học tập ít nhất.

Hệ thống cho phép người dùng tạo các bộ thẻ học tập, tổ chức chúng theo cấu trúc thư mục, và chia sẻ với cộng đồng. Mỗi bộ thẻ chứa các ghi chú từ vựng giàu thông tin — từ, nghĩa, phiên âm, câu ví dụ và âm thanh — từ đó tự động sinh ra thẻ học để người dùng ôn tập hàng ngày. Ngoài ra, RECALIO còn tích hợp trí tuệ nhân tạo để hỗ trợ tạo nội dung học tập, cùng với các tính năng xã hội và game hoá nhằm duy trì động lực và sự hứng thú trong quá trình học.

## Xác thực và Bảo mật

Hệ thống hỗ trợ hai phương thức xác thực: đăng ký/đăng nhập bằng tên người dùng và mật khẩu, hoặc thông qua tài khoản Google OAuth. Khi người dùng đăng nhập thành công, hệ thống cấp một cặp token bao gồm JWT (dùng cho xác thực yêu cầu) và refresh token (dùng để làm mới JWT khi hết hạn). Refresh token được bảo vệ bằng cơ chế rotation — mỗi lần làm mới, token cũ bị thu hồi và token mới được cấp, giúp phát hiện truy cập trái phép. Người dùng có thể đăng xuất khỏi một thiết bị cụ thể hoặc toàn bộ thiết bị cùng lúc.

## Quản lý Người dùng

Mỗi người dùng có một hồ sơ cá nhân với tên hiển thị, tiểu sử và ảnh đại diện, có thể chỉnh sửa sau khi đăng nhập. Hệ thống phân quyền với ba vai trò: người dùng thông thường (USER), người kiểm duyệt (MODERATOR) và quản trị viên (ADMIN). Quản trị viên có thể xem danh sách toàn bộ người dùng với bộ lọc tìm kiếm, kích hoạt hoặc vô hiệu hoá tài khoản, và thay đổi vai trò của người dùng khác.

## Bộ thẻ và Tổ chức Thư mục

Bộ thẻ (deck) là đơn vị tổ chức kiến thức trung tâm của hệ thống. Người dùng có thể tạo bộ thẻ với tên, mô tả, ảnh bìa và thẻ tag để phân loại. Các bộ thẻ được tổ chức theo cấu trúc cây thư mục, cho phép người dùng tạo tối đa bốn cấp cha con. Mỗi bộ thẻ có thể được đặt ở chế độ công khai để chia sẻ lên chợ bộ thẻ (marketplace) hoặc ở chế độ riêng tư.

Người dùng có thể sao chép (clone) bộ thẻ công khai của người khác về tài khoản của mình. Hệ thống chỉ cho phép mỗi người dùng clone một bộ thẻ cụ thể đúng một lần. Khi clone, toàn bộ cấu trúc bộ thẻ bao gồm ghi chú, thẻ học và cài đặt được nhân bản. Ngoài ra, người dùng có thể lưu trữ (archive) các bộ thẻ không còn sử dụng và khôi phục lại khi cần. Bộ thẻ bị xóa được soft-delete, nghĩa là dữ liệu vẫn tồn tại trong cơ sở dữ liệu nhưng không hiển thị. Quản trị viên có quyền cấm (ban) bộ thẻ vi phạm và hệ thống sẽ gửi thông báo đến chủ sở hữu.

## Cấu hình Thuật toán Ôn tập

Mỗi bộ thẻ có một bộ cài đặt ôn tập riêng, được tự động tạo với giá trị mặc định khi tạo bộ thẻ mới. Người dùng có thể tuỳ chỉnh các tham số của thuật toán Spaced Repetition bao gồm: lựa chọn thuật toán (SM-2 hoặc FSRS), số thẻ mới mỗi ngày, số lượt ôn tập mỗi ngày, các bước học tập, khoảng cách tốt nghiệp, hệ số dễ, phần thưởng dễ, bước khó, khoảng cách tối đa, bước học lại, ngưỡng chai (leech), và hành động khi phát hiện thẻ chai (tạm ngưng hoặc gắn cờ).

## Thẻ học và Thuật toán SM-2

Thẻ học (card) là đơn vị cơ bản của quá trình ôn tập. Mỗi thẻ có một mặt trước và mặt sau, được render từ mẫu (template) với các trường dữ liệu động. Hệ thống quản lý thẻ qua năm trạng thái: Mới (NEW), Đang học (LEARNING), Học lại (RELEARNING), Ôn tập (REVIEW) và Tạm ngưng (SUSPENDED).

Khi ôn tập, người dùng đánh giá mức độ ghi nhớ theo bốn mức: Lại (AGAIN), Khó (HARD), Tốt (GOOD) và Dễ (EASY). Thuật toán SM-2 xử lý từng mức đánh giá dựa trên trạng thái hiện tại của thẻ:

- Nếu thẻ ở trạng thái Mới, đánh giá Lại giữ thẻ ở trạng thái Mới, trong khi Dễ đưa thẻ thẳng sang Ôn tập với khoảng cách bốn ngày.
- Nếu thẻ đang học, hệ thống áp dụng các bước học tập tăng dần: đánh giá Lại đưa về bước đầu tiên, Khó nhân đôi khoảng cách bước hiện tại, Tốt chuyển sang bước kế tiếp và Dễ đưa thẳng sang Ôn tập.
- Nếu thẻ đang ôn tập, thuật toán điều chỉnh hệ số dễ: đánh giá Lại giảm 0.2 và đưa về trạng thái Học lại, Khó giảm 0.15 với khoảng cách nhân 1.2, Tốt giữ nguyên và Dễ tăng 0.15 với thưởng 1.3.

Người dùng có thể gắn cờ thẻ với bốn màu sắc khác nhau, tạm ngưng thẻ hoặc ẩn thẻ đến hết ngày. Hệ thống cung cấp API lấy danh sách thẻ đến hạn ôn tập và thống kê số lượng thẻ theo từng trạng thái.

## Ghi chú Từ vựng

Ghi chú (note) là đơn vị lưu trữ dữ liệu từ vựng bao gồm từ, nghĩa, phiên âm IPA, câu ví dụ và tệp âm thanh. Mỗi ghi chú thuộc về một bộ thẻ và một mẫu ghi chú (note template) xác định các trường dữ liệu.

Quy trình tạo ghi chú gồm hai bước. Đầu tiên, người dùng gửi dữ liệu đến API preview, hệ thống sẽ phát hiện ngôn ngữ của từ (dựa trên ký tự đặc trưng như Nhật, Hàn, Trung, Thái, Ả Rập, Việt, hoặc sử dụng thư viện franc) và kiểm tra xem tệp âm thanh đã tồn tại trong bộ nhớ đệm hay chưa. Sau đó, người dùng gửi yêu cầu confirm để tạo ghi chú chính thức. Hệ thống tự động tạo các thẻ học tương ứng dựa trên mẫu card template và xếp hàng đợi xử lý âm thanh nếu cần. Hệ thống cũng hỗ trợ tạo ghi chú từ tài liệu PDF đã qua xử lý AI và giới hạn tối đa 50 ghi chú mỗi bộ thẻ.

## Mẫu ghi chú và Mẫu thẻ

Mẫu ghi chú (note template) định nghĩa cấu trúc dữ liệu cho một loại ghi chú, bao gồm danh sách tên trường (ví dụ: Từ, Nghĩa, Phiên âm, Ví dụ). Hệ thống hỗ trợ ba loại mẫu: Cơ bản (BASIC — một thẻ mỗi ghi chú), Cơ bản đảo ngược (BASIC_REVERSED — hai thẻ xuôi và ngược), và Cloze (CLOZE — điền vào chỗ trống).

Mỗi mẫu ghi chú có thể có nhiều mẫu thẻ con (card template), mỗi mẫu định nghĩa mã HTML mặt trước, mặt sau và CSS với các placeholder động như {{Word}}, {{Meaning}}. Khi render, hệ thống thay thế các placeholder bằng dữ liệu thực tế từ ghi chú. Chỉ quản trị viên mới có quyền tạo, sửa hoặc xoá mẫu.

## Phiên học

Người dùng có thể bắt đầu một phiên học (study session) để tập trung ôn tập. Phiên học có thể gắn với một bộ thẻ cụ thể hoặc không, và hỗ trợ ba chế độ: Bình thường, Học cấp tốc (Cram) và Xem trước (Preview). Mỗi người dùng chỉ được có tối đa năm phiên đang hoạt động cùng lúc.

Khi kết thúc phiên, hệ thống tổng hợp thống kê bao gồm: số thẻ đã ôn, thời gian học, và phân bố điểm đánh giá. Người dùng có thể xem lịch sử các phiên học trước đây và chi tiết từng lượt ôn tập trong mỗi phiên.

## Đánh giá Bộ thẻ

Người dùng có thể đánh giá các bộ thẻ công khai trên thị trường với thang điểm từ một đến năm sao và để lại bình luận. Mỗi người dùng chỉ được đánh giá một bộ thẻ một lần, và không thể tự đánh giá bộ thẻ của chính mình. Chủ sở hữu bộ thẻ có quyền xoá các đánh giá trên bộ thẻ của họ.

## Ngôn ngữ

Hệ thống quản lý danh sách ngôn ngữ với mã ISO 639-1 và cung cấp API phát hiện ngôn ngữ tự động từ văn bản. Ngôn ngữ được xác định qua đặc điểm bộ ký tự và fallback bằng thư viện franc, hỗ trợ hơn 20 ngôn ngữ phổ biến. Quản trị viên có thể thêm, sửa hoặc xoá ngôn ngữ cũng như bật/tắt hỗ trợ cho từng ngôn ngữ.

## Thông báo

Hệ thống thông báo trong ứng dụng quản lý việc gửi và nhận thông báo qua nhiều kênh: email, web push và mobile push. Người dùng có thể tuỳ chỉnh cài đặt thông báo cá nhân, bao gồm bật/tắt từng kênh và đặt thời gian nhắc nhở học tập (mặc định 20:00 hàng ngày).

Bảy loại thông báo được hỗ trợ: nhắc nhở học tập, thẻ đến hạn, danh hiệu đạt được, bộ thẻ bị báo cáo, bộ thẻ bị cấm, góp ý mới và thông báo hệ thống. Một số loại thông báo như nhắc nhở học và thẻ đến hạn có thể gửi qua email, trong khi các loại khác chỉ gửi trong ứng dụng. Quản trị viên có thể gửi thông báo đến một người dùng cụ thể hoặc toàn bộ hệ thống. Thông báo được xử lý bất đồng bộ qua hàng đợi Bull/Redis.

## Trí tuệ Nhân tạo

RECALIO tích hợp Google Gemini API để cung cấp năm tính năng AI hỗ trợ học tập. Thứ nhất, trích xuất từ vựng từ văn bản tự do: người dùng dán một đoạn văn và hệ thống tự động phát hiện và trích xuất các từ/cụm từ quan trọng kèm nghĩa. Thứ hai, sinh từ vựng theo chủ đề: người dùng nhập chủ đề và hệ thống tạo ra danh sách từ vựng liên quan. Thứ ba, sinh từ đồng nghĩa và trái nghĩa cho một từ cụ thể. Thứ tư, xử lý tài liệu PDF: người dùng tải lên tệp PDF (tối đa hai trang), hệ thống trích xuất văn bản và AI chuyển đổi thành ghi chú có cấu trúc. Thứ năm, nhận diện vật thể trong ảnh: người dùng tải lên hình ảnh, AI phát hiện các đối tượng và tạo ghi chú từ vựng tương ứng.

Tất cả yêu cầu AI đều sử dụng prompt engineering chi tiết để đảm bảo đầu ra JSON hợp lệ và được phân tích cú pháp tự động.

## Theo dõi Người dùng

Hệ thống mạng xã hội cơ bản cho phép người dùng theo dõi và bỏ theo dõi lẫn nhau. Mỗi người dùng có thể xem danh sách người họ đang theo dõi, danh sách người theo dõi họ, và kiểm tra trạng thái theo dõi với một người dùng cụ thể. Quan hệ theo dõi là bất đối xứng — không có khái niệm bạn bè hai chiều. Người dùng không thể theo dõi chính mình.

## Báo cáo Vi phạm

Người dùng có thể báo cáo các bộ thẻ vi phạm chính sách với bốn lý do: vi phạm bản quyền, spam, nội dung không phù hợp và lý do khác. Mỗi người dùng chỉ được báo cáo một bộ thẻ một lần và không thể tự báo cáo bộ thẻ của mình. Khi một báo cáo được tạo, tất cả quản trị viên nhận được thông báo. Quản trị viên có thể xem danh sách báo cáo và cập nhật trạng thái thành Đã xem xét, Đã bác bỏ hoặc Đã xử lý.

## Game hoá

Hệ thống game hoá khuyến khích người dùng học tập đều đặn thông qua điểm kinh nghiệm (XP), cấp độ, chuỗi ngày học (streak), danh hiệu (achievement) và bảng xếp hạng.

Mỗi lượt ôn tập kiếm được năm XP. Công thức tính cấp độ là `floor(sqrt(totalXP / 100)) + 1`. Người dùng đặt mục tiêu ôn tập hàng ngày (mặc định 50 thẻ); khi hoàn thành, họ nhận được thưởng XP bằng 50% mục tiêu. Chuỗi ngày học được duy trì nếu người dùng học hôm qua hoặc hôm nay; nếu gián đoạn, chuỗi được đặt lại về một.

Danh hiệu được định nghĩa trong cơ sở dữ liệu với các điều kiện như chuỗi ngày đạt N, tổng số lượt ôn tập đạt N hoặc tổng số thẻ đạt N. Khi người dùng đạt điều kiện, danh hiệu tự động được mở khoá và hệ thống gửi thông báo kèm thưởng XP. Bảng xếp hạng hiển thị người dùng có tổng XP cao nhất, hỗ trợ ba khung thời gian: tuần này, tháng này và toàn thời gian.

## Góp ý

Người dùng có thể gửi phản hồi và góp ý về nền tảng. Mỗi góp ý kèm theo địa chỉ email của người gửi để quản trị viên có thể phản hồi. Quản trị viên có thể xem danh sách góp ý và đánh dấu đã đọc.

## Bài viết và Cộng đồng

Người dùng có thể tạo bài viết (post) để giới thiệu bộ sưu tập thẻ của mình. Mỗi bài viết có tiêu đề, nội dung, ảnh bìa, thẻ tag, và liên kết đến một hoặc nhiều bộ thẻ mà người dùng sở hữu (tối đa 50). Bài viết có thể được xuất bản để công khai.

Tính năng tương tác xã hội bao gồm: thích/bỏ thích bài viết (toggle like), báo cáo bài viết vi phạm (một báo cáo mỗi người dùng, không thể tự báo cáo), và bình luận.

## Bình luận

Bình luận trên bài viết hỗ trợ phân cấp một cấp — người dùng có thể bình luận trực tiếp hoặc trả lời bình luận khác, nhưng không thể trả lời vào câu trả lời. Mỗi bình luận giới hạn 2000 ký tự và có thể bị xoá mềm. Người dùng có thể thích hoặc bỏ thích bình luận. Chủ sở hữu bình luận có quyền sửa hoặc xoá bình luận của mình. Quản trị viên có quyền cấm bài viết nếu nội dung vi phạm.

## Tải lên và Quản lý Media

Hệ thống hỗ trợ tải lên tệp phương tiện qua Cloudinary CDN. Các định dạng được chấp nhận bao gồm JPEG, PNG, WebP và GIF cho hình ảnh, và MP3, WAV, OGG, M4A và WebM cho âm thanh. Dung lượng tối đa 500MB. Hệ thống tự động phân loại tệp vào thư mục hình ảnh hoặc âm thanh trên Cloudinary và hỗ trợ xoá tệp bằng publicId.

## Kiến trúc Kỹ thuật

Hệ thống được xây dựng trên NestJS với Prisma ORM kết nối PostgreSQL. Xác thực sử dụng JWT qua passport strategy với guard toàn cục. Các API được bảo vệ bằng role-based access control. Tài liệu API tự động được sinh qua Swagger.

Dữ liệu được tổ chức thành hơn 25 bảng với các mối quan hệ phức tạp. Hệ thống sử dụng hàng đợi Bull/Redis cho xử lý bất đồng bộ như tổng hợp âm thanh, gửi thông báo và xử lý tài liệu. Module đệm âm thanh giúp giảm thiểu yêu cầu API đến dịch vụ giọng nói bên ngoài.

Tổng cộng, hệ thống cung cấp hơn 100 điểm cuối API trải đều trên 18 module chức năng, tạo thành một nền tảng học tập hoàn chỉnh với đầy đủ tính năng từ quản lý nội dung, ôn tập thông minh, tương tác xã hội đến động lực học tập qua game hoá và hỗ trợ AI.
