Aws üzerinden VPC oluşturarak linux makinalara, LAMP yığınını kurmak,

Projeye uygun /CIDR vererek bir VPC oluşturarak başlayalım,
VPC oluştuktan sonra, VPC seçilerek Actions => Edit VPC setting => Dns hostnames'i aktifleştirelim.
Ağımızın internete çıkışını sağlamak için İnternet Gateway Oluşturalım ve oluşan İnternet Gateway'i VPC'ye attach edelim,
Hemen ardından Subnetleri oluşturalım, Projede belirtilen /CIDR'a göre 2 AZ'de 2 adet public subnet oluşturun, 2 adet private subnet oluşturun,
Public Subnetlerin otomatik ip alabilmeleri için Tek tek seçilerek Action => Edit Subnet Settings => Auto Assign ip bölümünde aktif edin.
Oluşan VPC'yi filtreleyerek Default olarak gelen  Route table'ı Public olarak belirleyin ve hemen ardından yeni bir Route Table oluşturun bunun ismine ise Private ismi verin. 
Oluşturulan Subnetleri Public ve private olarak  rotalara ilişkilendirin. 
Public'te oluşacak makinaların internete çıkması için İnternet Gateway'ı Public Routes seçin   Routes => Edit Routes => Add Routes => Destinations => 0.0.0.0/0 Target => İnternet Gateway olacak şekilde seçelim. 

Dipnot: Private makinaların internete çıkması için Nat instance yada Nat Gateway kullanın.

Security Group'ları oluşturalım;

İlk olarak Nat instance security group oluşturalım,
oluşan VPC'yi seçin ve İnbound rule'ları girin.
SSH ===============> Source: My IP 
HTTP =============> Anywhere-IPv4
HTTPS =============> Anywhere-IPv4
ICMP IPv4 =============> Anywhere-IPv4 (İnternete Bağlı olup olmadığımızı ping atarak öğrenmek için açılmıştır. İsteğe bağlı kaldırılabilir.)
Outbound default kalmalı.


Wordpress kurulacak olan makinanın security gropunu oluşturun;
oluşan VPC'yi seçin ve İnbound rule'ları girin.
SSH ===============> Source: Nat Instance 
HTTP =============> Anywhere-IPv4
HTTPS =============> Anywhere-IPv4
ICMP IPv4 =============> Anywhere-IPv4 (İnternete Bağlı olup olmadığımızı ping atarak öğrenmek için açılmıştır. İsteğe bağlı kaldırılabilir.)
Outbound default kalmalı.

Dipnot: Wordpress'in çalışması için bir adet database gereklidir, Bunun için RDS kurulmalı,

RDS security Group;
oluşan VPC'yi seçin ve İnbound rule'ları girin.
Mysql/aurora ============= > Source:  Wordpress Security group
Outbound default kalmalı.

İnternetten gelen http requestleri için ALB için securtity group oluşturalım;
oluşan VPC'yi seçin ve İnbound rule'ları girin.
HTTP ===============> Anywhere-IPv4
HTTPS ===============> Anywhere-IPv4
Outbound default kalmalı.

Security group oluşturma işlemleri tamamlanınca makinaları oluşturmaya başlayalım.

Nat istance oluşturmakla başlayalım;

CLI üzerinden oluşturmak için gerekli komut;

aws ec2 run-instances --image-id ami-005f9685cb30f234b --instance-type t2.micro --key-name <value> --security-group-ids <value> --subnet-id <value> --user-data file:/nat-instance.txtaws ec2 run-instances --image-id ami-005f9685cb30f234b --instance-type t2.micro --key-name <value> --security-group-ids <value> --subnet-id <value> --user-data file:/<value>


Nat instance oluşturmak için;

Standart ayarlarda bir Ec2 makina oluşturun Security Group'u seçin ve User Dataya aşağıdaki komutları ekleyin. Bu makinanız Nat instance olarak görev yapacaktır.Network settingsten Edit diyin ve oluşturulan VPC'yi ve Security Group'u seçin

User-data;

#! /bin/bash
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -s "Projede belirtilen İP aralığı" -j MASQUERADE

