# Kiến trúc CDSL Oralce 

## Mô hình tổng quan


> Cấu hình instance

- Mô hình Nonclustered

    + Tạo 2 database D1, D2 có instance I1, I2
    + Giả sử vùng instance chết , database còn I1 liên quan CRM, I2 liên quan ERP
    + Mô hình thường dùng giải pháp: 
    
        * DataGuard (Dùng database chung, trường hợp storage chết dùng 1 storage dự phòng ở nơi khác, vẫn phải backup, có thể replicate)
        * GoldenGate

- Mô hình Clustered

    Có vùng share, cổng kết nối. 

> Kết nối đến Instance CSDL 

- Connection: người dùng và 1 instance

- Session: người dùng với 1 instance thông qua 1 user process ( khi nhập user pass, chạy câu lệnh )

## Cấu trúc bộ nhớ CDSL Oralce

> Program Global Area - PGA 

- Chứa thông tin điều khiển Server Process, về Session làm việc của user, thông tin điều khiển tiến trình background

- Lưu trữ đối tượng tạm như bộ đệm, kết quả trung gian SQL phức tạp

- Kích thước phụ thuộc số lượng user và hoạt động. Khi PGA qua lớn ảnh hưởng hiệu suất

> System Global Area - SGA

Cách bộ nhớ hoạt động theo cách xoay vòng, nếu không sử dụng loại bỏ

- Shared Pool

    + Chứa Library cache: chứa câu lệnh SQL khi truy cập vào => ra được plan câu lệnh
    + Data dictionary cache: Cấu trúc CSDL bảng, user ... khi truy cập có ngay
    + Server result cache: Cache câu lệnh đỡ chạy lại

    Trải qua quá trình phân tích câu lệnh có trong shared pool

        * Có dựa theo plan trả cho user theo index id
        * Chưa có: lấy câu lệnh băm ra rồi phân tích tất cả theo Execution plan(Kế hoạch thực thi). Sau lấy ra được cost(chi phí) tốt nhất trả về theo user dùng câu lệnh này. Quá trình parse phase

- Database Buffer Cache

    + Cách truy cập dữ liệu nhanh nhất cache dữ liệu trên RAM

    + Block là khối nhỏ khi dữ liệu được đọc từ đĩa cứng lên bộ nhớ trung gian như dữ liệu bảng, chỉ mục, hoặc object view, sequence. Quản lý và thay thế block giúp tăng tốc độ truy xuất dữ liệu

    + Là vùng nhớ thuộc SGA trên RAM, lưu bản sao của block đọc từ datafile. Khi cần dữ liệu đọc sẽ có sẵn rồi không cần đọc lại.

    + Dựa theo cơ chế giải thuật least recently used - LRU những dữ liệu thường xuyên truy cập(block) để lại, ít truy cập loại bỏ khỏi bộ đệm

- Redo Log Buffer

    + Lưu các sự thay đổi của một instance: chứa redo entries câu lệnh DML, DDL tạo ra. Bản ghi redo là ngược lại câu lệnh tương ứng như INSERT là DELETE, hay lệnh UPDATE thì giá trị cũ làm trong redo entries

    + DML câu lệnh: INSERT, UPDATE, DELETE

    + DDL câu lệnh: CREATE, ALTER, DROP

    + Lưu trong bộ nhớ, ghi liên tục tối đa mỗi 3s. Có 3 file: current, active, 

- Large Pool

    + Dữ liệu thường truy cập đẩy lên Buffer Cache

    + Vùng chứa dữ liệu lớn, sử dụng bộ nhớ của session cho shared server và giao diện Oracle XA. Chứa hoạt động liên quan I/O và backup restore

    + Giao diện Oracle XA (Transaction Branch - XA) là một giao diện cho phép quản lý và điều phối các giao dịch đa cơ sở dữ liệu trong Oracle.


- Java Pool

    + Lưu java code trong Java Virtual Machine (JVM)

    + Mỗi khi có yêu cầu thực thi mã Java từ ứng dụng Oracle Database sẽ:

        * Tạo một JVM mới trong Java pool
        * Sử dụng JVM này để thực thi mã Java
        * Sau khi thực thi xong, JVM vẫn được giữ lại trong Java pool để phục vụ các lần gọi tiếp theo nếu cần.
    
    + Việc lưu trữ JVM trong Java pool giúp tăng hiệu suất thực thi mã Java bằng cách tránh phải khởi tạo lại JVM mới mỗi lần gọi.

- Streams Pool

    + Lưu message hàng đợi

    + Cấp vùng nhớ cho hoạt động Oracle Streams



## Kiến trúc tiến trình

> Tiến trình User

- Dùng TOAD, SQL Navigator có query kết nối CSDL

> Tiến trình Database

