# Kiberinfra_terraform


# main.tf
terraform {
  required_providers {
    openstack = {
      source = "terraform-provider-openstack/openstack"
    }
  }
}

provider "openstack" {}

# Создание SSH ключа
resource "openstack_compute_keypair_v2" "cloud_key" {
  name       = "cloud-key"
  public_key = var.ssh_public_key
}

# Создание сетей
resource "openstack_networking_network_v2" "mgmt_net" {
  name = "mgmt-net"
}

resource "openstack_networking_subnet_v2" "mgmt_subnet" {
  name       = "mgmt-subnet"
  network_id = openstack_networking_network_v2.mgmt_net.id
  cidr       = "10.0.1.0/24"
}

resource "openstack_networking_network_v2" "data_net" {
  name = "data-net"
}

resource "openstack_networking_subnet_v2" "data_subnet" {
  name       = "data-subnet"
  network_id = openstack_networking_network_v2.data_net.id
  cidr       = "10.0.2.0/24"
}

# Группы безопасности
resource "openstack_networking_secgroup_v2" "base_sg" {
  name        = "base-sg"
  description = "Base security group"
}

resource "openstack_networking_secgroup_rule_v2" "icmp" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "icmp"
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.base_sg.id
}

resource "openstack_networking_secgroup_rule_v2" "ssh" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 22
  port_range_max    = 22
  remote_ip_prefix  = "${var.cloud_adm_ip}/32"
  security_group_id = openstack_networking_secgroup_v2.base_sg.id
}

# Создание инстансов
data "openstack_images_image_v2" "alt_cloud" {
  name        = "alt-p10-cloud-x86_64"
  most_recent = true
}

resource "openstack_compute_instance_v2" "ha_nodes" {
  for_each = {
    "Cloud-HA01" = "1vCPU-512MB"
    "Cloud-HA02" = "1vCPU-512MB"
  }

  name            = each.key
  flavor_name     = each.value
  key_pair        = openstack_compute_keypair_v2.cloud_key.name
  security_groups = [openstack_networking_secgroup_v2.base_sg.name]

  block_device {
    uuid                  = data.openstack_images_image_v2.alt_cloud.id
    source_type           = "image"
    destination_type      = "volume"
    volume_size           = 10
    delete_on_termination = true
  }

  network {
    name = openstack_networking_network_v2.mgmt_net.name
  }
}

resource "openstack_compute_instance_v2" "db_web_nodes" {
  for_each = {
    "Cloud-DB01"  = "1vCPU-1GB"
    "Cloud-DB02"  = "1vCPU-1GB"
    "Cloud-WEB01" = "1vCPU-1GB"
    "Cloud-WEB02" = "1vCPU-1GB"
  }

  name            = each.key
  flavor_name     = each.value
  key_pair        = openstack_compute_keypair_v2.cloud_key.name
  security_groups = [openstack_networking_secgroup_v2.base_sg.name]

  block_device {
    uuid                  = data.openstack_images_image_v2.alt_cloud.id
    source_type           = "image"
    destination_type      = "volume"
    volume_size           = 10
    delete_on_termination = true
  }

  network {
    name = openstack_networking_network_v2.data_net.name
  }
}

# Балансировщик нагрузки
resource "openstack_lb_loadbalancer_v2" "web_lb" {
  name          = "Cloud-LB"
  vip_subnet_id = openstack_networking_subnet_v2.mgmt_subnet.id
}

resource "openstack_lb_listener_v2" "web_listener" {
  protocol        = "HTTP"
  protocol_port   = 80
  loadbalancer_id = openstack_lb_loadbalancer_v2.web_lb.id
}

resource "openstack_lb_pool_v2" "web_pool" {
  protocol    = "HTTP"
  lb_method   = "ROUND_ROBIN"
  listener_id = openstack_lb_listener_v2.web_listener.id
}

resource "openstack_lb_member_v2" "ha_members" {
  for_each = openstack_compute_instance_v2.ha_nodes

  pool_id       = openstack_lb_pool_v2.web_pool.id
  address       = each.value.access_ip_v4
  protocol_port = 80
}
