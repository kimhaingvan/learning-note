# Lộ trình học Regex thực chiến cho backend với Go

> Bản thiết kế curriculum trước khi bắt đầu Module 1. Lộ trình đi từ mô hình “Regex mô tả một tập chuỗi” đến validation, extraction, xử lý log và quyết định khi nào nên dùng code Go thay cho Regex.

## Hợp đồng học tập

- **Dành cho:** backend developer biết Go nhưng mới với Regex.
- **Đích đến:** đọc, viết, kiểm thử và review Regex dùng được trong công việc backend; phân biệt rõ matching, validation, extraction và business validation.
- **Dự án xuyên suốt:** phát triển bộ xử lý payment order code bắt đầu từ `PAY-20260720-123456`.
- **Môi trường:** Go `regexp`, cú pháp RE2; ví dụ ưu tiên raw string literal của Go.
- **Đã đối chiếu:** Go `regexp` và `regexp/syntax` phiên bản tài liệu `go1.26.5`, RE2 Syntax, ngày 2026-07-20. Máy cục bộ có Go `go1.26.2 darwin/arm64`.
- **Trạng thái hiện tại:** chưa bắt đầu Module 1; cần làm bài kiểm tra đầu vào ở cuối tài liệu trước.
- **Ngoài phạm vi cốt lõi:** tự cài đặt Regex engine, lý thuyết automata chi tiết, POSIX ERE chuyên sâu và các tính năng Go không hỗ trợ như lookaround hoặc backreference.

> **Trạng thái nguồn:** ba nguồn do người học cung cấp đã được audit ngày 2026-07-20. Link thứ tư chưa được cung cấp. Vì ba nguồn này không đặc tả Go `regexp`, S1–S4 vẫn là nguồn sự thật; S5–S7 đóng vai trò công cụ thực hành hoặc tài liệu bổ sung.

## 1. Source map

Ký hiệu nguồn được dùng xuyên suốt roadmap:

- **S1 — Go `regexp`:** <https://pkg.go.dev/regexp>
- **S2 — Go `regexp/syntax`:** <https://pkg.go.dev/regexp/syntax>
- **S3 — RE2 Syntax:** <https://github.com/google/re2/wiki/Syntax>
- **S4 — Regular Expression Matching Can Be Simple And Fast:** <https://swtch.com/~rsc/regexp/regexp1.html>
- **S5 — regex101:** <https://regex101.com/>
- **S6 — RegExr:** <https://regexr.com/>
- **S7 — GeeksforGeeks, Regex Tutorial — How to write Regular Expressions:** <https://www.geeksforgeeks.org/dsa/write-regular-expressions/>

### 1.1 Vai trò và giả định của từng nguồn

| Nguồn | Loại | Dùng để làm gì trong curriculum | Kiến thức nguồn ngầm giả định |
|---|---|---|---|
| S1 | Reference chính thức | Hành vi package, API compile/match/find/submatch/replace, linear-time guarantee | Biết Go cơ bản, string, slice, error, method |
| S2 | Specification/reference chính thức | Cú pháp chính xác mà Go chấp nhận | Biết cách đọc ký hiệu Regex; không phải tutorial |
| S3 | Specification/reference của RE2 | So sánh feature được/không được RE2 hỗ trợ | Đã biết Regex cơ bản; hiểu các engine có thể khác nhau |
| S4 | Explanation của tác giả Go/RE2 | Mental model tập chuỗi, automata, backtracking và lý do giới hạn của RE2 | Phần sau giả định biết C, thuật toán và độ phức tạp |
| S5 | Công cụ thực hành tương tác | Dự đoán rồi quan sát full match, capture và test suite với flavor **Golang** | Phải chủ động chọn đúng flavor; explanation tự sinh không thay thế đặc tả |
| S6 | Công cụ học và thực hành tương tác | Quan sát highlight, capture và test suite khi so sánh engine | Chỉ hỗ trợ JavaScript và PHP/PCRE, không hỗ trợ Go/RE2 |
| S7 | Tutorial cộng đồng đa ngôn ngữ | Preview ví dụ filename và ôn các phần tử Regex phổ biến | Ví dụ dùng C++, C, Java, Python, C# và JavaScript; không có Go và không tách rõ engine |

### 1.2 Phân loại section của nguồn

