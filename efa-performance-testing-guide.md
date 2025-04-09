# AWS Elastic Fabric Adapter (EFA) Performance Testing Guide

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites and Environment Setup](#prerequisites-and-environment-setup)
3. [Security Group Configuration for EFA](#security-group-configuration-for-efa)
4. [EFA Installation Process](#efa-installation-process)
5. [Perftest Installation and Compilation](#perftest-installation-and-compilation)
6. [Detailed EFA Performance Testing](#detailed-efa-performance-testing)
7. [Comparing EFA vs TCP/IP Performance](#comparing-efa-vs-tcpip-performance)
8. [Test Result Analysis and Interpretation](#test-result-analysis-and-interpretation)
9. [Troubleshooting and Best Practices](#troubleshooting-and-best-practices)

## Introduction

AWS Elastic Fabric Adapter (EFA) is a network interface for Amazon EC2 instances that enables customers to run applications requiring high levels of inter-node communications at scale on AWS. EFA provides lower and more consistent latency and higher throughput than the TCP transport traditionally used in cloud-based HPC systems.

Key benefits of EFA include:

- **Lower latency**: EFA reduces network latency compared to traditional TCP/IP networking
- **Higher bandwidth**: Enables higher throughput for data-intensive applications
- **OS-bypass capability**: Allows the hardware to communicate directly with the application, bypassing the operating system kernel
- **Scalability**: Designed for applications that scale to thousands of CPU cores
- **MPI support**: Optimized for Message Passing Interface (MPI) applications

This guide demonstrates how to set up EFA on Amazon Linux 2023 instances, install the perftest benchmarking tools, and measure the performance between two m6i.metal instances.

## Prerequisites and Environment Setup

### Instance Configuration

This guide uses the following environment:

- **Instance Type**: m6i.metal
- **Operating System**: Amazon Linux 2023
- **Placement Group**: Cluster placement group for optimal network performance
- **Instances**:
  - Server: efa-server-1.zzhe.xyz (Public IP: 3.231.6.128, Private IP: 172.31.94.37)
  - Client: efa-client-1.zzhe.xyz (Public IP: 3.232.156.232, Private IP: 172.31.93.171)

### Required Permissions

To complete this guide, you need:

- EC2 instances with EFA-enabled network interfaces
- SSH access to both instances
- Sudo privileges on both instances

### Verify EFA Hardware

Before proceeding, verify that your instances have EFA-enabled network interfaces:

```bash
# On both instances
lspci | grep -i efa
```

You should see output similar to this, indicating the presence of EFA devices:

```
2f:00.0 Ethernet controller: Amazon.com, Inc. Elastic Fabric Adapter (EFA)
```

If you don't see this output, ensure your instances have EFA-enabled network interfaces attached.

Note: The `fi_info` command will be available after installing the EFA software in the next section.

## Security Group Configuration for EFA

EFA requires specific security group settings to function properly. Unlike standard network interfaces that can use security groups to restrict traffic by port and protocol, EFA traffic bypasses these controls for its OS-bypass functionality.

### Required Security Group Rules

Create a security group with the following rules:

1. **All inbound traffic from members of the same security group**:
   - This allows EFA-enabled instances to communicate with each other

2. **SSH access (TCP port 22) from your IP address**:
   - This allows you to connect to your instances

### Creating the Security Group via AWS CLI

```bash
# Create the security group
aws ec2 create-security-group \
    --group-name EFA-enabled-sg \
    --description "Security group for EFA-enabled instances" \
    --vpc-id vpc-xxxxxxxx

# Allow all traffic between members of the security group
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxxxxxxxxxxxxx \
    --source-group sg-xxxxxxxxxxxxxxxxx \
    --protocol all

# Allow SSH access from your IP address
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxxxxxxxxxxxxx \
    --protocol tcp \
    --port 22 \
    --cidr YOUR_IP_ADDRESS/32
```

### Verifying Security Group Configuration

To verify your security group is correctly configured:

```bash
aws ec2 describe-security-groups \
    --group-ids sg-xxxxxxxxxxxxxxxxx
```

Ensure that the security group allows:
- All traffic between members of the same security group
- SSH access from your IP address

### Best Practices for EFA Security

1. **Limit EFA access**: Only attach EFA-enabled security groups to instances that require EFA functionality
2. **Use VPC isolation**: Place EFA-enabled instances in dedicated VPCs or subnets
3. **Regular audits**: Periodically review security group rules to ensure they meet your security requirements
4. **IAM restrictions**: Use IAM policies to control who can modify EFA-related security groups

## EFA Installation Process

The EFA installation process involves installing the EFA driver, Libfabric, and Open MPI on both instances.

### Installing EFA Software on Amazon Linux 2023

Run the following commands on both instances:

```bash
# Update the system
sudo dnf update -y

# Install development tools
sudo dnf install -y gcc gcc-c++ make cmake autoconf automake libtool

# Download the EFA installer
curl -O https://efa-installer.amazonaws.com/aws-efa-installer-latest.tar.gz

# Extract the installer
tar -xf aws-efa-installer-latest.tar.gz

# Navigate to the extracted directory
cd aws-efa-installer

# Run the installer
sudo ./efa_installer.sh -y

# Add the EFA library path to the system's library path
echo 'export LD_LIBRARY_PATH=/opt/amazon/efa/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

### Verifying EFA Installation

After installation, verify that EFA is correctly installed:

```bash
# Find your EFA device name
ls /sys/class/infiniband/

# Check EFA driver version (replace rdmap47s0 with your device name)
cat /sys/class/infiniband/rdmap47s0/device/driver/module/version

# Example output:
# 2.13.0g

# Check if EFA interface is available
/opt/amazon/efa/bin/fi_info -p efa

# Example output:
# provider: efa
#     fabric: efa
#     domain: rdmap47s0-rdm
#     version: 122.0
#     type: FI_EP_RDM
#     protocol: FI_PROTO_EFA
# provider: efa
#     fabric: efa
#     domain: rdmap47s0-dgrm
#     version: 122.0
#     type: FI_EP_DGRAM
#     protocol: FI_PROTO_EFA

# Verify that Libfabric is installed
/opt/amazon/efa/bin/fi_info --version

# Example output:
# /opt/amazon/efa/bin/fi_info: 1.22.0amzn5.0
# libfabric: 1.22.0amzn5.0
# libfabric api: 1.22
```

You should see output similar to the examples above, indicating that the EFA driver and Libfabric are installed correctly. Note that the EFA device name (shown as `rdmap47s0` in the example) may vary between instances. Use the name you find in your environment for subsequent commands.

## Perftest Installation and Compilation

Perftest is a collection of benchmarking tools for RDMA (Remote Direct Memory Access) that can be used to measure the performance of EFA.

### Installing Dependencies

First, install the required dependencies on both instances:

```bash
# Install dependencies
sudo dnf install -y git libudev-devel libnl3-devel pciutils-devel
```

### Downloading and Compiling Perftest

Clone and compile the perftest repository on both instances:

```bash
# Clone the perftest repository
git clone https://github.com/linux-rdma/perftest.git

# Navigate to the perftest directory
cd perftest

# Configure and compile
./autogen.sh
./configure
make
sudo make install
```

### Verifying Perftest Installation

Verify that perftest is correctly installed:

```bash
# Check if perftest binaries are available
which ib_send_bw ib_send_lat ib_write_bw ib_write_lat

# Example output:
# /usr/bin/ib_send_bw
# /usr/bin/ib_send_lat
# /usr/bin/ib_write_bw
# /usr/bin/ib_write_lat

# Check the version
ib_send_bw -V

# Example output (your version may differ):
# perftest version: 4.5-0.20
```

You should see output similar to the examples above, showing the paths to the perftest binaries and the version information.

## Detailed EFA Performance Testing

This section covers various performance tests to evaluate EFA performance between the two instances.

### Bandwidth Measurements

Bandwidth tests measure the maximum throughput between the instances.

#### RDMA Send Bandwidth Test

On the server instance (efa-server-1.zzhe.xyz):

```bash
ib_send_bw -d rdmap47s0 -x 0 -F -R
```

On the client instance (efa-client-1.zzhe.xyz):

```bash
ib_send_bw -d rdmap47s0 -x 0 -F -R 172.31.94.37
```

#### RDMA Write Bandwidth Test

On the server instance:

```bash
ib_write_bw -d rdmap47s0 -x 0 -F -R
```

On the client instance:

```bash
ib_write_bw -d rdmap47s0 -x 0 -F -R 172.31.94.37
```

Parameters explained:
- `-d rdmap47s0`: Use the EFA device (replace with your device name)
- `-x 0`: Use RDMA CM for connection establishment
- `-F`: Use events for completions
- `-R`: Print bandwidth in GB/s (instead of MB/s)

Expected results for m6i.metal instances:
- Send bandwidth: ~12-20 GB/s
- Write bandwidth: ~12-20 GB/s

### Latency Measurements

Latency tests measure the time it takes for a message to travel from one instance to another.

#### RDMA Send Latency Test

On the server instance:

```bash
ib_send_lat -d rdmap47s0 -x 0 -F
```

On the client instance:

```bash
ib_send_lat -d rdmap47s0 -x 0 -F 172.31.94.37
```

#### RDMA Write Latency Test

On the server instance:

```bash
ib_write_lat -d rdmap47s0 -x 0 -F
```

On the client instance:

```bash
ib_write_lat -d rdmap47s0 -x 0 -F 172.31.94.37
```

Expected results for m6i.metal instances:
- Send latency: ~15-25 μs
- Write latency: ~15-25 μs

### Message Rate Testing

Message rate tests measure how many messages can be sent per second.

On the server instance:

```bash
ib_send_lat -d rdmap47s0 -x 0 -F -n 1000000
```

On the client instance:

```bash
ib_send_lat -d rdmap47s0 -x 0 -F -n 1000000 172.31.94.37
```

Parameters explained:
- `-n 1000000`: Send 1 million messages

Calculate message rate by dividing the number of iterations by the total time:
- Message rate = iterations / (total time in seconds)

Expected results for m6i.metal instances:
- Message rate: ~100,000-200,000 messages per second for small messages

### RDMA Operations Performance

Compare the performance of different RDMA verbs (READ, WRITE, SEND).

#### RDMA Read Bandwidth Test

On the server instance:

```bash
ib_read_bw -d rdmap47s0 -x 0 -F -R
```

On the client instance:

```bash
ib_read_bw -d rdmap47s0 -x 0 -F -R 172.31.94.37
```

Compare the results with the WRITE and SEND tests to understand the performance differences between RDMA operations.

### Scalability Testing

Test how performance scales with multiple parallel streams.

#### Multiple Streams Bandwidth Test

On the server instance:

```bash
ib_send_bw -d rdmap47s0 -x 0 -F -R -q 4
```

On the client instance:

```bash
ib_send_bw -d rdmap47s0 -x 0 -F -R -q 4 172.31.94.37
```

Parameters explained:
- `-q 4`: Use 4 queue pairs (parallel streams)

Try different values for the `-q` parameter (1, 2, 4, 8, 16) to see how bandwidth scales with the number of parallel streams.

### CPU Utilization

Monitor CPU utilization during tests to evaluate efficiency.

On both instances, run the following command in a separate terminal before starting the tests:

```bash
mpstat 1
```

This will show CPU utilization statistics every second. Compare CPU utilization between EFA and TCP/IP tests to understand the efficiency benefits of EFA.

### Bidirectional Performance

Test simultaneous data transfer in both directions.

#### Bidirectional Bandwidth Test

On the server instance:

```bash
ib_send_bw -d rdmap47s0 -x 0 -F -R -b
```

On the client instance:

```bash
ib_send_bw -d rdmap47s0 -x 0 -F -R -b 172.31.94.37
```

Parameters explained:
- `-b`: Enable bidirectional testing

Compare the results with unidirectional tests to understand the impact of bidirectional communication.

## Comparing EFA vs TCP/IP Performance

This section compares the performance of EFA with traditional TCP/IP networking.

### Setting Up TCP/IP Performance Testing

Install iperf3 for TCP/IP performance testing:

```bash
# On both instances
sudo dnf install -y iperf3
```

### Bandwidth Comparison

#### EFA Bandwidth Test

Run the EFA bandwidth test as described earlier:

On the server instance:

```bash
ib_send_bw -d rdmap47s0 -x 0 -F -R
```

On the client instance:

```bash
ib_send_bw -d rdmap47s0 -x 0 -F -R 172.31.94.37
```

#### TCP/IP Bandwidth Test

On the server instance:

```bash
iperf3 -s
```

On the client instance:

```bash
iperf3 -c 172.31.94.37 -P 4 -t 30
```

Parameters explained:
- `-P 4`: Use 4 parallel streams
- `-t 30`: Run the test for 30 seconds

#### TCP/IP Mode in Perftest

You can also use perftest in TCP mode for a more direct comparison:

On the server instance:

```bash
ib_send_bw -d rdmap47s0 -x 0 -F -R -T 90
```

On the client instance:

```bash
ib_send_bw -d rdmap47s0 -x 0 -F -R -T 90 172.31.94.37
```

Parameters explained:
- `-T 90`: Use TCP mode with a timeout of 90 seconds

### Latency Comparison

#### EFA Latency Test

Run the EFA latency test as described earlier:

On the server instance:

```bash
ib_send_lat -d rdmap47s0 -x 0 -F
```

On the client instance:

```bash
ib_send_lat -d rdmap47s0 -x 0 -F 172.31.94.37
```

#### TCP/IP Latency Test

On the server instance:

```bash
ib_send_lat -d rdmap47s0 -x 0 -F -T 90
```

On the client instance:

```bash
ib_send_lat -d rdmap47s0 -x 0 -F -T 90 172.31.94.37
```

You can also use the ping command for a basic latency test:

```bash
ping -c 100 172.31.94.37
```

### CPU Utilization Comparison

Monitor CPU utilization during both EFA and TCP/IP tests:

```bash
mpstat 1
```

Compare the CPU utilization between EFA and TCP/IP tests. EFA typically shows lower CPU utilization due to its OS-bypass capability.

### Performance Under Different Workloads

Test both EFA and TCP/IP performance with different message sizes:

```bash
# EFA test with different message sizes
ib_send_bw -d rdmap47s0 -x 0 -F -R -s 1024    # 1KB messages
ib_send_bw -d rdmap47s0 -x 0 -F -R -s 65536   # 64KB messages
ib_send_bw -d rdmap47s0 -x 0 -F -R -s 1048576 # 1MB messages

# TCP/IP test with different message sizes
ib_send_bw -d rdmap47s0 -x 0 -F -R -T 90 -s 1024    # 1KB messages
ib_send_bw -d rdmap47s0 -x 0 -F -R -T 90 -s 65536   # 64KB messages
ib_send_bw -d rdmap47s0 -x 0 -F -R -T 90 -s 1048576 # 1MB messages
```

### Expected Performance Differences

Based on typical results for m6i.metal instances:

| Metric | EFA | TCP/IP | Improvement |
|--------|-----|--------|-------------|
| Bandwidth | 12-20 GB/s | 5-10 GB/s | 2-4x |
| Latency | 15-25 μs | 50-100 μs | 3-5x |
| Message Rate | 100K-200K msg/s | 20K-50K msg/s | 4-5x |
| CPU Utilization | Low | High | 30-50% reduction |

### Use Cases Where EFA Shows the Most Significant Advantages

1. **HPC applications**: Applications that require low-latency, high-bandwidth communication between nodes
2. **Machine learning distributed training**: Frameworks like TensorFlow and PyTorch that benefit from fast gradient synchronization
3. **Tightly coupled simulations**: CFD, weather modeling, and other simulations that require frequent communication between nodes
4. **High-frequency trading**: Applications that require minimal latency for market data processing
5. **Large-scale data analytics**: Applications that process large datasets across multiple nodes

## Test Result Analysis and Interpretation

This section explains how to interpret the results of the performance tests.

### Understanding Perftest Output

The output of perftest tools includes several important metrics:

#### Bandwidth Test Output

```
#bytes #iterations    BW peak[GB/s]    BW average[GB/s]   MsgRate[Mpps]
65536  1000             12.41            12.38             0.198039
```

- **#bytes**: Message size in bytes
- **#iterations**: Number of messages sent
- **BW peak[GB/s]**: Maximum bandwidth achieved
- **BW average[GB/s]**: Average bandwidth over the test duration
- **MsgRate[Mpps]**: Message rate in millions of packets per second

#### Latency Test Output

```
#bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]    t_stdev[usec]   99% percentile[usec]   99.9% percentile[usec]
8      1000             15.32         26.47         16.45             16.83          1.47            21.37                  26.47
```

- **#bytes**: Message size in bytes
- **#iterations**: Number of messages sent
- **t_min[usec]**: Minimum latency in microseconds
- **t_max[usec]**: Maximum latency in microseconds
- **t_typical[usec]**: Typical latency (mode) in microseconds
- **t_avg[usec]**: Average latency in microseconds
- **t_stdev[usec]**: Standard deviation of latency
- **99% percentile[usec]**: 99th percentile latency
- **99.9% percentile[usec]**: 99.9th percentile latency

### What Good/Expected Values Look Like for m6i.metal Instances

For m6i.metal instances with EFA, you can expect:

- **Bandwidth**: 12-20 GB/s for large messages (64KB+)
- **Latency**: 15-25 μs for small messages (8 bytes)
- **Message Rate**: 100,000-200,000 messages per second for small messages
- **CPU Utilization**: Significantly lower than TCP/IP for the same workload

### Comparison with Theoretical Maximums

The theoretical maximum bandwidth for EFA on m6i.metal instances is around 25 GB/s. If your results are significantly lower, consider:

1. **Network contention**: Other instances in the same placement group might be using network resources
2. **CPU bottlenecks**: The CPU might not be able to generate or consume data fast enough
3. **Memory bandwidth limitations**: Memory access might be limiting the overall performance
4. **Software configuration**: Suboptimal configuration of EFA or the application

### Visualizing Results

To visualize the results, you can use tools like gnuplot or create spreadsheets. Here's a simple example of how to collect data for visualization:

```bash
# Create a CSV file with bandwidth results for different message sizes
echo "Message Size (bytes),EFA Bandwidth (GB/s),TCP Bandwidth (GB/s)" > bandwidth_results.csv

# Run tests with different message sizes and append results to the CSV file
for size in 1024 4096 16384 65536 262144 1048576; do
  # EFA test
  efa_bw=$(ib_send_bw -d rdmap47s0 -x 0 -F -R -s $size | grep $size | awk '{print $4}')
  
  # TCP test
  tcp_bw=$(ib_send_bw -d rdmap47s0 -x 0 -F -R -T 90 -s $size | grep $size | awk '{print $4}')
  
  echo "$size,$efa_bw,$tcp_bw" >> bandwidth_results.csv
done
```

You can then use this CSV file to create charts showing the performance comparison.

## Troubleshooting and Best Practices

### Common Issues and Solutions

#### EFA Device Not Found

If `/opt/amazon/efa/bin/fi_info -p efa` doesn't show any EFA providers:

1. Verify that the instance has an EFA-enabled network interface:
   ```bash
   aws ec2 describe-network-interfaces --network-interface-ids eni-xxxxxxxxxxxxxxxxx
   ```

2. Check if the EFA driver is loaded:
   ```bash
   lsmod | grep efa
   ```

3. Reinstall the EFA driver:
   ```bash
   sudo ./efa_installer.sh -u
   sudo ./efa_installer.sh -y
   ```

#### Poor Performance

If you're experiencing poor performance:

1. Verify that both instances are in the same placement group:
   ```bash
   aws ec2 describe-instances --instance-ids i-xxxxxxxxxxxxxxxxx i-yyyyyyyyyyyyyyyyy
   ```

2. Check for network contention using CloudWatch metrics

3. Ensure that the security group allows all traffic between the instances

4. Verify that the EFA MTU is set correctly:
   ```bash
   ip link show
   ```

5. Check system resources during tests:
   ```bash
   top
   ```

#### Connection Issues

If you're having trouble establishing connections:

1. Verify that both instances can ping each other:
   ```bash
   ping 172.31.94.37
   ping 172.31.93.171
   ```

2. Check if the required ports are open:
   ```bash
   sudo netstat -tulpn | grep LISTEN
   ```

3. Verify that the security group allows all traffic between the instances

### Best Practices for EFA Performance

1. **Use cluster placement groups**: Always place EFA-enabled instances in a cluster placement group for optimal performance

2. **Choose the right instance type**: Different instance types have different EFA performance characteristics

3. **Optimize message sizes**: Use message sizes that are optimal for your workload (typically 64KB-1MB for bandwidth-intensive workloads)

4. **Use multiple streams**: For bandwidth-intensive workloads, use multiple parallel streams to maximize throughput

5. **Monitor CPU and memory usage**: Ensure that CPU and memory are not bottlenecks

6. **Use the latest EFA driver**: Regularly update the EFA driver to benefit from performance improvements and bug fixes

7. **Consider application-specific optimizations**: Different applications may require different optimizations for optimal EFA performance

### Maintenance and Updates

To update the EFA software:

```bash
# Download the latest EFA installer
curl -O https://efa-installer.amazonaws.com/aws-efa-installer-latest.tar.gz

# Extract the installer
tar -xf aws-efa-installer-latest.tar.gz

# Navigate to the extracted directory
cd aws-efa-installer

# Run the installer
sudo ./efa_installer.sh -y
```

Regularly check for updates to the EFA software to benefit from performance improvements and bug fixes.

## Conclusion

This guide has demonstrated how to set up EFA on Amazon Linux 2023 instances, install the perftest benchmarking tools, and measure the performance between two m6i.metal instances. By following these steps, you can evaluate the performance benefits of EFA for your specific workloads and compare them with traditional TCP/IP networking.

EFA provides significant performance advantages for applications that require high-bandwidth, low-latency communication between instances. By understanding these performance characteristics, you can make informed decisions about when to use EFA for your workloads.

For more information about EFA, refer to the [AWS EFA documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html).
