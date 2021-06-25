# Project-04: Phonebook App (Python Flask) deployed on AWS Application Load Balancer with Auto Scaling and Relational Database Service (RDS) using AWS CloudFormation

## Description

The Phonebook Application aims to create a phonebook application via Python and deployed as a web application with Flask on AWS Application Load Balancer with Auto Scaling Group of Elastic Compute Cloud (EC2) Instances and Relational Database Service (RDS) using AWS CloudFormation Service.

## Problem Statement

- Your company has recently started a project that aims to serve as phonebook web application. You and your colleagues have started to work on the project. Your teammates have developed the UI part the project as shown in the template folder and they need your help to develop the coding part and deploying the app in development environment.


- As a first step, you need to write program that creates a phonebook, adds requested contacts to the phonebook, finds and removes the contacts from the phonebook.

- Application should allow users to search, add, update and delete the phonebook records and the phonebook records should be kept in separate MySQL database in AWS RDS service. Following is the format of data to be kept in db.

- As a second step, after you finish the coding, you are requested to deploy your web application using Python's Flask framework.


- You need to transform your program into web application using the `index.html`, `add-update.html` and `delete.html` within the `templates` folder. Note the followings for your web application.

- User should face first with `index.html` when web app started and th user should be able to;
- The Web Application should be accessible via web browser from anywhere.
- Lastly, after transforming your code into web application, you are requested to push your program to the project repository on the Github and deploy your solution in the development environment on AWS Cloud using AWS Cloudformation Service to showcase your project. In the development environment, you can configure your Cloudformation template using the followings,

  - The application stack should be created with new AWS resources.

  - Template should create Application Load Balancer with Auto Scaling Group of Amazon Linux 2 EC2 Instances within default VPC.

  - Application Load Balancer should be placed within a security group which allows HTTP (80) connections from anywhere.

  - EC2 instances should be placed within a different security group which allows HTTP (80) connections only from the security group of Application Load Balancer.

  - The Auto Scaling Group should use a Launch Template in order to launch instances needed and should be configured to;

    - use all Availability Zones.

    - set desired capacity of instances to `2`

    - set minimum size of instances to `1`

    - set maximum size of instances to `3`

    - set health check grace period to `90 seconds`

    - set health check type to `ELB`

  - The Launch Template should be configured to;

    - prepare Python Flask environment on EC2 instance,

    - download the Phonebook Application code from Github repository,

    - deploy the application on Flask Server.

  - EC2 Instances type can be configured as `t2.micro`.

  - Instance launched by Cloudformation should be tagged `Web Server of StackName`

  - For RDS Database Instance;
  
    - Instance type can be configured as `db.t2.micro`

    - Database engine can be `MySQL` with version of `8.0.19`.

  - Phonebook Application Website URL should be given as output by Cloudformation Service, after the stack created.

## Steps to Solution
  
- Step 1: Download or clone project definition from `sezginerdem` repo on Github

- Step 2: Create project folder for local public repo on your pc

- Step 3: Write the Phonebook Application in Python

- Step 4: Transform your application into web application using Python Flask framework

- Step 5: Prepare a cloudformation template to deploy your app on Application Load Balancer together with RDS