| Nguồn và section | Phân loại học | Ghi chú sử dụng |
|---|---|---|
| S4 — **Regular Expressions** | Bắt buộc cho người mới | Mental model, literal, nối tiếp, alternation, repetition và precedence |
| S1 — **Overview**, đoạn đầu | Bắt buộc cho người mới | Go dùng RE2, UTF-8 code points, thời gian tuyến tính theo kích thước input |
| S2 — **Single characters**, **Composites**, **Repetitions**, **Grouping**, **Empty strings** | Bắt buộc, đọc theo module | Không đọc toàn bộ cùng một lượt |
| S2 — **Escape sequences**, **Character class elements**, **Perl character classes** | Bắt buộc một phần, còn lại tra cứu | Chú ý `\d`, `\s`, `\w` là ASCII-only |
| S1 — **Example (Package)**, `Compile`, `MustCompile`, `MatchString` | Bắt buộc khi vào Go | Phân biệt compile-time configuration với input runtime |
| S1 — `FindString`, `FindAllString`, `FindStringSubmatch`, `FindAllStringSubmatch` | Bắt buộc cho tìm kiếm và extraction | Phần tử 0 là full match; group bắt đầu từ 1 |
| S1 — `ReplaceAllString`, `ReplaceAllStringFunc`, `QuoteMeta` | Bắt buộc theo use case | Học sau matching và capture |
| S1 — các API `[]byte`, `Reader`, `Index`, `Expand`, `Split` | Tra cứu hoặc học sau | Chưa cần cho project ban đầu |
| S1 — `CompilePOSIX`, `MustCompilePOSIX`, `Longest` | Nâng cao | Semantics khác lộ trình RE2 mặc định |
| S2 — Unicode classes, flags, named captures | Nên học sau | Chỉ đưa vào khi requirement cần Unicode, multiline hoặc maintainability |
| S2 — parser/compiler types và functions | Nâng cao/tra cứu | Dành cho người xây tooling, không cần để dùng `regexp` |
| S3 — toàn bộ bảng **NOT SUPPORTED** | Tra cứu quan trọng | Dùng để kiểm tra pattern lấy từ JavaScript, Python hoặc PCRE |
| S4 — **Finite Automata**, **Regular Expression Search Algorithms** | Nên học sau | Giải thích giới hạn và hiệu năng của Go |
| S4 — các section **Implementation...**, **Caching...** | Nâng cao, bỏ lần đầu | C code và chi tiết cài đặt không phục vụ mục tiêu ứng dụng ban đầu |
| S4 — benchmark năm 2007 | Explanation lịch sử, không dùng làm benchmark hiện tại | Dùng để hiểu xu hướng thuật toán, không suy rộng số đo sang production hiện nay |
| S5 — **Editor** với flavor **Golang**, **Explanation**, **Match Information**, **Unit Tests** | Công cụ thực hành từ M1 | Dự đoán trước, sau đó dùng highlight/output để kiểm tra; cuối cùng xác nhận bằng test Go |
| S5 — **Quick Reference** | Chỉ tra cứu | Nội dung phụ thuộc flavor; không học tuần tự và không dùng thay S2/S3 |
| S5 — **Community Patterns**, **Code Generator**, **Benchmark Regex** | Bỏ lần đọc đầu | Pattern cộng đồng không phải nguồn đã thẩm định; code/benchmark web không chứng minh behavior production |
| S6 — **Expression**, **Details**, **Explain**, **Tests** | Thực hành so sánh ở M15 | Có thể minh họa khác biệt engine; không dùng để xác nhận pattern Go |
| S6 — **Cheatsheet**, **Reference**, **Community Patterns** | Tra cứu engine khác | Có backreference và lookaround mà Go không hỗ trợ |
| S7 — mở bài và ví dụ **Match a Filename Ending with .jpg, .png, or .gif** | Nên đọc sau M7 | Ví dụ tích hợp class, quantifier, escape, group, alternation và anchor; không copy API ngôn ngữ sang Go |
| S7 — **Importance of Regular Expression** | Đọc lướt ở M1 | Cung cấp use case, không giải thích contract hoặc giới hạn |
| S7 — **Common Elements Used in Regular Expressions** | Đọc chọn lọc M2–M8 | Dùng làm ví dụ bổ sung; mọi semantics phải đối chiếu S2 |
| S7 — các tab code C++/C/Java/Python/C#/JavaScript | Không áp dụng trực tiếp cho Go | Có API, string escaping và engine semantics khác nhau; bỏ trong lần đọc đầu |

### 1.3 Audit ba nguồn do người học cung cấp

#### S5 — regex101

- **Kết luận:** dùng được làm playground chính vì có flavor **Golang**, test suite, explanation và match information.
- **Bắt buộc trước mỗi bài:** kiểm tra flavor đang là **Golang**; trang editor mặc định có thể mở ở PCRE2.
- **Phần nên dùng:** Editor, Explanation, Match Information và Unit Tests.
- **Phần chỉ tra cứu:** Quick Reference sau khi đã chọn Golang.
- **Phần bỏ lần đầu:** Community Patterns, Code Generator, Debugger và Benchmark.
- **Rủi ro:** explanation tự sinh có thể tạo cảm giác đã hiểu; pattern cộng đồng có thể đúng cú pháp nhưng sai requirement. Kết quả cuối cùng vẫn phải chạy bằng package Go `regexp`.

#### S6 — RegExr

- **Kết luận:** không dùng làm playground cho project Go. Trang công bố chỉ hỗ trợ JavaScript và PHP/PCRE.
- **Phần nên dùng:** M15, để quan sát cùng một pattern thay đổi behavior hoặc bị hỗ trợ khác giữa engine.
- **Phần chỉ tra cứu:** Cheatsheet và Reference cho engine đã chọn.
- **Phần không áp dụng cho Go:** backreference `\1`, positive/negative lookahead và các behavior phụ thuộc JavaScript/PCRE.
- **Rủi ro:** một pattern “Passed” trên RegExr chưa nói gì về khả năng compile trong Go.

#### S7 — GeeksforGeeks tutorial

- **Kết luận:** dùng làm tutorial bổ sung, không phải specification và không phải hướng dẫn Go.
- **Phần nên đọc:** định nghĩa mở đầu, ví dụ filename, **Importance of Regular Expression**, rồi từng mục cần thiết trong **Common Elements Used in Regular Expressions**.
- **Phần bỏ lần đầu:** toàn bộ code theo ngôn ngữ khác; chỉ quay lại khi M15 cần so sánh API/escaping.
- **Kiến thức ngầm định:** người đọc đã biết ít nhất một ngôn ngữ trong các tab code và hiểu khác biệt giữa tìm đoạn với kiểm toàn chuỗi.
- **Rủi ro:** các mô tả như “digit”, “word character”, start/end được đơn giản hóa và không nêu flavor; phải đối chiếu S2 trước khi chuyển thành pattern Go.

### 1.4 Thứ tự ưu tiên khi nguồn khác nhau

1. Hành vi API Go: S1.
2. Cú pháp Go chấp nhận: S2.
3. Feature RE2 hỗ trợ hoặc từ chối: S3.
4. Mental model và lý do thiết kế: S4.
5. Playground Golang và test suite để quan sát: S5.
6. Tutorial cộng đồng để bổ sung ví dụ: S7.
7. So sánh JavaScript/PCRE với Go: S6.
8. Ví dụ từ engine khác chỉ được chuyển sang Go sau khi kiểm tra với S2/S3 và test bằng Go.

## 2. Prerequisite map

Bạn không cần biết lý thuyết compiler hoặc automata trước khi bắt đầu.

