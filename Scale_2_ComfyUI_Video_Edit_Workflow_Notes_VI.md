# Ghi chú Workflow chỉnh sửa Video Scale 2 trong ComfyUI

Các ghi chú này được tổng hợp từ transcript đính kèm. Workflow dùng cho chỉnh sửa video với Scale 2: thay thế nhân vật, thay đổi nền, chỉnh sửa vật thể cục bộ, compositing mask tùy chọn, và mở rộng theo chunk cho video dài hơn.

## Ý tưởng cốt lõi

Scale 2 chỉnh sửa video có sẵn bằng cách sử dụng:

- Một video nguồn.
- Một ảnh tham chiếu, thường là khung hình đầu tiên của video nguồn sau khi chỉnh sửa đối tượng/nền mục tiêu thành giao diện mong muốn.
- Prompt văn bản mô tả những gì đang diễn ra trong cảnh.
- Một video mask, thường được tạo bằng SAM 3.1 Multiplex, để giới hạn những gì được thay thế.
- Compositing tùy chọn để giữ nguyên các vùng không thay đổi từ video gốc.

Đây không được mô tả như một workflow tham chiếu-đến-video thuần túy. Ảnh tham chiếu hướng dẫn chỉnh sửa trong khi video nguồn cung cấp chuyển động và cấu trúc thời gian.

## Chuẩn bị ảnh tham chiếu

1. Trích xuất hoặc chọn khung hình đầu tiên từ video nguồn.
2. Chỉnh sửa khung hình đó thành giao diện mục tiêu bằng editor/model ảnh riêng biệt, chẳng hạn như Nano Banana, ChatGPT image editing, hoặc Qwen Image.
3. Giữ pose và framing gần với khung nguồn.
4. Sử dụng kích thước Scale 2 thực tế:
   - Ngang: `896 x 512`
   - Dọc: `512 x 896`

Transcript khuyến nghị sử dụng khung hình đầu tiên đã chỉnh sửa này làm ảnh tham chiếu khi thay thế nhân vật hoặc một vùng cụ thể.

## Khu vực Model

Transcript đề cập đến stack model này:

- Model Scale 2:
  - Ưu tiên FP8 nếu có.
  - Q8 được đề cập, nhưng presenter chuyển sang FP8.
- LoRA:
  - `LyteX 2v mis 2v 64 rank distill auto`
  - Các lựa chọn thay thế VRAM thấp hơn: rank `32` hoặc `16`.
- Text encoder:
  - `umt5_xxl_fp8` / `UMT5 XXL FP8`
- VAE:
  - `Wan 2.1 VAE`

Presenter cũng đề cập đến ComfyOrg DPO LoRA có thể cải thiện nhẹ chất lượng chỉnh sửa video, nhưng nói rằng sự khác biệt là nhỏ.

## Cài đặt Sampling

Baseline được đề xuất từ transcript:

- Sampler: `Euler`
- Steps: `8`
- Denoise: `1.0`
- Tùy chọn draft nhanh hơn: `4` steps
- Ví dụ số khung hình: `81` frames

Sử dụng số khung hình thấp hơn và LoRA rank thấp hơn nếu VRAM bị giới hạn.

## Prompting

Giữ prompt đơn giản. Presenter nói không nên mô tả quá mức.

Sử dụng mô tả ngắn gọn về cảnh/hành động, ví dụ:

```text
A person walks forward through the room while raising one arm.
(Một người đi về phía trước qua phòng trong khi giơ một tay lên.)
```

Đối với chỉnh sửa cục bộ, sử dụng text conditioning riêng biệt khi có thể:

- Prompt tham chiếu: mô tả đối tượng hoặc vật thể mới.
- Prompt mask/video: mô tả vùng cần phát hiện hoặc thay thế.

Ví dụ prompt mask mục tiêu:

```text
the person (người)
the shirt (áo sơ mi)
the right arm (cánh tay phải)
the background (nền)
the gauntlet (găng tay giáp)
```

## Masking

Transcript sử dụng SAM 3.1 Multiplex để tạo mask.

Luồng mask điển hình:

1. Load các khung hình video nguồn.
2. Phát hiện vùng mục tiêu bằng SAM 3.1 Multiplex.
3. Đưa mask vào node Scale 2 color mask / driving data.
4. Đưa pose video, mask, ảnh tham chiếu và conditioning vào node Scale 2 video.