- Server Process khi người dùng truy cập

- Background Process tiến trình chạy cùng Oracle instance được bật

> Tiến trình Daemon / Application

- Tiến trình nền tảng bật OS có rồi: Listener mạng, Grid Infrastructure daemon



## Cấu trúc Process

### Background Process

> Tiến trình Database Writer ( DBWn)

- Ghi dữ liệu thay đổi trong database buffer cache dưới dạng dirty block khi:

    + Data buffer cache đầy 2/3
    + Checkpoint diễn ra (Giả sử diễn ra định kỳ 30p nó sẽ đồng bộ này, ko commit sẽ bỏ ra - đảm bảo ko mất thất thoát dữ liệu)
    + Khi tablespace OFFLINE hay READ ONLY hay BEGIN BACKUP
    + Khi bảng DROP hoặc TRUNCATE

> Tiến trình Log Writer (LGWR)

- Khi commit 1 transaction có 1 số sys/number, chưa ghi xuống datafile chỉ đánh dấu đã commit

- Khi có sự kiện ALTER SYSTEM SWITCH LOGFILE

- Khi redo log buffer đầy 1/3 or chứa 1MB trong buffer data

- Trước khi DBWn ghi dung lượng dữ liệu thay đổi data buffer cache xuống đĩa để trong trường hợp dữ liệu ghi xuống lỗi có redo log buffer để khôi phục

- Xảy ra liên tục 3s

> Tiến trình Checkpoint (CKPT)

- Cấu trúc dữ liệu định nghĩa sys/number

- System number là một số nguyên duy nhất được gán cho mỗi file redo log khi nó được tạo ra. System number sẽ tăng lên liên tục cho mỗi lần tạo file redo log mới.

- Ghi 2 thông tin

    + Số SCN (System change number) trong Control file (chứa thông tin cấu hình cơ sở dữ liệu như vị trí của các data file, log file, redo log sequence number,...)
    + Số SCN trong datafile (chứa dữ liệu của cơ sở dữ liệu)

- Khi có tín hiệu checkpoint ghi dữ liệu dirty block xuống đĩa, dùng background process là DBWn. Đồng bộ dữ liệu giữa control file và datafile

> Tiến trình System Monitor (SMON)

- Khi bật database sẽ kiểm tra tính toàn vẹn nếu database shutdown bị đột ngột, bật database trải qua quá trình crash recovery tìm tất cả các thành phần nào đã commit chưa được ghi xuống datafile thì ghi xuống datafile, đã thực hiện nhưng chưa commit thì rollback lại.

- Dọn dẹp những segment tạm không sử dụng

> Tiến trình Process Monitor (PMON)

- Thực hiện khôi phục các chương trình user bị lỗi bằng cách dọn dẹp vùng nhớ data buffer cache, giải phóng tài nguyên phục vụ user khác

- Giám sát các session idle timeout

> Tiến trình Recoverer (RECO)

- Dùng cấu hình CSDL phân tán

- Tự động kết nối CSDL liên quan đến transaction phân tán

- Tự động giải quyết vấn đề với transaction phân tán

- Giải phóng các row tương ứng với transaction

> Tiến trình Listener Registration (LREG)

- Đăng ký các listener và instance

> Tiến trình Archiver (ARCn)

- Khi redo log data quay vòng rồi, data nào ở trạng thái active thì ghi xuống Archiver,  ARCHIVELOG data cũ vứt ra ngoài sẽ an toàn. Có Archiver có thể restore

- Đảm bảo cho an toàn data bằng cách sao lưu liên tục ra file Archivelog



## Kiến trúc Storage CSDL

> Control files

- Bản đồ CSDL: chứa thông tin cấu trúc vật lý tên, thông tin đường dẫn datafile, redo log file, ....

> Data files

- Chứa data phân cấp theo table space về table, index, view

> Online redo log files

- Có trạng thái current active, inactive

> Archived redo log files

- Dữ liệu cũ của online redo log files dùng khôi phục

> Parameter file

- File chứa tham số , cấu hình ban đầu khởi tạo, startup

> Backup files

- Định kỳ backup khôi phục dữ liệu ở kỳ thời điểm nào khi lỗi hoặc mất file

> Password file

- Lưu password có permission role sysdba, sysbackup, sysoper, sysdg, syskm, sysasm

> Alert log and trace files

- Sinh ra trong qua trình vận hành, có tất cả các log, lỗi đều ở dạng alert log

## So sánh kiến trúc logic và vật lý của CSDL

> Logic: Database -> Tablespace -> Segment -> Extent -> Oracle data block

- Database: Được chia 1 cách hợp lý thành 2 hoặc nhiều tablespace