| Nền tảng | Mức cần thiết | Dùng ở đâu | Cách tự kiểm tra |
|---|---|---|---|
| Go raw string `` `...` `` và interpreted string `"..."` | Bắt buộc | Escaping pattern | Giải thích vì sao `` `\d+` `` khác `"\\d+"` ở source Go nhưng tạo cùng pattern |
| `string`, slice và index | Bắt buộc | Full match, submatch, API trả slice | Đọc được `matches[0]`, `matches[1]` và kiểm tra độ dài trước khi truy cập |
| `error` và panic | Bắt buộc | `Compile` so với `MustCompile` | Nêu được pattern động nên trả error, pattern hằng có thể fail-fast |
| Function, method, package và test table | Bắt buộc | `regexp.Regexp` và unit test | Viết được một table-driven test có input và `want bool` |
| Unicode/UTF-8 ở mức code point và byte | Nên có | `.`/class, index API | Biết byte index không luôn bằng vị trí ký tự người dùng nhìn thấy |
| Business rule của payment code | Cung cấp trong bài | Phân biệt format và semantics | Biết `20260231` đúng hình thức `YYYYMMDD` nhưng không phải ngày hợp lệ |

## 3. Dependency map

```text
Requirement + positive/negative examples
                 │
                 ▼
      Literal + substring matching
                 │
        ┌────────┴────────┐
        ▼                 ▼
 Character class     Escape/metacharacter
        │                 │
        └────────┬────────┘
                 ▼
      Quantifier và phạm vi áp dụng
                 │
                 ▼
      Anchor và whole-string validation
                 │
        ┌────────┴────────┐
        ▼                 ▼
 Group/capture       Alternation/optional
        └────────┬────────┘
                 ▼
 Matching vs validation vs extraction
                 │
                 ▼
 Go API + table-driven tests + debugging
                 │
        ┌────────┴────────┐
        ▼                 ▼
 Log/masking       Format vs semantics
        └────────┬────────┘
                 ▼
 Maintainability + engine limits + non-Regex alternatives
```

Nút thắt nhận thức chính là: `MatchString` trả `true` khi **có một đoạn** khớp, không mặc nhiên có nghĩa toàn bộ input hợp lệ. Anchor và test âm phải được học trước khi gọi một pattern là validator.

## 4. Roadmap theo module

Mỗi module dự kiến 25–45 phút. “Sản phẩm” là bằng chứng quan sát được để quyết định có chuyển module hay chưa.

| # | Module | Kết quả học tập quan sát được | Requirement mới của project | Phụ thuộc | Sản phẩm/checkpoint |
|---:|---|---|---|---|---|
| 1 | Regex là tập chuỗi và phép tìm đoạn | Đọc literal từ trái sang phải; chỉ đúng phần được match | Tìm `PAY` trong text | Không | Test matrix cho `PAY`; giải thích substring match |
| 2 | Character class | Mô tả đúng một vị trí ký tự từ một tập cho phép | Nhận diện một chữ số trong code | M1 | Tự viết và giải thích class chữ số |
| 3 | Shorthand và ASCII/Unicode | Chọn `\d`, `\w`, `\s` hoặc class tường minh có chủ đích | Nhận diện các đoạn số | M2 | Bảng so sánh shorthand với class tường minh trong Go |
| 4 | Quantifier và phạm vi áp dụng | Gắn `{n}`, `?`, `+`, `*` vào đúng atom/group | Order ID có đúng sáu chữ số | M2–3 | Pattern cho sáu chữ số cùng boundary tests |
| 5 | Anchor và validation toàn chuỗi | Chứng minh pattern kiểm tra cả input | Validate `PAY-20260720-123456` theo format | M1–4 | Validator có test prefix/suffix rác |
| 6 | Group và capture | Phân biệt full match với từng capture | Trích xuất ngày và order ID | M5 | Go result chứa full/date/order ID |
| 7 | Alternation và precedence | Giới hạn `|` bằng group đúng phạm vi | Prefix là `PAY` hoặc `REF` | M6 | Pattern không vô tình anchor một nhánh duy nhất |
| 8 | Optional pattern | Làm một **group** trở thành tùy chọn | Thêm suffix `-R1` tùy chọn | M4, M6–7 | Test có/không suffix và suffix dở dang |
| 9 | Escaping ở hai tầng | Phân biệt escape của Regex và escape của Go string | Match dấu `.` hoặc prefix/suffix lấy từ cấu hình | M1–8 | So sánh raw/interpreted string; dùng `QuoteMeta` đúng chỗ |
| 10 | Matching, validation, extraction | Chọn đúng operation và API theo mục tiêu | Tìm code trong log nhưng validate input API toàn chuỗi | M5–9 | Cùng pattern family, ba contract và ba test matrix |
| 11 | Go `regexp` trong ứng dụng | Compile, reuse, match, find, capture và replace an toàn | Parse nhiều code trong log; mask order ID | M10 | Package Go nhỏ với API rõ và table-driven tests |
| 12 | Testing và debug độ rộng | Tìm counterexample cho pattern quá rộng/quá hẹp | Xử lý input sai và gần đúng | M11 | Bộ positive/negative/edge cases làm lộ ít nhất hai lỗi |
| 13 | Format so với semantics | Tách kiểm tra cấu trúc khỏi ngày/business rule | Từ chối `20260231`, prefix hoặc suffix trái rule | M11–12 | Regex + `time.Parse`/code Go, test nêu rõ lớp lỗi |
| 14 | Readability và maintainability | Giảm capture thừa, đặt tên, chia rule và ghi contract | Review pattern production của project | M6–13 | Bản review theo rubric và phiên bản dễ bảo trì hơn |
| 15 | Engine differences, giới hạn và khi không dùng Regex | Phát hiện pattern sai engine và chọn parser/code khi hợp lý | So sánh giải pháp Regex với Go parser | M1–14 | Decision record giải thích lựa chọn và trade-off |

## 5. Thiết kế chi tiết từng module

Phần này xác định đủ thành phần để mở từng module theo quy trình Preview → Predict → Read → Explain → Build → Practice → Verify → Record. Worked example chỉ nêu **bài toán đích**, chưa đưa pattern hoàn chỉnh để không làm lộ đáp án trước khi học.

### Module 1 — Regex là tập chuỗi và phép tìm đoạn

