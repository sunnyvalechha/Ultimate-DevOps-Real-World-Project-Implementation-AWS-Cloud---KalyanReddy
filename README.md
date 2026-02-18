Section: 10 - Advance Variables, Loops, Functions

c1-versions.tf

    terraform{
        required_version = ">= 1.10"
        required_providers {
            aws = {
                source  = "hashicorp/aws"
                version = ">= 6.0"
            }
        }
    }
    
    provider "aws" {
        region = var.aws_region
    }

c2-variables.tf

    variable "aws_region" {
      description = "region to deploy resources"
      type = string
      default = "ap-south-1"
    }
    
    variable "environment_name" {
      description = "environment name to deploy resources"
      type = string
      default = "dev"
    }
    
    variable "vpc_cidr" {
      description = "cidr block for vpc"
      type = string
      default = "10.0.0.0/16"
    }
    
    variable "tags" {
        description = "global tags to apply on all resources"
        type = map(string)
        default = {
            Terraform       = "true"
        }
    }
    
    variable "subnet_newbits" {
      description = "number of newbits to add to vpc cidr to generate subnets (eg 8 means /24 from /16)"
      type = number
        default = 8
    }

c3-datasources-locals.tf

* data source to get the availability zones in the region where we are creating our infrastructure. 
* We will use this data source to create subnets in different availability zones for high availability and fault tolerance.
  
    data "aws_availability_zones" "available" {
        state = "available"
    }
  
* We will use slice function to use first 3 availability zones for our subnets.
* This is because we want to create 3 public and 3 private subnets in different availability zones

* cidr calculates the subnet address within given IP network address prefix.
* We will use this function to create subnets in different availability zones.

* newbits is the number of additional bits with which to extend prefix. Ex: if given prefix ending /16 and newbits value is 4 the resulting subnet address is /20
* netnum - is the whole number that can be represented as a binary integer with no more than 'newbits
  
    locals {
      azs = slice(data.aws_availablity_zones.available.names, 0, 3)
      public_subnets = [for k, az in local.azs : cidrsubnet(var.vpc_cidr, var.subnet_newbits, k)]
      private_subnets = [for k, az in local.azs : cidrsubnet(var.vpc_cidr, var.subnet_newbits, k + 10)]
    }

  c4-vpc.tf

    # Res-1: VPC

  * Terraform merge function combines multiple maps or objects into a single map or object
  * 2 maps are here 'var.tags' & 'environment_name'
  * lifecycle is meta-argument that control how terraform creates your infra.
  
    resource "aws_vpc" "main" {
      cidr_block = var.vpc_cidr
      enable_dns_hostnames = true
      enable_dns_support = true
      tags = merge(var.tags, {Name = "${var.environment_name}-vpc"}) 
      lifecycle {
        prevent_destroy = false
      }
    }
  
  # Res-2: Internet Gateway
  
    resource "aws_internet_gateway" "igw" { 
      vpc_id = aws_vpc.main.id
      tags = merge(var.tags, { Name = "${var.environment_name}-igw"})
    }
  
  # Res-3: Public Subnet
  
    resource "aws_subnet" "public_subnet" {
      vpc_id = aws_vpc.main.id
      cidr_block = "10.0.0.0/16"
      availability_zone = "ap-south-1a"
      map_public_ip_on_launch = true
      tags = merge(var.tags, { Name = "${var.environment_name}-public_subnet"})
    }

* Note: Suppose we want to create multiple Public Subnets, do we need to create multiple resource block?
* It's not recommended, instead use 'for each' and 'count'
* count and for_each are meta-arguments used to create multiple instances of a resource or module from a single configuration block.
* 'count' creates instances based on a number, using numerical indices.
* 'for_each' creates instances based on a map or set, using unique keys
* Here, our requirement is subnet should create in each Az, hence using 'for_each'
  
    resource "aws_subnet" "public_subnet" {
      for_each = { for idx, az in local.azs : az => local.public_subnets[idx] }
      vpc_id = aws_vpc.main.id
      cidr_block = each.value
      availability_zone = each.key
      map_public_ip_on_launch = true
      tags = merge(var.tags, { Name = "${var.environment_name}-public-${each.key}" })
    }
  
  # Res-4: Private Subnet
  
    resource "aws_vpc" "main" {
      for_each = { for idx, az in local.azs : az => local.private_subnets[idx] }
      vpc_id = aws_vpc.main.id
      cidr_block = each.value
      availability_zone = each.key
      tags = merge(var.tags, { Name = "${var.environment_name}-private-${each.key}" })
    }
  
  # Res-5: Elastic IP for Nat Gateway
  
    resource "aws_eip" "nat" {
      tags = merge(var.tags, { Name = "${var.environment_name}-nat-eip" })
    }
  
  # Res-6: Nat Gateway
  
    resource "aws_nat_gateway" "nat" {
      allocation_id = aws_eip.nat.id
      subnet_id = values(aws_subnet.public)[0].id
      tags = merge(var.tags, { Name = "${var.environment_name}-nat" })
      depends_on = [ aws_internet_gateway.igw ]
    
    }
  
  # Res-7: Public Route Table
  
    resource "aws_route_table" "public_rt" {
      vpc_id = aws_vpc.main.id
      route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.igw
    
      }
      
    }
  
  # Res-8: Public Route Table Associate to Public Subnet
  
    resource "aws_route_table_association" "public_rt_association" {
      for_each = aws_subnet.public_subnet
      subnet_id = each.value.id
      route_table_id = aws_route_table.public_rt.id
    }
  
  # Res-9: Private Route Table
  
    resource "aws_route_table" "private_rt" {
      vpc_id = aws_vpc.main.id
      route {
        cidr_block = "0.0.0.0/0"
        nat_gateway_id = aws_nat_gateway.nat.id
    }
    tags = merge(var.tags, { Name = "${var.environment_name}-private-rt" })
    }

  # Res-10: Private Route Table Association to Private Subnet
  
    resource "aws_route_table_association" "private_rt_assiociation" {
      for_each = aws_subnet.private_subnet
      subnet_id = each.value.id
      route_table_id = aws_route_table.private_rt.id
    }

  c5-outputs.tf

      output "vpc_id" {
      value = aws_vpc.main.id
      description = "The ID of the created VPC"
    }
    
    output "public_subnet_ids" {
      value = [for subnet in aws_subnet.public_subnet : subnet.id]
      description = "List of IDs of the created public subnets"
    }
    
    output "private_subnet_ids" {
      value = [for subnet in aws_vpc.main : subnet.id]
      description = "List of private subnets IDs"
    }   
    
    output "public_subnet_map" {
      value = {for az, subnet in aws_subnet.public: az => subnet.id }
      description = "Map of availability zones to public subnet IDs"
    }


# Terraform Drift

* Terraform drift occurs when the actual, real-world infrastructure deviates from the desired state defined in your Terraform code.
* We should not make manual change, consider changes through terraform declarative approach.
* When we run "terraform plan" it will show output as new change has made but should be 'null' means nothing should be here.
* When we run "terraform apply" it will remove the changes had done manually.

# Terraform show

* The terraform show command provides human-readable output from a Terraform state file or a saved plan file.
* It is primarily used to inspect a plan before applying it

# Precedence of terraform variables

* Variables can be defined by using files like:

Lowest Priority to Highest Priority

1. variables.tf                    (default value)
2. TF_VAR_aws_region=ap-south-1    (Environment variables)
3. terraform.tfvars                (Auto-loaded if present in working directory)
4. abc.auto.tfvars                (auto-loaded & overides 'terraform.tfvars')
5. -var / -var-file                (terraform plan -var-file=abc.tfvars)

