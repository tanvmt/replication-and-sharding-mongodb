# Xây dựng Cụm MongoDB Sharding & Replication với Docker Compose

Dự án này hướng dẫn cách thiết lập một cụm (cluster) MongoDB phân tán có đầy đủ hai tính năng Replication và Sharding bằng Docker Compose. Toàn bộ hệ thống được giả lập trên một máy tính duy nhất, giúp cho việc học tập và thử nghiệm trở nên dễ dàng mà không cần đến nhiều máy chủ vật lý.

## Kiến trúc Hệ thống
Hệ thống được thiết kế bao gồm 10 container Docker, hoạt động như 10 máy chủ độc lập:

- 1 Mongos Query Router: Cổng giao tiếp chính của cluster, lắng nghe trên cổng 27017.
- 1 Config Server Replica Set (config-rs): Gồm 3 node (mongo-cfg1, mongo-cfg2, mongo-cfg3), chịu trách nhiệm lưu trữ metadata của cluster. Các node này chạy trên cổng nội bộ 27019.
- 2 Shards: Nơi lưu trữ dữ liệu chính. Mỗi shard là một Replica Set gồm 3 node để đảm bảo tính sẵn sàng cao.
  - Shard 1 (shard1-rs): Gồm 3 node (mongo-shard1a, 1b, 1c), chạy trên cổng nội bộ 27018.
  - Shard 2 (shard2-rs): Gồm 3 node (mongo-shard2a, 2b, 2c), chạy trên cổng nội bộ 27020.
## Yêu cầu Cài đặt
- [Docker Desktop](https://docs.docker.com/get-docker/) (đã bao gồm docker-compose).
- Một trình biên dịch dòng lệnh như Command Prompt, PowerShell hoặc Git Bash.
## Hướng dẫn Cài đặt và Cấu hình
Quá trình này yêu cầu thực hiện các lệnh theo đúng thứ tự.

### Bước 1: Khởi chạy toàn bộ Cluster
Tạo một thư mục cho dự án và đặt file `docker-compose.yml` này vào trong đó.
Mở terminal tại thư mục dự án và chạy lệnh sau để khởi động tất cả 10 container:

```
docker-compose up -d
```
Chờ một lát để Docker tải image và khởi động các container. Bạn có thể dùng lệnh `docker ps` để kiểm tra xem tất cả đã ở trạng thái "Up" hay chưa.

### Bước 2: Khởi tạo các Replica Set

#### 2.1. Khởi tạo Replica Set cho Config Servers
Mở một terminal mới và chạy lệnh sau để truy cập vào shell của `mongo-cfg1`:

```
docker exec -it mongo-cfg1 mongosh --port 27019
```

Trong shell của `mongosh` vừa mở, dán và chạy lệnh sau để khởi tạo replica set `config-rs`:

```
rs.initiate({
    _id: "config-rs", 
    configsvr: true, 
    members: [
        {_id: 0, host: "mongo-cfg1:27019"}, 
        {_id: 1, host: "mongo-cfg2:27019"}, 
        {_id: 2, host: "mongo-cfg3:27019"}
    ]
})
```
Sau khi thực hiện xong, gõ `exit` và nhấn Enter để thoát.

#### 2.2 Khởi tạo Replica Set cho Shard 1
Trong terminal, chạy lệnh để truy cập vào `mongo-shard1a`:

```
docker exec -it mongo-shard1a mongosh --port 27018
```

Trong shell `mongosh`, chạy lệnh sau:

```
rs.initiate({
    _id: "shard1-rs", 
    members: [
        {_id: 0, host: "mongo-shard1a:27018"}, 
        {_id: 1, host: "mongo-shard1b:27018"}, 
        {_id: 2, host: "mongo-shard1c:27018"}
    ]
})
```

Gõ `exit` để thoát.

#### 2.3 Khởi tạo Replica Set cho Shard 2
Trong terminal, chạy lệnh để truy cập vào `mongo-shard2a`:

```
docker exec -it mongo-shard2a mongosh --port 27020
```
Trong shell `mongosh`, chạy lệnh sau:

```
rs.initiate({
    _id: "shard2-rs", 
    members: [
        {_id: 0, host: "mongo-shard2a:27020"}, 
        {_id: 1, host: "mongo-shard2b:27020"}, 
        {_id: 2, host: "mongo-shard2c:27020"}
    ]
})
```
Gõ `exit` để thoát.
### Bước 3: Thêm các Shard vào Cluster
Lưu ý: Hãy chờ khoảng 15-20 giây sau khi hoàn thành Bước 2 để các replica set có thời gian bầu cử node Primary.

Bây giờ, cluster đã có các replica set nhưng router `mongos` chưa biết về chúng. Hãy kết nối vào `mongos` từ máy thật của bạn:

```
mongosh --port 27017
```
Trong shell của `mongos`, chạy lần lượt 2 lệnh sau để thêm 2 shard vào cluster:

```
sh.addShard("shard1-rs/mongo-shard1a:27018,mongo-shard1b:27018,mongo-shard1c:27018")
sh.addShard("shard2-rs/mongo-shard2a:27020,mongo-shard2b:27020,mongo-shard2c:27020")
```

### Bước 4: Kiểm tra Cluster
Vẫn trong shell của `mongos`, chạy lệnh:

```
sh.status()
```
Nếu output hiển thị thông tin về `shard1-rs` và `shard2-rs` trong mục shards, xin chúc mừng, cluster của bạn đã được thiết lập thành công!

## Dọn dẹp Môi trường
Khi bạn đã hoàn thành và muốn xóa toàn bộ cluster, hãy quay lại terminal tại thư mục dự án và chạy lệnh:

```
docker-compose down -v
```
Lệnh này sẽ dừng, xóa tất cả các container và cả dữ liệu đã được lưu trong volumes.