- **Đọc:** S4 **Regular Expressions**, từ đầu đến trước **Finite Automata**; S1 **Overview**, câu mô tả search; bỏ benchmark và implementation.
- **Câu hỏi định hướng:** “Pattern `PAY` nói gì về vị trí của `PAY` trong input?”
- **Mental model:** pattern mô tả một tập chuỗi; engine tìm một đoạn thuộc tập đó trong input trừ khi contract thêm boundary.
- **Worked example đích:** tìm `PAY` trong `status=ok code=PAY-20260720-123456`.
- **Test seeds:** phải match `PAY`; không match `REF`; edge `PREPAYMENT` phải nêu phần `PAY` được match.
- **Thực hành giảm dần:** completion `P_Y`; tự làm tìm literal `REF`; chuyển giao tìm marker `ERROR` trong log.
- **Common mistake:** diễn giải “có match” thành “toàn bộ input hợp lệ”.
- **Retrieval:** literal là gì? Pattern bắt đầu tìm ở đâu? Full match hiện tại là đoạn nào?
- **Hoàn thành khi:** tự thêm một positive, một negative, một edge case và đọc pattern từ trái sang phải.

### Module 2 — Character class

- **Đọc:** S2 **Single characters** và phần đầu **Character class elements**; bỏ Unicode classes.
- **Câu hỏi định hướng:** “Một character class tiêu thụ bao nhiêu vị trí trong input?”
- **Mental model:** `[abc]` là một lựa chọn cho **một** vị trí, không phải chuỗi `abc`.
- **Worked example đích:** tìm vị trí chữ số đầu tiên trong payment code.
- **Test seeds:** `7` phải match; `A` không match; edge `７` (fullwidth digit) buộc làm rõ ASCII/Unicode.
- **Thực hành:** điền range trong `[0-_]`; tự tạo class prefix letter; chuyển giao match severity `I`, `W`, `E`.
- **Go:** `FindString` để quan sát đoạn đầu tiên được match.
- **Common mistake:** dùng `[PAY]` để biểu diễn literal `PAY`.
- **Retrieval/hoàn thành:** giải thích class, range, negated class và đưa counterexample cho `[PAY]`.

### Module 3 — Shorthand character class

- **Đọc:** S2 **Perl character classes**; S3 **Perl character classes**; chỉ đọc `\d`, `\D`, `\s`, `\S`, `\w`, `\W`.
- **Câu hỏi định hướng:** “Trong Go, `\d` có match mọi chữ số Unicode không?”
- **Mental model:** shorthand là tên rút gọn cho một tập ký tự cụ thể; trong Go, các shorthand trên là ASCII-only.
- **Worked example đích:** tìm chuỗi chữ số trong order code.
- **Test seeds:** `123` phải match; `ABC` không match; edge `１２３` không được giả định là `\d`.
- **Thực hành:** completion chọn `\_`; tự thay shorthand bằng class tương đương; chuyển giao tách số latency trong `latency=42ms`.
- **Common mistake:** hiểu `\w` là mọi “từ” tự nhiên hoặc Unicode letter.
- **Hoàn thành khi:** chọn class tường minh thay shorthand khi contract yêu cầu ASCII rõ ràng.

### Module 4 — Quantifier và phạm vi áp dụng

- **Đọc:** S2 **Repetitions**; S4 **Regular Expressions** đoạn repetition và precedence; bỏ repetition count trên 1000 như chi tiết tra cứu đến khi cần.
- **Câu hỏi định hướng:** “Trong `AB{3}`, phần nào được lặp ba lần?”
- **Mental model:** quantifier gắn với atom đứng ngay trước nó; group biến nhiều atom thành một atom.
- **Worked example đích:** order ID gồm đúng sáu chữ số.
- **Test seeds:** `123456` phải match; `12345` không match toàn bộ; edge `x1234567y` làm lộ substring nếu chưa có anchor.
- **Thực hành:** điền số trong `[0-9]{_}`; tự làm ngày tám chữ số; chuyển giao trace ID 16 hex chars.
- **Common mistake:** dùng `+` khi requirement nói “đúng sáu”; quên xác định quantifier áp dụng cho gì.
- **Hoàn thành khi:** giải thích được từng repetition rule và dự đoán ba test trước khi chạy.

### Module 5 — Anchor và validation toàn chuỗi

- **Đọc:** S2 **Empty strings** (`^`, `$`, `\A`, `\z`); S1 `MatchString`; bỏ multiline flags ở lượt đầu.
- **Câu hỏi định hướng:** “Vì sao `MatchString` có thể trả `true` cho input chứa rác?”
- **Mental model:** anchor khớp vị trí rỗng; nó không tiêu thụ ký tự. Validation cần contract whole-string rõ ràng.
- **Worked example đích:** validate đúng format ban đầu của payment code.
- **Test seeds:** `PAY-20260720-123456` phải match; `xPAY-...` và `PAY-...x` không match; edge newline cuối chuỗi phải kiểm tra theo anchor đã chọn.
- **Thực hành:** completion thêm boundary còn thiếu; tự làm validator reference code; chuyển giao validate request ID.
- **Common mistake:** chỉ anchor đầu hoặc hiểu `$` giống hệt `\z` trong mọi mode/engine.
- **Hoàn thành khi:** chứng minh bằng test rằng prefix và suffix rác đều bị từ chối.

### Module 6 — Group và capture

- **Đọc:** S2 **Grouping**; S1 Overview đoạn quy ước submatch và `FindStringSubmatch`; bỏ named group cho đến cuối module.
- **Câu hỏi định hướng:** “Index 0 và index 1 trong kết quả khác nhau thế nào?”
- **Mental model:** group vừa gom cấu trúc vừa có thể capture; full match luôn ở vị trí 0, capture đánh số theo dấu `(` mở từ trái sang phải.
- **Worked example đích:** trích xuất `20260720` và `123456`.
- **Test seeds:** code hợp lệ trả 3 phần tử; code sai trả `nil`; edge group tùy chọn vắng mặt để dành cho M8.
- **Thực hành:** completion thêm ngoặc cho order ID; tự capture prefix; chuyển giao capture status code trong log.
- **Go:** kiểm tra `match != nil` và độ dài trước khi index.
- **Common mistake:** nghĩ full match là group 1 hoặc dùng capture khi chỉ cần grouping.
- **Hoàn thành khi:** chỉ đúng full match và mọi capture trên output Go dự đoán.

