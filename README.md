# PROJECT: WEBSITE BUILD OF WORDPRESS ON AWS

Trong dự án này, tôi sẽ trình bày cách triển khai một trang web động trên AWS bằng kiến trúc tham chiếu dự án LampStack.
Các dịch vụ được sử dụng bao gồm: VPC, Nat Gateways, Security Group, EC2, RDS, Application LoadBalancer, Route 53, Autoscaling Group, Certificate Manager, EFS và một số thành phần khác, cụ thể sẽ được trình bày chi tiết ở phần bên dưới.

# Lamp stack project reference architecture
![1 _LAMP_Stack_Project_Reference_Architecture](https://user-images.githubusercontent.com/115881685/225284346-905bb80b-ad99-4b5c-9f28-832aa455dde3.jpg)

# Tạo VPC với Mạng con Công cộng và Riêng tư (Public and Private Subnets)
- Để bắt đầu cho dự án, tôi sẽ xây dựng một VPC tuỳ chỉnh, cụ thể hãy xem kiến trúc tham chiếu VPC bên dưới, kích hoạt theo 3 tầng (3 tiered).
- Trong kiến trúc tham chiếu vpc 3 tầng. Tầng đầu tiên chứa một Subnet công khai, bao gồm Nat Gateway, Bastion Host và Load Balancer. Ở tầng thứ 2, chúng ta có một Subnet riêng tư chưa các Web Server (EC2 Instances). Tầng thứ ba chưa một Subnet riêng tư, nơi lưu trữ cơ sở dữ liệu. Tất cả các subnet này được sao chép qua nhiều vùng khả dụng (Availability Zones) để cung cấp tính khả dụng cao và khả năng chịu lỗi. Cuối cùng, chúng ta sẽ tạo một Internet Gateway và bảng định tuyến, điều này sẽ cung cấp truy cập Internet cho các tài nguyên trong VPC.

## Kiến trúc tham chiếu VPC

![2 _VPC_Reference_Architecture](https://user-images.githubusercontent.com/115881685/225285355-25409fca-777c-4784-b8de-23e4dcb70191.jpg)

- Để tạo VPC, vào bảng điều khiển AWS, gõ "VPC" vào ô tìm kiếm và chọn VPC. Trong bảng điều khiển VPC, chọn VPC, sau đó nhấn "Tạo VPC" (Create VPC)

- Ở đây, tôi đặt tên cho VPC là "DEV VPC". Trong ô khối CIDR IPv4, nhập "10.0.0.0/16". Để tất cả các cài đặt khác theo mặc định, cuộn xuống và nhấn "tạo VPC" (Create VPC)

![image](https://user-images.githubusercontent.com/115881685/225288822-88cf185b-e21f-495c-b72d-84fd55696d3d.png)
![image](https://user-images.githubusercontent.com/115881685/225289117-2781efec-f4eb-42b6-b70b-faac8863321d.png)
![image](https://user-images.githubusercontent.com/115881685/225289229-2cb26b84-533e-49b6-acf3-fbe68687aad5.png)

![image](https://user-images.githubusercontent.com/115881685/225289405-5f49df39-ef2e-4fbc-b1bc-1f88f8385f59.png)

## Bước tiếp theo cần bật DNS Hostname trong VPC
- Để thực hiện, chọn "action" và sau đó chỉnh sửa "Cài đặt VPC" (Edit VPC Setting). Trong trang tiếp theo mở ra, tích vào ô "Bật DNS Hostname" (Enable DNS Hostname), sau đó cuộn xuống và nhấn nút lưu (Save).

![image](https://user-images.githubusercontent.com/115881685/225291345-c31135a1-fa3a-4846-9bac-3a3e72cf1ff7.png)
![image](https://user-images.githubusercontent.com/115881685/225291408-914debf2-26c2-4844-b03a-d971f6255a57.png)

## Tạo Internet Gateway
- Trong bảng điều khiển VPC, nhấp vào "Internet Gateway" và nhấn "Tạo Internet Gateway" (Create Internet Gateway). Đặt tên là "Dev Internet Gateway", sau đó cuộn xuống và nhấn "Tạo Internet Gateway" (Create Internet Gateway).

![image](https://user-images.githubusercontent.com/115881685/225293323-2a957e58-e968-4d79-ae90-c4aa55883d03.png)
![image](https://user-images.githubusercontent.com/115881685/225293451-87a74411-1ddd-4cc6-b793-d6b1a23f1f51.png)

- Bước tiếp theo là gắn Internet Gateway vào VPC đã được tạo. Để thực hiện, nhấp vào "Attach to a VPC" hoặc "action".
- Trong phần "Available VPCs", chọn DEV VPC đã được tạo, sau đó cuộn xuống và nhấn "Attach Internet Gateway".

![image](https://user-images.githubusercontent.com/115881685/225295224-23de14b1-fbe9-4e38-9a31-386cfd680595.png)
![image](https://user-images.githubusercontent.com/115881685/225294860-ccab48fd-65a2-45d0-b749-0eb458b2ec7f.png)
![image](https://user-images.githubusercontent.com/115881685/225295395-8e663aa2-607e-4d47-a3f3-c472ce689cc1.png)

## Tạo Mạng con (Create Subnets)
- Bước tiếp theo là tạo các Subnet trong vùng khả dụng thứ nhất và thứ hai theo kiến trúc tham chiếu VPC. Để thực hiện, chọn "Subnet" trong bảng điều khiển VPC và nhấn "tạo Subnet" (Create Subnets).
- Trong phần "VPC ID", chọn "Dev VPC" đã tạo ở phàn trước, sau đó cuộn xuống đến phần "Tên Subnet" (Subnet Name), theo kiến trúc tham chiếu, đặt tên là "Public Subnet AZ1". Trong phần "Vùng khả dụng" (Availability Zone), chọn "us-east-1a". Trong ô IPv4 CIDR blakc, nhập "10.0.0.0/24". Cuộn xuống và nhấn "Tạo Subnets" (Create Subnets).

![image](https://user-images.githubusercontent.com/115881685/225298356-e7b1d44f-c588-4aa2-bdb7-778ef1f33487.png)
![image](https://user-images.githubusercontent.com/115881685/225298547-6dd77db0-f912-458d-b68a-78a1f043fbaa.png)
![image](https://user-images.githubusercontent.com/115881685/225298768-26513a48-edc6-4206-89a9-b9fa9bf3bc96.png)

- Đối với subnet thứ hai, làm theo các bước tương tự như subnet đầu tiên. Trong phần “Tên subnet” (subnet name), đặt tên là “public subnet AZ2”. Đối với vùng khả dụng (availability zone), chọn “us-east-1b”. Trong ô IPv4 CIDR block, nhập “10.0.1.0/24”, sau đó cuộn xuống và nhấn “tạo subnet” (create subnets).
- Để xem hai subnet, hãy lọc theo VPC đã tạo.

![image](https://user-images.githubusercontent.com/115881685/225300423-b79e295f-951a-4718-94b2-99b14af0aec3.png)
![image](https://user-images.githubusercontent.com/115881685/225300692-99f0dfa5-06fb-413b-9638-dcb44e307e12.png)
![image](https://user-images.githubusercontent.com/115881685/225303279-3a1661a4-9914-4dd0-b20f-242012fffa96.png)

## Bật Auto-Assign IP cho Mạng con (Subnet)
- Để thực hiện việc này, chọn subnet đầu tiên và nhấp vào “action”, sau đó nhấn “chỉnh sửa cài đặt subnet” (edit subnet settings). Trong phần “auto-assign IP setting”, bật “tự động gán địa chỉ IPv4 công khai” (auto-assign public IPv4 address), sau đó cuộn xuống và nhấn “lưu” (save).
