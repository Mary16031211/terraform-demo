# terraform-demo 说明
## 体验目标
一键部署一个web应用
## 操作步骤
### Step1:登录体验环境
https://shell.aliyun.com

使用RAM用户密码登录账号：
- 小组1：demo-test1@1716965298866854.onaliyun.com
- 小组2：demo-test2@1716965298866854.onaliyun.com
- 小组3：demo-test3@1716965298866854.onaliyun.com
- 小组4：demo-test4@1716965298866854.onaliyun.com
- 小组5：demo-test5@1716965298866854.onaliyun.com

密码：本人口述

### Step2:创建资源栈空间
`mkdir testDemo`

`cd testDemo/`

`vim main.tf`

### Step3:资源编排
1、设置provider
`provider "alicloud" {}`

2、创建vpc网络和交换机
```
resource "alicloud_vpc" "vpc" {
   name       = "tf_test_foo"
   cidr_block = "172.16.0.0/12"
}
resource "alicloud_vswitch" "vsw" {
   vpc_id            = alicloud_vpc.vpc.id
   cidr_block        = "172.16.0.0/21"
   availability_zone = "cn-hangzhou-b"
}
```
说明：alicloud_vpc 的name需要自定义（建议在示例demo中增加组号后缀）

3、执行初始化和创建资源命令

`terraform init`

`terraform plan`

`terraform apply`

`terraform show`

4、创建安全组,并设置安全组作用于上一步的VPC上
```
resource "alicloud_security_group" "default" {
  name = "default"
  vpc_id = alicloud_vpc.vpc.id
}

#设置安全组
resource "alicloud_security_group_rule" "allow_all_tcp" {
  type              = "ingress"
  ip_protocol       = "tcp"
  nic_type          = "intranet"
  policy            = "accept"
  port_range        = "1/65535"
  priority          = 1
  security_group_id = alicloud_security_group.default.id
  cidr_ip           = "0.0.0.0/0"
}
```
说明：alicloud_security_group 的name需要自定义（建议在示例demo中增加组号后缀）

5、查看执行计划和执行创建资源命令

`terraform plan`

`terraform apply`

`terraform show`

6、创建负载均衡并配置监听器
```
resource "alicloud_slb" "slb" {
  name       = "test-slb-tf"
  vswitch_id = alicloud_vswitch.vsw.id
  address_type = "internet"
  specification = "slb.s1.small"
}

resource "alicloud_slb_listener" "http" {
  load_balancer_id = alicloud_slb.slb.id
  backend_port = 8080
  frontend_port = 80
  bandwidth = 10
  protocol = "http"
  sticky_session = "on"
  sticky_session_type = "insert"
  cookie = "testslblistenercookie"
  cookie_timeout = 86400
  health_check="on"
  health_check_type = "http"
  health_check_connect_port = 8080
}

#打印设置的公网IP
output "slb_public_ip"{
  value = alicloud_slb.slb.address
}
```
说明：alicloud_slb 的name需要自定义（建议在示例demo中增加组号后缀）
7、查看执行计划和执行创建资源命令

`terraform plan`

`terraform apply`

`terraform show`


8、创建弹性伸缩组，设置测试请求
```
resource "alicloud_ess_scaling_group" "scaling" {
  min_size = 2
  max_size = 10
  scaling_group_name = "tf-scaling"
  vswitch_ids = alicloud_vswitch.vsw.*.id
  loadbalancer_ids = alicloud_slb.slb.*.id
  removal_policies   = ["OldestInstance", "NewestInstance"]
  depends_on = ["alicloud_slb_listener.http"]
}
#设置弹性伸缩组，并初始化测试应用
resource "alicloud_ess_scaling_configuration" "config" {
  scaling_group_id = alicloud_ess_scaling_group.scaling.id
  image_id = "ubuntu_18_04_64_20G_alibase_20190624.vhd"
  instance_type = "ecs.sn1.medium"
  security_group_id = alicloud_security_group.default.id
  active= true
  enable= true
  user_data = "#!/bin/bash\necho \"Hello, World\" > index.html\nnohup busybox httpd -f -p 8080&"
  internet_max_bandwidth_in =10
  internet_max_bandwidth_out =10
  internet_charge_type = "PayByTraffic"
  force_delete= true
}
```
说明：alicloud_ess_scaling_group 的scaling_group_name需要自定义（建议在示例demo中增加组号后缀）

9、查看执行计划和执行创建资源命令

`terraform plan`

`terraform apply`

`terraform show`

10、设置伸缩规则
```
resource "alicloud_ess_scaling_rule" "rule" {
  scaling_group_id = alicloud_ess_scaling_group.scaling.id
  adjustment_type  = "TotalCapacity"
  adjustment_value = 2
  cooldown = 60
}
```
11、查看执行计划和执行创建资源命令

`terraform plan`

`terraform apply`

`terraform show`

### Step4:结果验证

终端输入：
`curl http://${slb_public_ip}`
或浏览器访问：http://${slb_public_ip}

### Step5:一键删除资源栈

`terraform destory`

`terraform show`
   
