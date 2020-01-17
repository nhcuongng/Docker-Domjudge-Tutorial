# Các công cụ cần chuẩn bị

- Kiến thứ cơ bản về docker cũng như domjudge
- Tải [Docker](https://docs.docker.com/docker-for-windows/install/)
- Tải Image từ docker về bằng câu lệnh sau ```docker pull domjudge/domserver:7.1.2```. Mình đã test phiên bản mới nhất nhưng bị lỗi, vì thế đã kiểm tra lần lượt và tìm được phiên bản này, tính đến bây giờ (18/01/2020) là ko lỗi. sau khi tải xong, kiểm tra image đã pull về như sau ```docker images```

# Các bước tiến hành

## MariaDB container

Trước khi chạy domjudge và judgehost, ta cần chuẩn bị một database như là MariaDB/Mysql

Trong trường hợp này mình lựa chọn MariaDB

Cách nhanh nhất để chuẩn bị đó là dùng MariaDb docker container, chạy câu lệnh dưới đây để khởi tạo container
> docker run -it --restart=always --name dj-mariadb -e MYSQL_ROOT_PASSWORD=rootpw -e MYSQL_USER=domjudge -e MYSQL_PASSWORD=djpw -e MYSQL_DATABASE=domjudge -p 13306:3306 mariadb --max-connections=1000

Câu lệnh này sẽ start MariaDB container, cài root password là ```rootpw```, MySQL user tên là ```domjudge``` cùng với password là ```djpw``` and và tạo database rỗng là ```domjudge```

Đồng thời ta có thể quản trị database qua cổng 13306 bằng MYSQL GUI như là [My SQL Workbench](https://dev.mysql.com/downloads/workbench/)

If you want to save the MySQL data after removing the container, please read the MariaDB Docker Hub page for more information.

## DOMserver container

### Bước 1: khởi tạo và chạy

Chạy câu lệnh sau để chạy ```DOMserver container``` cũng như pull image (về nếu ko có)
>docker run --link dj-mariadb:mariadb -it -e MYSQL_HOST=mariadb -e MYSQL_USER=domjudge -e MYSQL_DATABASE=domjudge -e MYSQL_PASSWORD=djpw -e MYSQL_ROOT_PASSWORD=rootpw -p 12345:80 --name domserver domjudge/domserver:7.1.2

Mình sẽ giải thích ý nghĩa một số câu lệnh như sau:

- ```--link``` là để liên kết với MariaDb đã tạo ở trên
- ```-e``` là các biến môi trường,chứa các thông tin để có thể liên kết với database Mariadb đã tạo ở trên
- ```-p``` đây là khai báo cổng theo format ```Local machine:container``` có nghĩa là ở máy thật sẽ có thể truy cập vào container qua cổng 12345, còn ở container sẽ mở cổng 80. để máy thật và máy ảo liên kết đc với nhau
- ```-name``` là tên của container

sau khi chạy xong, trên màn hình sẽ hiện rõ password của tài khoản admin là gì, nếu ko nhớ bạn có thể xem thông qua câu lệnh này
> docker exec -it domserver cat /opt/domjudge/domserver/etc/initial_admin_password.secret

Sau đó bạn có thể truy cập địa chỉ [http://localhost:12345/](http://localhost:12345/) và đăng nhập tài khoản admin

### Bước 2: Thay đổi tài khoản judgehost

Đây là tài khoản để sau này khi ta tạo ```Judgehost container``` thì có thể xác thực (để có thể truy cập API do Domjudge cung cấp)

Các bước thay đổi như sau
- Vào [http://localhost:12345/jury](http://localhost:12345/jury) để vào trang web ```DOMjudge Jury interface```
- Chọn ```Users -> judgehost -> điền mật khẩu``` sau đó save lại
- Hãy ghi chú lại mật khẩu bạn vừa điền

### Bước 3: Kiểm tra IP

Để truy cập vào giao diện bash (các bạn có thể hiểu là giao diện Command line) của ```DOMserver container```

Các bạn ghõ câu lệnh sau:
> docker exec -it domserver bash

Sau đó kiểm tra ip của DOMserver bằng câu lệnh sau
> ip addr show

Và lưu lại, sẽ cần thiết cho các bước sau này

## Judgehost container

### Bước 1: khởi tạo và chạy Judgehost container

Để chạy và khởi tạo một single judgehost, ta chạy câu lệnh sau
> docker run -it --privileged -v /sys/fs/cgroup:/sys/fs/cgroup:ro --name judgehost-0 --link domserver:domserver --hostname judgedaemon-0 -e DAEMON_ID=0 -e JUDGEDAEMON_PASSWORD=12345678 -e DOMSERVER_BASEURL=http://172.17.0.3:80/ domjudge/judgehost:7.1.2

Mình sẽ giải thích một số điều quan trọng như sau
  - ```DAEMON_ID``` là id của từng judgehost, mỗi judgehost cần Id khác nhau nhé, trong trường hợp này mình set là 0
  - ```--hostname``` sẽ là tên của judgehost sau khi judgehost được gắn vào domserver
  - ```JUDGEDAEMON_PASSWORD``` là password mà bạn đã nhập ở mục ```Bước 2: Thay đổi tài khoản judgehost```
  - ```DOMSERVER_BASEURL``` sẽ được diền như sau ```ip của DOMserver:PortDOMserver``` trong trường hợp này thì của mình là ```http://172.17.0.3:80```

### Bước 2: Ping to DOMserver

> docker exec -it judgehost-0 bash
> nmap -p 80 ${địa chỉ ip của domserver}

Bạn đã thành công

### Bước 3: Xem file restapi.secret

Bước này sẽ sửa file ```restapi.secret```, đây là file đóng vai trò then chốt trong việc xác thực của judgehost với DOMserver
Sau khi chạy xong câu lệnh ở phần 1, bạn có thể xem thông tin của restapi.secret qua câu lệnh sau
> docker exec -it judgehost-0 cat /opt/domjudge/domserver/etc/restapi.secret

# Tài liệu

- [Docker Hub Domjudge](https://hub.docker.com/r/domjudge/domserver)
- [Domjudge docs](https://www.domjudge.org/docs/manual/index.html)
