```bash
-----------SonarqubeUpdate_Final------------


kubectl create namespace sonarqube

sudo mkdir -p /mnt/sonarqube
sudo chown -R 1000:1000 /mnt/sonarqube
sudo chmod -R 775 /mnt/sonarqube

```
```bash
----Setup database-----

sudo apt update && sudo apt upgrade -y
sudo apt install postgresql postgresql-contrib -y
sudo -i -u postgres
psql
Cấu hình cho phép kết nối từ xa (Nếu cần)
Sửa file postgresql.conf
sudo nano /etc/postgresql/16/main/postgresql.conf
Tìm dòng #listen_addresses = 'localhost' và đổi thành: listen_addresses = '*'

Sửa file pg_hba.conf
sudo nano /etc/postgresql/16/main/pg_hba.conf
Thêm dòng này vào cuối file để cho phép mọi IP (hoặc giới hạn IP cụ thể của bạn):
host    all             all             0.0.0.0/0            md5

Mở Firewall và Khởi động lại
sudo ufw allow 5432/tcp
sudo systemctl restart postgresql

systemctl status postgresql

sudo -i -u postgres psql

Tạo Database và User

Tạo Database mới
CREATE DATABASE my_project_db;

Tạo User mới với mật khẩu mạnh
CREATE USER project_admin WITH ENCRYPTED PASSWORD 'mat_khau_cua_ban';

Cấp toàn quyền cho User trên Database đó
GRANT ALL PRIVILEGES ON DATABASE my_project_db TO project_admin;

----Flush database------
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;
GRANT ALL ON SCHEMA public TO postgres;
GRANT ALL ON SCHEMA public TO public;


Chọn Database của SonarQube
sudo -u postgres psql
\c sonarqube
3. Cấp quyền cho User
Giả sử User bạn dùng cho SonarQube là sonar_user, hãy chạy lệnh sau:
-- Cấp quyền sử dụng schema public
GRANT USAGE ON SCHEMA public TO sonar_user;

-- Cấp quyền tạo bảng trong schema public (Quan trọng nhất)
GRANT CREATE ON SCHEMA public TO sonar_user;

-- Đảm bảo user sở hữu database này (tùy chọn nhưng nên làm)
ALTER DATABASE sonarqube OWNER TO sonar_user;

------Config postgres store data với sonarqube--------
sudo -u postgres psql
-- 1. Tạo user
CREATE USER sonar WITH ENCRYPTED PASSWORD 'mypassword';

-- 2. Tạo database
CREATE DATABASE sonarqube OWNER sonar;

-- 3. Cấp quyền (Rất quan trọng cho bản Postgres 15+)
GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonar;

-- Kết nối vào db vừa tạo để cấp quyền schema
\c sonarqube
GRANT ALL ON SCHEMA public TO sonar;

```
```bash
sudo nano /etc/sysctl.conf
Thêm các dòng sau vào cuối file:
vm.max_map_count=262144
fs.file-max=65536

sudo sysctl -p
```
```bash
Run file volume.yaml, service.yaml, configupdate.yaml
   cd SonarqubeUpdate_Final
   kubectl apply -f volume.yaml
   kubectl apply -f service.yaml
   kubectl apply -f configupdate.yaml
  
```
```bash
Theo dõi tiến trình khởi động
kubectl get pods -n sonarqube -w
kubectl logs sonarqube-general-0 -n sonarqube -f

Xem Log để xác nhận "Operational"
kubectl logs sonarqube-general-0 -n sonarqube -f

kubectl logs sonarqube-general-0 -n sonarqube | grep "is operational"
kubectl logs -f statefulset/sonarqube-general -n sonarqube
kubectl describe pod sonarqube-general-0 -n sonarqube
kubectl get svc -n sonarqube
kubectl get pvc -n sonarqube
sudo kubectl get all -n sonarqube
sudo kubectl get pvc -n sonarqube

Khi trạng thái chuyển sang Running, bạn có thể kiểm tra xem nó có nằm đúng Node không:
kubectl get pod -n sonarqube -o wide

kubectl delete pod sonarqube-general-0 -n sonarqube
kubectl delete svc sonarqube -n sonarqube


kubectl delete pvc data-sonarqube -n sonarqube --ignore-not-found
kubectl delete pv sonar-pv --ignore-not-found
sudo kubectl delete -f volume.yaml
sudo rm -rf /mnt/sonarqube/*
sudo kubectl delete deployment sonarqube-general -n sonarqube

Kiểm tra init-sysctl (Cài đặt cấu hình hệ thống)
kubectl logs sonarqube-general-0 -n sonarqube -c init-sysctl
kubectl logs sonarqube-general-0 -n sonarqube -c init-fs
```