Nat instance oluştuktan sonra makinayı seçin Actions => Networking => Change source/destination => Paket kontrölünü Stop edin ve kaydedin.
Wordpress makinamız internete çıkmak ve bastion host/jump box yapmak için Nat instance gereklidir.
Nat İnstance oluştuktan sonra VPC altında Route Tables içinde Private Route table seçin ve Routes => Edit Routes => Destination 0.0.0.0/0 => Target İnstance seçin ve Nat instance'i seçin.
 
RDS Subnet oluşturalım;

Subnet Group sekmesi altından en az 2 AZ ve 2 Private subnet seçilecek şekilde Subnet group oluşturalım.

RDS kurulumuna geçebiliriz,

RDS : 

-Standart seçerek devam edelim,
-Database üstünde çalışacağı veritabanı: Mysql 
-Template olarak uygun olanı seçin.
-Credetentials Settings'te Username ve Password ayarlayın,
-Instance configuration da uygun olan makinayı seçin,
-Storage olarak isteğe bağlı gb seçimi yapın.
-Connectivity Olarak "Don’t connect to an EC2 compute resource" seçili kalsın ve oluşturulan VPC'yi seçin ve subnet group'u belirleyin. Public Acces -Kapalı olsun, Oluşturulan Security Groubu Seçin, 3306 Portunun açık olduğuna emin olun.
-Additional configuration altında ilk database ismini verin. Backup, Encryption, Maintenance isteğe bağlı seçilmelidir.
-RDS database oluşturalım ve wordpress olacak makina kurulumuna geçelim.

Wordpress Kurulacak Instance;

CLI üzerinden oluşturmak için gerekli komut;

aws ec2 run-instances --image-id ami-005f9685cb30f234b --instance-type t2.micro --key-name <value> --security-group-ids <value> --subnet-id <value> --user-data file:/<value>


Standart ayarlarda bir Ec2 makina oluşturun Wordprees için oluşturduğumuz Security Group'u seçin ve User Dataya aşağıdaki komutları ekleyin. Bu makinanız web sitemizin yayın yapacağı instance olarak görev yapacaktır.
Network settingsten Edit diyin ve oluşturulan VPC'yi ve Security Group'u seçin

User Data'ya eklenecek script;

#!/bin/bash
db_username=admin
db_user_password=123456789
db_name=wordpress
db_user_host=RDS endpoint
yum update -y
amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
yum install -y httpd
systemctl start httpd
systemctl enable httpd
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/
cd /var/www/html/
cp wp-config-sample.php wp-config.php
chown -R apache /var/www
chgrp -R apache /var/www
chmod 775 /var/www
find /var/www -type d -exec sudo chmod 2775 {} \;
find /var/www -type f -exec sudo chmod 0664 {} \;
sed -i "s/database_name_here/$db_name/g" wp-config.php
sed -i "s/username_here/$db_username/g" wp-config.php
sed -i "s/password_here/$db_user_password/g" wp-config.php
sed -i "s/localhost/$db_user_host/g" wp-config.php
systemctl restart httpd

wordpress makinamız internete çıkması için jump box olarak kullandığımız Nat instance'ı Route Table'a ekleyelim.
Routes => Edit Routes => Destination 0.0.0.0/0 => Target İnstance seçin ve Nat instance'i seçin.

Wordpress makinamız oluştuktan sonra Target Group oluşturmaya geçelim,

Target group;

Target Type'ını İnstance olarak seçelim,
Target ismi verelim.
Protocol ve port ayarlayalım isteğe bağlı default kalabilir.
Protocol version: HTTP1 olarak seçilebilir.
Health checks ise isteğe bağlı olarak ayarlanabilir.
Bu target group'a wordpress instance'ı ekleyelim


Target Grouplar bittikten sonra Load balancer oluşturalım.

Load Balancer:

Load Balancer tipini seçerek başlayalım:
ALB (Application Load Balancer) olarak devam edeceğiz,
Load balancer ismi verelim,
Default olarak devam edelim
VPC seçelim 2 AZ ve 2 Public subnet seçelim,
ALB için oluşturulan Security Group'u seçin,
Listeners and routing altında  oluşturulan Target Gropu'u seçelim.
Oluşturalım.

ALB DNS Adresini alın ve web browser'a yapıştırın.

TEBRİKLER SİTENİZ KULLANIMA HAZIR.