### Module 7 — Alternation và precedence

- **Đọc:** S2 **Composites** và **Grouping**; S4 đoạn alternation, concatenation, precedence.
- **Câu hỏi định hướng:** “`^PAY|REF$` anchor phần nào của mỗi nhánh?”
- **Mental model:** `|` có precedence thấp; group định nghĩa phạm vi lựa chọn.
- **Worked example đích:** prefix `PAY` hoặc `REF` nhưng phần còn lại dùng chung.
- **Test seeds:** cả hai prefix phải match; `OTHER` không match; edge `PAYxxx`/`xxxREF` bắt lỗi scope anchor.
- **Thực hành:** completion đặt group đúng; tự thêm `REV`; chuyển giao severity `INFO|WARN|ERROR`.
- **Common mistake:** lặp toàn pattern hoặc anchor từng nhánh không nhất quán.
- **Hoàn thành khi:** thêm prefix mới mà không đổi phần date/order và không mở rộng boundary.

### Module 8 — Optional pattern

- **Đọc:** S2 **Repetitions** mục `?` và **Grouping**; S4 đoạn zero-or-one.
- **Câu hỏi định hướng:** “Requirement cho phép suffix vắng mặt hay từng ký tự suffix vắng mặt?”
- **Mental model:** `?` áp dụng cho atom/group ngay trước; để cả suffix tùy chọn, gom cả dấu phân cách với nội dung.
- **Worked example đích:** cho phép suffix `-R1` hoặc không có suffix.
- **Test seeds:** code cơ sở và code có suffix phải match; suffix dở dang không match; edge dấu `-` thừa.
- **Thực hành:** completion đặt `?`; tự đổi suffix thành `-R1|-R2`; chuyển giao version suffix `-v2`.
- **Common mistake:** làm mỗi ký tự tùy chọn hoặc để dấu `-` bắt buộc ngoài group.
- **Hoàn thành khi:** diễn giải chính xác tập hai hình dạng được chấp nhận.

### Module 9 — Escaping ở Regex và Go string