CloudFormation Template imin ne ise yaradigini gosteren description bolume aciklama ekledim.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: |
  CloudFormation Template for Phonebook Application. This template creates Application Load Balancer 
  with Auto Scaling Group of Amazon Linux 2 (ami-0947d2ba12ee1ff75) EC2 Instances which host Python Flask Web Application.
  EC2 instances are placed within WebServerSecurityGroup which allows http (80) connections only from ALBSecurityGroup,
  and allows tcp(3306) connections only within itself. RDS DB instance is placed within WebServerSecurityGroup so that
  Database Server can communicate with Web Servers.
  Application Load Balancer is placed within ALBSecurityGroup which allows http (80) connections from anywhere.
  WebServerASG Auto Scaling Group is using the WebServerLT Launch Template in order to spin up instances needed.
  WebServerLT Launch Template is configured to prepare Python Flask environment on EC2,
  and to deploy Phonebook Application on Flask Server after downloading the app code from Github repository.
  ```

Sonrasinda resources bolumunde oncelik olarak security group ekledim. Ilk once launch template ile olusacak olan ec2larin sg larini yazdim. Default VPC kullandigim icin VPC tanimlamadim. Ingress rule tanimladigim icin engress rule tanimlamam gerek yok. Ingress rule olarak HTTP ve SSH portlarini actim. HTTP portunda CidrIp yerine SourceSecurityGroupId tanimladim. Burada !GetAtt metodu ile olusacak SG un id sini aldim. Bunun nedeni client in istek yapacagi ilk resource alb, ec2 lar da elb den gelen istekleri gerceklstirecegi icin elb sg unu buraya tanimlamam gerekiyor. 

```yaml
Resources:
  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable HTTP for Flask Web Server and Ssh for entering to EC2
      SecurityGroupIngress:
         - IpProtocol: tcp
           FromPort: 22
           ToPort: 22
           CidrIp: 0.0.0.0/0
         - IpProtocol: tcp
           FromPort: 80
           ToPort: 80
           SourceSecurityGroupId: !GetAtt ALBSecurityGroup.GroupId
```

ALBSecurityGroup adi ile alb sg olusturdum. sadece 80 portundan dinleyecegi icin sadece bu portu actim. Default VPC kullandim. 

```yaml
ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable HTTP for Application Load Balancer
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
```

WebServerLT adi ile Launch Template imi olusturdum. Launch Template imde instance tipimi t2.micro olarak belirledim. Keyname olarak bende olan key imin ismini yazdim. SG olarak daha once olusturmus oldugum WebServerSecurityGroup un GroupId sini !GetAtt metodu ile aldim. Stack e verilecek ismin tag olarak atanmasi icin !Sub fonksiyonu u ile cektim. 

```yaml
WebServerLT:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData: 
        ImageId: ami-0947d2ba12ee1ff75
        InstanceType: t2.micro
        KeyName: ec2keyname
        SecurityGroupIds: 
          - !GetAtt WebServerSecurityGroup.GroupId
        TagSpecifications: 
          - ResourceType: instance
            Tags:
              - Key: Name
                Value: !Sub Web Server of ${AWS::StackName} Stack
```
Launch Template im user datasini olusturdum. !GetAtt metodu ile db in endpointi cektim ve MyDBURI adli degiskene atadim. Bu degiskeni !cho "${MyDBURI}" > /home/ec2-user/dbserver.endpoint" komutu ile ec2 icerisinde bir dosyaya kayit ettim. Token olusturdum ve bu token sayesinde curl komutu ile githubdaki dosyalarimi cektim. En son olarak da python3 ile uygulamami calistirdim.
```yaml
        UserData:
          Fn::Base64:
            !Sub 
              - |
                #! /bin/bash
                yum update -y
                yum install python3 -y
                pip3 install flask
                pip3 install flask_mysql
                echo "${MyDBURI}" > /home/ec2-user/dbserver.endpoint
                TOKEN="94eddca57871d6cbcb24babb5e0ff31f30226804"
                FOLDER="https://$TOKEN@https://raw.githubusercontent.com/sezginerdem/p04-Phonebook-App-deployed-on-AWS-ALB-with-AS-and-RDS-using-CF/master/"
                curl -s --create-dirs -o "/home/ec2-user/templates/index.html" -L "$FOLDER"templates/index.html
                curl -s --create-dirs -o "/home/ec2-user/templates/add-update.html" -L "$FOLDER"templates/add-update.html
                curl -s --create-dirs -o "/home/ec2-user/templates/delete.html" -L "$FOLDER"templates/delete.html
                curl -s --create-dirs -o "/home/ec2-user/app.py" -L "$FOLDER"phonebook-app.py
                python3 /home/ec2-user/app.py
              - MyDBURI: !GetAtt MyDatabaseServer.Endpoint.Address