- Tablespace: Bao gồm các segments ứng dụng tối ưu thao tác quản trị

    + Tablespace SYSTEM và SYSAUX

            SYSTEM tablespace chứa các bản dữ liệu của data dictionary

            SYSAUX tablespace chứa các thành phần phụ trợ lại cho SYSTEM 

            Không sử dụng 2 tablespace cho ứng dụng

- Segments: Trong các tablespace, là nhóm các extent

    + Mức phân chia cao hơn nữa về logic của tablespace trong database. Có nhiều loại segment:

        * Data segment: 

                Mỗi một non-clustered table có một data segment. Các dữ liệu trong một table được lưu trữ trong các extents thuộc data segment đó. Với một partitioned table thì mỗi each partition lại tương ứng với một data segment. 

                Mỗi Cluster tương ứng với một data segment. Dữ liệu của tất cả các table trong cluster đó đều được lưu trữ trong data segment thuộc Cluster đó.

        * Index segment: 

                Mỗi một index đều có một index segment lưu trữ các dữ liệu của nó. Trong partitioned index thì mỗi partition cũng lại tương ứng với một index segment

        * Rollback segment:

                Một hoặc nhiều rollback segments của database được tạo lập bởi người quản trị database để lưu trữ các dữ liệu trung gian phục vụ cho việc khôi phục dữ liệu.

                Các thông tin trong Rollback segment được sử dụng để:

                   Tạo sự đồng nhất các thông tin đọc được từ      database

                   Sử dụng trong quá trình khôi phục dữ liệu
                   Phục hồi lại các giao dịch chưa commit đối với mỗi user

        * Temporary segment: 

                Temporary segments được tự động tạo bởi Oracle mỗi khi một câu lệnh SQL statement cần đến một vùng nhớ trung gian để thực hiện các công việc của mình như sắp xếp dữ liệu. Khi kết thúc câu lệnh đó, các extent thuộc temporary segment sẽ lại được hoàn trả cho hệ thống.

- Extents: Nhóm các data block liên tiếp nhau, lưu trữ tại cùng thiết bị, cùng loại thông tin

    + Mức phân chia cao hơn về logic

- Oracle data blocks: Được ánh xạ với block trên đĩa

    + Mức phân cấp logic thấp nhất, dữ liệu được lưu trong các data block. Tương ứng nhất định với một số lượng nhất định các bytes vật lý của database trong không gian đĩa cứng.

> Vật lý: Data file, Control file, Redo log file

## Oracle Container Database

> CSDL Multitenant container: Có 1 instance duy nhất, các file CSDL dạng vật lý

- Multitenant Architecture: Tiết kiệm bộ nhớ vì chung bộ nhớ, vẫn xảy ra sự tranh chấp nếu sử dụng nhiều

- Chia sẻ vùng nhớm metadate, redo log files, control file, undo tablespace

> CSDL Pluggable: Tập hợp các database schemas(lược đồ CSDL) dạng logic để user và ứng dụng là các CSDL độc lập riêng biệt

> Tất cả CSDL pluggable chia sẻ:

    - Tiến trình background

    - Bộ nhớ chia sẻ SGA

    - Oracle metadata

## Automatic Storage Management (ASM)

- Một tính năng quản lý không gian lưu trữ tự động trong Oracle Database. Nó cung cấp một giải pháp quản lý và tổ chức không gian lưu trữ dựa trên cấu trúc dữ liệu gọi là disk group

- ASM cho phép quản lý các ổ đĩa vật lý và phân chia chúng thành các disk group. Mỗi disk group là một nhóm các ổ đĩa vật lý được quản lý bởi ASM. ASM sử dụng các disk group để tổ chức và lưu trữ dữ liệu, bao gồm các tập tin dữ liệu, tập tin redo log và tập tin điều khiển.

- Các đặc điểm và lợi ích chính của ASM bao gồm:

    + Tự động phân chia và phân bố dữ liệu trên các ổ đĩa vật lý: ASM tự động phân chia và phân bố dữ liệu trên các disk group, giúp tối ưu hóa hiệu suất và sử dụng không gian lưu trữ.

    + Tự động quản lý và phục hồi dữ liệu: ASM cung cấp các tính năng tự động quản lý và phục hồi dữ liệu, bao gồm sao lưu và khôi phục dữ liệu tự động.

    + Tích hợp với Oracle Database: ASM tích hợp chặt chẽ với Oracle Database, cho phép quản lý và truy cập dữ liệu thông qua các giao diện quản lý cơ sở dữ liệu Oracle.

    + Tăng cường hiệu suất và khả năng mở rộng ASM CLuster file system: ASM cung cấp khả năng tăng cường hiệu suất và khả năng mở rộng bằng cách tự động phân chia và phân bố dữ liệu trên các ổ đĩa vật lý.





#created: 10/25/2023