- **Đọc:** S2 **Escape sequences** phần metacharacter/octal/hex cơ bản; S1 `QuoteMeta`; bỏ escape hiếm.
- **Câu hỏi định hướng:** “Dấu `\` đang được Go parser hay Regex parser xử lý?”
- **Mental model:** source Go và Regex là hai lớp ngôn ngữ; raw string giảm một lớp escaping nhưng không thay đổi cú pháp Regex.
- **Worked example đích:** match dấu chấm literal và prefix cấu hình chứa metacharacter.
- **Test seeds:** literal `.` phải match; ký tự bất kỳ không được match; edge input cấu hình `A+B` cần `QuoteMeta`.
- **Thực hành:** completion raw string; tự viết interpreted string tương đương; chuyển giao tìm filename `.log` literal.
- **Common mistake:** dùng `.` khi muốn dấu chấm hoặc nối input người dùng trực tiếp vào pattern.
- **Hoàn thành khi:** giải thích được giá trị pattern sau khi Go xử lý string và khi nào gọi `QuoteMeta`.

### Module 10 — Matching, validation và extraction

- **Đọc:** S1 `MatchString`, `FindString`, `FindAllString`, `FindStringSubmatch`; bỏ Index variants.
- **Câu hỏi định hướng:** “Cần câu trả lời boolean, đoạn text hay các field?”
- **Mental model:** cùng cú pháp pattern nhưng operation và contract khác: search tìm đoạn, validation kiểm toàn input, extraction trả dữ liệu có cấu trúc.
- **Worked example đích:** log search, API input validation và field extraction cho cùng payment code.
- **Test seeds:** một log có code, input chỉ là code, log không có code; edge nhiều code trong một dòng.
- **Thực hành:** completion chọn API; tự viết ba contract; chuyển giao xử lý trace ID.
- **Common mistake:** dùng `MatchString` rồi cắt chuỗi bằng offset hard-code.
- **Hoàn thành khi:** chọn đúng API và test matrix cho từng mục tiêu.

### Module 11 — Go `regexp` trong ứng dụng

- **Đọc:** S1 `Compile`, `MustCompile`, `Regexp` concurrency note, `FindAllString(Submatch)`, `ReplaceAllString(Func)`.
- **Câu hỏi định hướng:** “Pattern là hằng đã biết hay dữ liệu động có thể sai?”
- **Mental model:** compile pattern thành `*regexp.Regexp` một lần rồi dùng nhiều lần; chọn API theo shape output.
- **Worked example đích:** parse nhiều order code trong log và mask order ID.
- **Test seeds:** 0/1/nhiều code; malformed code; edge replacement có `$` để phân biệt literal/template semantics.
- **Thực hành:** completion table-driven test; tự viết hàm extract; chuyển giao mask transaction reference.
- **Common mistake:** compile trong hot path, dùng `MustCompile` cho input runtime, không kiểm tra `nil`.
- **Hoàn thành khi:** `go test` pass và giải thích compile policy, output shape, concurrency safety.

### Module 12 — Testing và debug pattern quá rộng/quá hẹp

- **Đọc:** S1 examples liên quan; Go tutorial **Add a test** nếu chưa quen table-driven test: <https://go.dev/doc/tutorial/add-a-test>.
- **Câu hỏi định hướng:** “Counterexample nhỏ nhất làm pattern hiện tại sai là gì?”
- **Mental model:** test là executable contract; positive chống quá hẹp, negative chống quá rộng, edge case kiểm boundary và assumption.
- **Worked example đích:** cố ý review một validator thiếu anchor và một validator sai repetition.
- **Test seeds:** valid chuẩn; rác trước/sau; digit thiếu/thừa; Unicode digit; date hình thức đúng nhưng vô nghĩa.
- **Thực hành:** completion cột `want`; tự làm mutation testing thủ công; chuyển giao test customer reference.
- **Common mistake:** chỉ test happy path hoặc lấy pattern làm nguồn sự thật thay requirement.
- **Hoàn thành khi:** mỗi rule có ít nhất một test chứng minh và lỗi báo rõ case nào sai.

### Module 13 — Format validation và semantic validation

- **Đọc:** không cần cú pháp mới; dùng `time.Parse` reference khi triển khai: <https://pkg.go.dev/time#Parse>.
- **Câu hỏi định hướng:** “Regex biết `20260231` không tồn tại bằng cách nào?”
- **Mental model:** Regex kiểm shape; code Go kiểm ý nghĩa, quan hệ field và business state.
- **Worked example đích:** Regex tách date, `time.Parse("20060102", date)` xác thực lịch, code kiểm business rule.
- **Test seeds:** ngày thật; `20260231`; leap day đúng/sai; prefix hợp format nhưng không được phép theo tenant.
- **Thực hành:** completion call `time.Parse`; tự phân lớp error; chuyển giao validate port `0..65535`.
- **Common mistake:** nhồi lịch Gregorian hoặc range phức tạp vào một pattern khó review.
- **Hoàn thành khi:** mỗi failure được gán đúng lớp format, semantic hoặc business.

### Module 14 — Readability và maintainability

- **Đọc:** ôn S1/S2 phần đã dùng; S1 `SubexpNames`/`SubexpIndex` nếu named capture cải thiện contract.
- **Câu hỏi định hướng:** “Người sửa code sau sáu tháng sẽ biết rule nào đang được bảo vệ bằng test nào không?”
- **Mental model:** Regex là code: cần tên, contract, test và phạm vi nhỏ; pattern ngắn hơn không mặc nhiên dễ hiểu hơn.
- **Worked example đích:** review một pattern dồn mọi format/semantic rule vào một dòng.
- **Test seeds:** giữ nguyên behavior trước/sau refactor; edge capture numbering thay đổi.
- **Thực hành:** completion tên constant; tự viết comment theo requirement; chuyển giao review log parser.
- **Common mistake:** capture mọi group, nối fragment mơ hồ hoặc comment diễn giải sai pattern.
- **Hoàn thành khi:** pattern và tests cho phép reviewer truy ngược từng rule về requirement.

### Module 15 — Engine differences, giới hạn và khi không dùng Regex

- **Đọc:** S3 các mục **NOT SUPPORTED**; S1 Overview về RE2/linear time; S4 **Regular Expression Search Algorithms**. Bỏ C implementation.
- **Câu hỏi định hướng:** “Pattern copy từ PCRE/JavaScript có feature nào Go sẽ từ chối?”
- **Mental model:** Regex là một họ engine; Go chọn RE2 và tính dự đoán thời gian, nên không hỗ trợ các feature cần backtracking như lookaround/backreference.
- **Worked example đích:** thay một solution dùng lookbehind bằng capture/code Go; so sánh parser thủ tục cho payment code.
- **Test seeds:** pattern Go hợp lệ; lookahead/lookbehind/backreference compile error; edge nested/escaped structured data.
- **Thực hành:** completion xác định feature sai engine; tự đề xuất code alternative; chuyển giao chọn JSON parser thay Regex.
- **Common mistake:** tin mọi pattern trên regex tester đều chạy trong Go hoặc dùng Regex parse HTML/JSON tùy ý.
- **Hoàn thành khi:** viết decision record nêu requirement, lựa chọn, giới hạn, test và phương án không Regex.

## 6. Mapping module ↔ source section

S1–S4 quyết định nội dung đúng; S5–S7 cung cấp hoạt động quan sát hoặc ví dụ bổ sung.

| Module | Nguồn sự thật cần đọc | S5 regex101 | S6 RegExr | S7 GeeksforGeeks | Nguồn bổ sung |
|---:|---|---|---|---|---|
| 1 | S1 Overview; S2 Single characters; S4 Regular Expressions | Golang: Editor, Explanation, Match Information | — | Mở bài; Importance | — |
| 2 | S1 `FindString`; S2/S3 Character class elements | Golang: Editor | — | Common Elements: Character Classes; Negated Character Class | — |
| 3 | S2/S3 Perl character classes | Golang: Quick Reference sau khi dự đoán | — | Common Elements: Character Classes | — |
| 4 | S1 `MatchString`; S2/S3 Repetitions; S4 Regular Expressions | Golang: Editor, Unit Tests | — | Common Elements: Repeaters, `*`, `+`, `{}` | — |
| 5 | S1 `MatchString`; S2/S3 Empty strings | Golang: Unit Tests | — | Common Elements: caret, dollar | — |
| 6 | S1 submatches/`FindStringSubmatch`; S2/S3 Grouping | Golang: Match Information | — | Ví dụ filename: capture group | — |
| 7 | S2/S3 Composites/Grouping; S4 Regular Expressions | Golang: Explanation | — | Ví dụ filename: alternation | — |
| 8 | S2/S3 Repetitions/Grouping; S4 Regular Expressions | Golang: Unit Tests | — | Common Elements: optional `?` | — |
| 9 | S1 `QuoteMeta`; S2/S3 Escape sequences; S4 Regular Expressions | Golang: Explanation | — | Ví dụ filename: escaped dot | — |
| 10 | S1 Match/Find/FindAll/Submatch APIs | Golang: Match Information, Unit Tests | — | — | — |
| 11 | S1 Compile/MustCompile/Regexp/FindAll/Replace | Dùng để thử trước khi chuyển sang Go test | — | Bỏ code đa ngôn ngữ | — |
| 12 | S1 Examples | Golang: Unit Tests | — | — | Go: Add a test |
| 13 | Không có syntax mới | — | — | — | Go `time.Parse` |
| 14 | S1 `SubexpNames`/`SubexpIndex`; S2/S3 named grouping | Golang: Explanation, Format Regex | — | — | — |
| 15 | S1 Overview; S2 exclusions; S3 NOT SUPPORTED; S4 Search Algorithms | So sánh Golang với PCRE2 | So sánh JavaScript/PCRE; xem Tests/Details | Xem lại code đa ngôn ngữ có chọn lọc | — |

## 7. Phần chỉ dùng để tra cứu

Không đọc tuần tự các phần này trong lượt đầu:

- Toàn bộ API matrix `Find(All)?(String)?(Submatch)?(Index)?`; chỉ mở biến thể khi output cần đúng shape đó.
- `Reader` và `[]byte` variants.
- `Expand`, `ExpandString`, `Split`, marshal/unmarshal text, `LiteralPrefix`.
- `CompilePOSIX`, `MustCompilePOSIX`, `Longest` và POSIX semantics.
- Parser/AST/compiler internals trong `regexp/syntax`.
- Danh sách đầy đủ Unicode scripts/categories và ASCII named classes.
- Escape octal/hex hiếm, flags nâng cao, lazy quantifier và repetition limit.
- RE2 syntax của các engine/tool khác được đánh dấu **NOT SUPPORTED**.
- C implementation, NFA-to-DFA caching và benchmark chi tiết trong S4.
- S5 Quick Reference ngoài các token đang học; Community Patterns, Code Generator và Benchmark.
- S6 Cheatsheet/Reference ngoài hoạt động so sánh M15; toàn bộ Community Patterns.
- S7 code C++/C/Java/Python/C#/JavaScript trong lần đọc đầu.

## 8. Khác biệt cần chú ý giữa Go và engine khác

| Chủ đề | Go `regexp` / RE2 | JavaScript, Python hoặc PCRE có thể khác | Hệ quả học tập |
|---|---|---|---|
| Lookahead/lookbehind | Không hỗ trợ | Nhiều engine hỗ trợ một phần hoặc toàn bộ | Dùng capture, nhiều bước hoặc code Go |
| Backreference trong pattern (`\1`) | Không hỗ trợ | Python/PCRE/JavaScript thường hỗ trợ | Không copy pattern dựa vào lặp lại capture |
| Conditional/atomic group/possessive quantifier | Không hỗ trợ | PCRE/Python/Java tùy feature | Kiểm S2/S3 trước khi dùng |
| `\d`, `\w`, `\s` | Các Perl class này là ASCII-only | Unicode behavior/default/flag khác theo engine | Contract Unicode phải dùng class Unicode hoặc code rõ ràng |
| Anchor | Có `^`, `$`, `\A`, `\z`; flag `m` đổi line behavior | Tên anchor và newline semantics khác | Với validation, test cả newline và rác hai đầu |
| Named capture | Go hỗ trợ `(?P<name>...)` và `(?<name>...)` theo S2 | Cú pháp truy cập/thay thế khác | Không trộn syntax replacement giữa engine |
| Matching strategy | Mặc định leftmost-first; có linear-time guarantee theo input size | Backtracking engine có thể có pathological cases | Lợi về predictability nhưng feature set hẹp hơn |
| `.` và newline | Mặc định không match newline; flag `s` thay đổi | Có engine/tool cấu hình khác | Luôn test multiline log thay vì suy đoán |
| String escaping | Go raw/interpreted strings tạo thêm một lớp cú pháp | JavaScript literal, Python raw string khác | Tách “pattern value” khỏi “cách viết trong source” |
| Whole-string API | `MatchString` kiểm **có chứa match**, không có API tên `FullMatch` | Python có `fullmatch`; Java có semantics API khác | Go validator phải encode boundary trong pattern/contract |

## 9. Rubric đánh giá bài tập Regex

Mỗi bài được chấm 20 điểm. Chỉ chuyển module khi đạt ít nhất **16/20**, đồng thời không được 0 điểm ở tiêu chí Requirement, Boundary hoặc Engine compatibility.

| Tiêu chí | 0 điểm | 1 điểm | 2 điểm |
|---|---|---|---|
| Requirement | Hiểu sai mục tiêu | Đúng phần chính, thiếu constraint | Diễn đạt đủ operation, input và rule |
| Positive cases | Không có/không đúng | Có happy path | Bao phủ từng biến thể hợp lệ |
| Negative cases | Không có | Có nhưng không nhắm rule | Mỗi rule có counterexample gần |
| Edge cases | Không xét | Nêu nhưng chưa dự đoán | Có boundary/Unicode/empty/near-miss phù hợp |
| Đọc trái sang phải | Không giải thích được | Mô tả ký hiệu rời rạc | Giải thích vai trò và phạm vi từng phần |
| Quantifier/group scope | Sai phạm vi | Đúng nhưng lý do mơ hồ | Nêu rõ atom/group bị tác động |
| Boundary/operation | Nhầm substring với full input | Có anchor nhưng thiếu test | Chọn đúng search/validate/extract và chứng minh bằng test |
| Go/engine compatibility | Dùng feature Go không hỗ trợ | Chạy được nhưng trộn assumption engine | Đúng RE2 và đúng escaping Go |
| Format vs semantics | Nhồi hoặc bỏ business rule | Phân biệt được nhưng chưa tách code | Tách rõ Regex, parse và business validation |
| Maintainability | Pattern khó hiểu, không test | Chạy đúng nhưng contract yếu | Pattern nhỏ, tên rõ, test truy vết được rule |

Mức kết luận khi review:

- **18–20:** đúng, rõ, có khả năng chuyển giao.
- **16–17:** đạt module; ghi một điểm cần củng cố.
- **12–15:** đúng một phần; sửa test/mental model rồi làm lại.
- **0–11:** quá rộng, quá hẹp, sai engine hoặc chưa bám requirement; quay lại bước Build.

## 10. Template learning checkpoint

```markdown
## Learning checkpoint — [Module/section]