```
WebServerTG adi ile target group tanimlamasi yaptim. Burada bana isterlere bakarak tanimlamalari yaptim. VpcId yi de !GetAtt fonksiyonu ile WebServerSecurityGroup.VpcId den aldim. 

```yaml
 WebServerTG:
    Type: "AWS::ElasticLoadBalancingV2::TargetGroup"
    Properties:
      Port: 80
      Protocol: HTTP
      TargetType: instance
      UnhealthyThresholdCount: 3
      HealthyThresholdCount: 2
      VpcId: !GetAtt WebServerSecurityGroup.VpcId
```

ApplicationLoadBalancer adi ile Load Balancer olusturdum. SecurityGroup olustururken !GetAtt fonksiyonu ile ALBSecurityGroup.GroupId den SG u cektim. Default subnetlerimi buraya aldim. Application Load balancer oldugu icin Type olarak application yazdim. 


```yaml
ApplicationLoadBalancer:
    Type: "AWS::ElasticLoadBalancingV2::LoadBalancer"
    Properties:
      IpAddressType: ipv4
      Scheme: internet-facing
      SecurityGroups:
        - !GetAtt ALBSecurityGroup.GroupId
      Subnets:
        - subnet-077c9758
        - subnet-3246e63c
        - subnet-3ccd235a
        - subnet-8d8dbfb3
        - subnet-c41ba589
        - subnet-ed49bccc
      Type: application
```

ALBListener adi ile listenerimi yazdim. DefaultActions da TG u dinlemesi gerektigini yazdim. TG u belirlemenin birkac farkli yontemi var ben bunu tercih ettim. "TargetGroupArn: !Ref WebServerTG" ile bunu sagladim. Type ini da forward olarak belirliyorum. Portunu 80 protocolunu de HTTP olarak belirledim. 


```yaml
ALBListener:
    Type: "AWS::ElasticLoadBalancingV2::Listener"
    Properties:
      DefaultActions: #required
        - TargetGroupArn: !Ref WebServerTG
          Type: forward
      LoadBalancerArn: !Ref ApplicationLoadBalancer #required
      Port: 80 #required
      Protocol: HTTP #required
```
WebServerASG adi ile AutoScaling Group olusturdum. !GetAZs "" ile ASG un tum AZ lerde olusabilecegini tanimladim. InstanceId yerien LaunchTemplate kullandim. "LaunchTemplateId: !Ref WebServerLT" satirindaki !Ref fonksiyonu ile LT nin ismini yazarak buraya cektim. "Version: !GetAtt WebServerLT.LatestVersionNumber" satiri ile LT in VersionNumber ini aldim. Bu da bir TG dinliyor. Ayni LB daki adres gostermem gibi TargetGroupARNs de !Ref fonksiyonu ile WebServerTG u adres gosteriyorum. 


```yaml
 WebServerASG:
    Type: "AWS::AutoScaling::AutoScalingGroup"
    Properties:
      AvailabilityZones:
        !GetAZs ""
      DesiredCapacity: 2
      HealthCheckGracePeriod: 300
      HealthCheckType: ELB
      LaunchTemplate:
        LaunchTemplateId: !Ref WebServerLT
        Version: !GetAtt WebServerLT.LatestVersionNumber
      MaxSize: 3 #required
      MinSize: 1 #required
      TargetGroupARNs:
        - !Ref WebServerTG