Lưu ý về chế độ thay thế từ transcript:

- Đối với thay thế nền, tắt chế độ placement/replacement.
- Đối với thay thế nhân vật, bật các toggle character/person replacement.

Tên toggle chính xác có thể khác nhau tùy thuộc vào phiên bản node bạn cài đặt.

## Luồng Node chính

Đồ thị cấp cao được mô tả trong transcript:

```text
Video nguồn
  -> video frames / first frame
  -> SAM 3.1 mask
  -> Scale 2 color mask driving data

Ảnh tham chiếu khung đầu tiên đã chỉnh sửa
  -> Scale 2 video node

Scale 2 model + LoRA + CLIP + VAE
  -> text conditioning
  -> Scale 2 video node
  -> SamplerCustom
  -> VAE decode
  -> video combine/save
```

Node chính được đặt tên trong transcript là `Scale 2 Video`. Nó nhận source pose/video, dữ liệu masking, ảnh tham chiếu và conditioning, sau đó xuất dữ liệu latent cho `SamplerCustom`.

## Compositing cho chỉnh sửa cục bộ

Nếu chỉ một đối tượng hoặc bộ phận cơ thể cần thay đổi, composite vùng được tạo trở lại lên video gốc.

Lý do:

- Sampling toàn video có thể thay đổi khuôn mặt, nền và các vùng khác.
- Sử dụng cùng mask sau sampling cho phép bạn giữ nguyên video gốc ở mọi nơi ngoại trừ vùng đã chỉnh sửa.

Luồng:

```text
Khung hình video gốc
Khung hình video được tạo
SAM mask
  -> composite vùng masked được tạo lên khung hình gốc
  -> video cuối cùng
```

Điều này hữu ích cho các chỉnh sửa như chỉ thay thế cánh tay, găng tay giáp, áo sơ mi hoặc đạo cụ nhỏ.

## Mở rộng Video dài hơn

Transcript mô tả việc sử dụng các node vòng lặp ComfyUI Easy Use:

- `For Loop Start`
- `For Loop End`

Ý tưởng phân đoạn (chunking):

1. Tạo chunk đầu tiên, chẳng hạn như `49` hoặc `81` frames.
2. Gửi output video đã tạo vào giá trị khởi tạo vòng lặp 1.
3. Gửi `video_frame_offset` từ node Scale 2 video vào giá trị khởi tạo vòng lặp 2.
4. Đưa các khung hình trước đó trở lại vào node Scale 2 video.
5. Đưa video frame offset trở lại vào node Scale 2 video.
6. Hợp nhất mỗi output vòng lặp vào video cuối cùng.

Công thức số lần lặp từ transcript:

```text
tổng_số_khung_nguồn / số_khung_mỗi_chunk
```

Ví dụ:

```text
300 tổng số khung / 49 khung mỗi chunk = khoảng 7 lần lặp
```

Điều này tránh load/tạo toàn bộ video cùng lúc và giúp ngăn chặn lỗi hết bộ nhớ.

## Giá trị mặc định thực tế

Bắt đầu với:

- Độ phân giải: `896 x 512` hoặc `512 x 896`
- Số khung hình: `49` hoặc `81`
- Steps: `8`
- Sampler: `Euler`
- Denoise: `1.0`
- Model mask: `SAM 3.1 Multiplex`
- Sử dụng compositing cho chỉnh sửa cục bộ.
- Chỉ sử dụng mở rộng vòng lặp sau khi chỉnh sửa cơ bản 5 giây hoạt động.

## Khắc phục sự cố

- Nếu toàn bộ khung hình thay đổi bất ngờ, thêm compositing với video gốc.
- Nếu vùng sai thay đổi, tinh chỉnh prompt mask và xem trước SAM mask trước khi sampling.
- Nếu chuyển động trôi dạt, đảm bảo khung tham chiếu khớp với pose và framing khung nguồn.
- Nếu hết VRAM, giảm số khung hình, sử dụng LoRA rank thấp hơn, giảm độ phân giải hoặc sử dụng các chunk vòng lặp.
- Nếu thay thế nhân vật yếu, cải thiện khung hình đầu tiên đã chỉnh sửa và giữ pose đối tượng gần hơn với nguồn.