- **Section đã hoàn thành:**
- **Requirement đã xử lý:**
- **Cú pháp mới:**
- **Mental model tôi có thể tự giải thích:**
- **Misconception đã sửa:**
- **Regex tôi tự viết:**
- **Đọc từ trái sang phải:**
- **Full match và capture groups:**
- **Positive cases đã chạy:**
- **Negative cases đã chạy:**
- **Edge cases đã chạy:**
- **Kết quả dự đoán trước khi chạy / kết quả thật:**
- **Pattern đang match substring hay toàn bộ string:**
- **Format rule / semantic rule / business rule:**
- **Phần còn mơ hồ:**
- **Điểm rubric:** /20
- **Quyết định:** học tiếp / luyện lại
- **Section tiếp theo và lý do:**
```

## 11. Bài kiểm tra đầu vào

Không chạy code hoặc tra cheat sheet ở lượt đầu. Mục tiêu là xác định mental model hiện tại, không phải lấy điểm cao. Trả lời theo số 1–10 và ghi `không chắc` nếu cần. **Đáp án chưa được đưa trong tài liệu này.**

### Câu 1 — Literal và phần được match

Với pattern `PAY` và input `PREPAYMENT`, hãy trả lời:

1. Có match không?
2. Nếu có, chính xác đoạn nào là full match?
3. Kết quả đó có chứng minh toàn bộ input là payment code hợp lệ không?

### Câu 2 — Character class

Không chạy code. Pattern `[PAY]` match những input nào dưới đây, và mỗi lần match bao nhiêu ký tự?

- `P`
- `PAY`
- `A`
- `REF`

Sau đó giải thích `[PAY]` khác `PAY` thế nào.

### Câu 3 — Quantifier scope

Đọc pattern `AB{3}` từ trái sang phải. Pattern này mô tả `ABABAB`, `ABBB`, hay một tập khác? Quantifier `{3}` áp dụng cho phần nào?

### Câu 4 — Dự đoán substring match

Giả sử Go chạy:

```go
re := regexp.MustCompile(`[0-9]{6}`)
re.MatchString("id=1234567")
```

Hãy dự đoán boolean trả về, đoạn có thể được match và lý do. Không sửa pattern ở câu này.

### Câu 5 — Whole-string validation

Requirement: “Toàn bộ input phải có đúng dạng `PAY-` theo sau bởi sáu chữ số.”

Hãy viết trước:

- hai positive cases;
- ba negative cases, trong đó có một case chứa một payment code hợp lệ nhưng có rác ở ngoài.

Sau đó mới thử viết pattern. Nếu chưa biết anchor, hãy mô tả bằng ngôn ngữ tự nhiên phần còn thiếu.

### Câu 6 — Full match và capture

Với pattern `(PAY|REF)-([0-9]{6})` và input `REF-123456`, hãy dự đoán:

- full match;
- capture group 1;
- capture group 2.

Giải thích thứ tự đánh số group.

### Câu 7 — Alternation và precedence

So sánh hai pattern:

```text
^PAY|REF$
^(PAY|REF)$
```

Bạn dự đoán chúng có cùng nghĩa không? Hãy dùng `PAYxxx` hoặc `xxxREF` để giải thích dự đoán.

### Câu 8 — Escaping trong Go

Hai cách viết dưới đây có dự định tạo cùng một Regex “một hoặc nhiều chữ số”:

```go
regexp.MustCompile(`\d+`)
regexp.MustCompile("\\d+")
```

Bạn nghĩ chúng có tương đương không? Dấu `\` được xử lý qua những lớp nào?

### Câu 9 — Format và semantics

Input `PAY-20260231-123456` đúng shape `PAY-YYYYMMDD-NNNNNN`, nhưng ngày không tồn tại.

Hãy phân loại phần nào nên do Regex kiểm tra, phần nào nên do code Go kiểm tra, và nêu lý do.

### Câu 10 — Go engine và lựa chọn công cụ

Một đồng nghiệp gửi pattern dùng lookbehind `(?<=order=)[0-9]+` từ một Regex tester.

1. Bạn sẽ kiểm tra điều gì trước khi đưa vào Go?
2. Nếu Go từ chối pattern, bạn đề xuất hướng giải quyết nào mà chưa cần viết pattern hoàn chỉnh?
3. Nêu một loại dữ liệu có cấu trúc mà bạn sẽ ưu tiên parser chuyên dụng thay vì Regex.

## 12. Cách gửi câu trả lời

Gửi câu trả lời 1–10 trong một message. Với mỗi câu, ưu tiên ghi dự đoán và lý do; không cần cố dùng thuật ngữ nếu chưa chắc. Sau đó roadmap sẽ được điều chỉnh theo ba tín hiệu:

1. **Mental model:** có phân biệt substring, full match và capture không.
2. **Construction skill:** có đi từ examples và boundary đến pattern không.
3. **Go transfer:** có phân biệt Regex engine, Go string escaping và semantic validation không.

Chỉ sau bước điều chỉnh đó mới bắt đầu **Module 1 — Regex là tập chuỗi và phép tìm đoạn**.

## Nguồn chính

- Go team, [`regexp` package](https://pkg.go.dev/regexp), tài liệu `go1.26.5`, kiểm tra 2026-07-20.
- Go team, [`regexp/syntax` package](https://pkg.go.dev/regexp/syntax), tài liệu `go1.26.5`, kiểm tra 2026-07-20.
- Google RE2, [Syntax](https://github.com/google/re2/wiki/Syntax), kiểm tra 2026-07-20.
- Russ Cox, [Regular Expression Matching Can Be Simple And Fast](https://swtch.com/~rsc/regexp/regexp1.html), phần mental model và thuật toán.
- [regex101](https://regex101.com/), playground thực hành; luôn chọn flavor Golang.
- [RegExr](https://regexr.com/), playground JavaScript/PHP-PCRE; chỉ dùng để so sánh engine.
- GeeksforGeeks, [Regex Tutorial — How to write Regular Expressions](https://www.geeksforgeeks.org/dsa/write-regular-expressions/), tutorial bổ sung được cập nhật ngày 2025-12-22.
- Go team, [Add a test](https://go.dev/doc/tutorial/add-a-test), nguồn bổ sung cho Module 12.
- Go team, [`time.Parse`](https://pkg.go.dev/time#Parse), nguồn bổ sung cho Module 13.