```

DB server olsuturmak icin 2 tane resource olusturmam gerekiyor. Bunlardan biri DB SG bir digeri de DB.

DB icin SG tanimlamasi yaptim adina da MyDBSecurityGroup verdim. CIDRIP blog olarak 0.0.0.0/0 tanimladim ancak normalde olusturulacak VPC nin CIDRIP blogu buraya yazilmalidir veya ozellikle koymak istediginiz bir subnet araligini varsa buraya yazilabilir. EC2SecurityGroupId olarak buraya ulasacak WebServerSecurityGroup.GroupId lerini !GetAtt fonksiyonu ile getiriyorum. 


```yaml
 MyDBSecurityGroup:
    Type: AWS::RDS::DBSecurityGroup
    Properties:
      GroupDescription: Front-end access
      DBSecurityGroupIngress:
        - CIDRIP: 0.0.0.0/0
        - EC2SecurityGroupId: !GetAtt WebServerSecurityGroup.GroupId
```

DB ismini MyDatabaseServer olarak belirledim. Storage olarak 20GB belirledim. "AllowMajorVersionUpgrade: false" ile mevcut mysql versionunu kullanmak istiyorum ve major upgarde etmek istemedigimi belirtiyorum. "AutoMinorVersionUpgrade: true" ile minor upgrade istedigimi belirtiyorum.


```yaml
  MyDatabaseServer:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Delete
    Properties:
      AllocatedStorage: 20
      AllowMajorVersionUpgrade: false 
      AutoMinorVersionUpgrade: true
      BackupRetentionPeriod: 0 #herhangi bir backup alma
      DBInstanceIdentifier: phonebook-app-db2
      DBName: phonebook
      DBSecurityGroups: #db sg u MyDBSecurityGroup adini yazarak cektim
        - !Ref MyDBSecurityGroup
      Engine: MySQL #engine belirledim
      DBInstanceClass: db.t2.micro #Required
      EngineVersion: 8.0.19 #versionumu belirledim
      MasterUsername: Sezgin 
      MasterUserPassword: Sezgin_1
      MultiAZ: false #failover senaryoda baska bir AZ de db ayaga kalkmasini istemedigim icin false yazdim
      Port: 3306
      PubliclyAccessible: true #VPC disindan ec2 larimin ulasmasina izin veriyorum
```
DNS adresine kolay erismek icin otput kismini olusturdum bunun icin asagidaki kodlamayi yaptim

```yaml
Outputs:
  WebsiteURL:
    Value: !Sub 
      - http://${ALBAddress} #atadigim adresi http nin onune koydum ve boylelikle bir adres olusmus oldu
      - ALBAddress: !GetAtt ApplicationLoadBalancer.DNSName #ALB DNS name inin atama islemini yaptim
    Description: Phonebook Application Load Balancer URL
```

- Step 6: Push your application into your own public repo on Github

- Step 7: Deploy your application on AWS Cloud using Cloudformation template to showcase your app

## Outcome

![Phonebook App Search Page](./search-snapshot.png)

## Farkli Cozum yollari
Baska bir ec2 ayaga kaldirisiniz bu ec2 da init-phonebook-db.py dosyasini calsirstirir. Ardindan da bu ec2 nun kisa bir sure icinde kendini termnitane etmesini saglayabilirsiniz. Onun sadece gorevi db olusturmak olur. Bunu yaptiktan sonra da phonebook-app.py icerisindeki initialize bolumunu kaldirirsiniz onceki ec2 sadece db i initialize eder ardindan da autoscalingler icerisinde kurulacak launch templatelerde initialize bolumu olmaz boyle bir initialize bolumu olmadigi icin de her yeni uretilen makina db i silmeyecektir. 
Hazirlayacagim template de InstanceInitiatedStutdownBehaviour alanini terminate olarak ayarlayip bu template in icine initialize.py yi atip ne kadar sonra terminate edilecegini de ayarlayabilirsiniz. Userdata icerisine yerlestirilecek bir kod ile ne zaman sonra terminate edilecegini ayarlayabilirsiniz. 

en zorlandigim kisim db endpoitinin ec2 icerisinde bir yere atilarak db ile ec2 arasinda baglanti kurulmasiydi. 