#!/bin/bash

# Check if the script is run as root
if [ "$(id -u)" -ne 0 ]; then
    echo "Please run the script as root using sudo."
    exit 1
fi

# Default value
default_network="192.168.69.0/24"

# Create an associative array to store the results
declare -A machines_map
declare -A open_ports_map

# Define well-known ports to scan
declare -a ports=(22 80 443 139 445)

# Function to display the welcome message and instructions.
welcome_message() {
    echo -e "\033[1;35m" # Set text color to purple and make it bold
    echo "███    ██ ███████ ████████ ██     ██  ██████  ██████  ██   ██               "
    echo "████   ██ ██         ██    ██     ██ ██    ██ ██   ██ ██  ██                "
    echo "██ ██  ██ █████      ██    ██  █  ██ ██    ██ ██████  █████                 "
    echo "██  ██ ██ ██         ██    ██ ███ ██ ██    ██ ██   ██ ██  ██                "
    echo "██   ████ ███████    ██     ███ ███   ██████  ██   ██ ██   ██               "
    echo "                                                                            "
    echo "                ███████  ██████  █████  ███    ██ ███    ██ ███████ ██████  "
    echo "                ██      ██      ██   ██ ████   ██ ████   ██ ██      ██   ██ "
    echo "                ███████ ██      ███████ ██ ██  ██ ██ ██  ██ █████   ██████  "
    echo "                     ██ ██      ██   ██ ██  ██ ██ ██  ██ ██ ██      ██   ██ "
    echo "                ███████  ██████ ██   ██ ██   ████ ██   ████ ███████ ██   ██ "
    echo -e "\033[0m" # Reset text effects
    echo -e "\033[1;34mCOMP1671-Penetration Testing and Ethical Vulnerability Scanning\033[0m"
    echo -e "\033[1;32mActive Enumeration Week2\033[0m\n"
    echo -e "\033[1mInstructions:\033[0m"
    echo -e "1. Enumerate all machines on the network using ICMP protocol - ping (considering TTL and Packet Size)"
    echo -e "2. Enumerate all machines on the network using ARP protocol - arping"
    echo -e "3. Enumerate and map machines on the network based on ICMP and ARP - ping arping"
    echo -e "4. Enumerate at least 5 well-known ports on each machine and map them - timeout (default ports: 22,80,139,443,445)"
    echo -e "5. Banner Grabbing - netcat (Run after option 4)"
    echo -e "6. Reverse DNS - host"
    echo -e "7. Analyze bandwidth of Option 4"
    echo -e "8. Quit\n"
    echo -e "\033[1mPlease enter the number of your choice:\033[0m"
}

# ICMP scan function
icmp_scan() {
    local network="$1"
    local ttl="$2"
    local packet_size="$3"
    local tmp_icmp_file=$(mktemp)

    # Split the network into base network and prefix length
    IFS='/' read -r full_network prefix <<<"$network"
    base_network="${full_network%.*}"

    # Calculate the number of addresses based on the prefix length
    num_host_bits=$((32 - prefix))
    num_addresses=$((2 ** num_host_bits - 2))

    # Perform ping for each address in the network
    for i in $(seq 1 $num_addresses); do
        ip="$base_network.$i"
        (ping -c 1 -W 0.2 -t $ttl -s $packet_size "$ip" &>/dev/null && echo "$ip" >>$tmp_icmp_file) &

        # Limit the number of concurrent ping processes
        limit_concurrent_processes

    done
    wait

    echo $tmp_icmp_file

}

# ARP scan function
arp_scan() {
    local network="$1"
    local tmp_arp_file=$(mktemp)

    # Split the network into base network and prefix length
    IFS='/' read -r full_network prefix <<<"$network"
    base_network="${full_network%.*}"

    num_host_bits=$((32 - prefix))
    num_addresses=$((2 ** num_host_bits - 2))

    # Perform arping for each address in the network
    for i in $(seq 1 $num_addresses); do
        ip="$base_network.$i"
        (sudo arping -I eth0 -c 1 "$ip" 2>/dev/null | grep "bytes from" >>$tmp_arp_file) &

        # Limit the number of concurrent ping processes
        limit_concurrent_processes
    done
    wait

    echo $tmp_arp_file
}

# Function to limit the number of concurrent processes
limit_concurrent_processes() {
    if [ $(jobs | wc -l) -ge 300 ]; then
        wait
    fi
}

# Function to enumerate all machines on the network using ICMP protocol
enumerate_icmp() {
    read -p "Enter the network in CIDR notation (default is $default_network): " network
    
    # Set the default value if input is empty
    network=${network:-$default_network} 
    
    # Split the network into base network and prefix length
    IFS='/' read -r base_network prefix <<<"$network"

    # Calculate the number of addresses based on the prefix length
    num_host_bits=$((32 - prefix))
    num_addresses=$((2 ** num_host_bits - 2))

    read -p "Enter the TTL (Time to Live) for ICMP packets (default is 64): " ttl
    ttl=${ttl:-64}

    read -p "Enter the packet size in bytes (default is 64): " packet_size
    packet_size=${packet_size:-64}

    echo "Enumerating all machines on the network using ICMP protocol..."

    tmp_icmp_file=$(icmp_scan "$network" "$ttl" "$packet_size")

    # Display table header
    echo "+-----------------+--------+"
    echo "| IP Address      | Status |"
    echo "+-----------------+--------+"

    while read -r ip; do
        printf "| %-15s | Active |\n" "$ip"
    done <"$tmp_icmp_file"

    # Display table footer
    echo "+-----------------+--------+"

    rm "$tmp_icmp_file"

    echo "Enumeration using ICMP completed."
    read -p "Press Enter to return to the main menu..."
}

# Function to enumerate all machines on the network using ARP protocol
enumerate_arp() {
    read -p "Enter the network in CIDR notation (default is $default_network): " network
    network=${network:-$default_network} # Set the default value if input is empty

    echo "Enumerating all machines on the network using ARP protocol..."

    tmp_arp_file=$(arp_scan "$network")

    # Display table header
    echo "+-----------------+-------------------+--------+"
    echo "| IP Address      | MAC Address       | Status |"
    echo "+-----------------+-------------------+--------+"

    # Process each line of the arping output
    while IFS= read -r line; do
        # Extract IP and MAC from the line
        ip=$(echo "$line" | awk '{print $5}' | sed 's/[():]//g')
        mac=$(echo "$line" | awk '{print $4}')

        if [ -n "$ip" ] && [ -n "$mac" ]; then
            printf "| %-15s | %-17s | Active |\n" "$ip" "$mac"
        fi
    done <"$tmp_arp_file"

    # Display table footer
    echo "+-----------------+-------------------+--------+"

    rm "$tmp_arp_file"

    read -p "Press Enter to return to the main menu..."
}

# Function to enumerate and map machines on the network based on ICMP and ARP
enumerate_and_map() {
    declare -A machines_map
    declare -A machines_mac

    # Prompt user for network, TTL, and packet size
    read -p "Enter the network in CIDR notation (default is $default_network): " network
    network=${network:-$default_network} # Set the default value if input is empty

    IFS='/' read -r base_network prefix <<<"$network"

    read -p "Enter the TTL (Time to Live) for ICMP packets (default is 64): " ttl
    ttl=${ttl:-64}

    read -p "Enter the packet size in bytes (default is 64): " packet_size
    packet_size=${packet_size:-64}

    tmp_icmp_file=$(icmp_scan "$network" "$ttl" "$packet_size")
    tmp_arp_file=$(arp_scan "$network")

    # Populate the machines_map array from ICMP results
    while read -r ip; do
        machines_map["$ip"]="ICMP"
    done <"$tmp_icmp_file"

    # Populate the machines_map and machines_mac array from ARP results
    while IFS= read -r line; do
        ip=$(echo "$line" | awk '{print $5}' | sed 's/[():]//g')
        mac=$(echo "$line" | awk '{print $4}')
        machines_mac["$ip"]="$mac"
        if [ "${machines_map["$ip"]}" == "ICMP" ]; then
            machines_map["$ip"]="ICMP/ARP"
        else
            machines_map["$ip"]="ARP"
        fi
    done <"$tmp_arp_file"

    # Clean up temporary files
    rm "$tmp_icmp_file" "$tmp_arp_file"

    # Display the mapping results
    echo "Mapping Results:"
    # display the table with the IP address, MAC address, method, and status
    echo "+-----------------+-------------------+----------+--------+"
    echo "| IP Address      | MAC Address       | Method   | Status |"
    echo "+-----------------+-------------------+----------+--------+"
    for ip in "${!machines_map[@]}"; do
        printf "| %-15s | %-17s | %-8s | Active |\n" "$ip" "${machines_mac["$ip"]}" "${machines_map["$ip"]}"
    done
    echo "+-----------------+-------------------+----------+--------+"

    read -p "Press Enter to return to the main menu..."
}

# Function to enumerate at least 5 well-known ports on each machine and map them
enumerate_ports() {
    rm -f /tmp/portscan_traffic.pcap

    tcpdump -i eth0 -w /tmp/portscan_traffic.pcap >/dev/null 2>&1 &
    TCPDUMP_PID=$!
    # Reset the associative arrays for this run
    declare -A machines_map
    declare -A machines_mac
    declare -A machines_ports

    # Prompt user for network, TTL, and packet size
    read -p "Enter the network in CIDR notation (default is $default_network): " network
    network=${network:-$default_network} # Set the default value if input is empty

    IFS='/' read -r base_network prefix <<<"$network"
    read -p "Enter the TTL (Time to Live) for ICMP packets (default is 64): " ttl
    ttl=${ttl:-64}
    read -p "Enter the packet size in bytes (default is 64): " packet_size
    packet_size=${packet_size:-64}

    # Prompt user for additional ports
    echo "Default ports to be scanned are: 22,80,139,443,445."
    read -p "Enter additional ports to scan, separated by commas (just press Enter if none): " additional_ports
    local all_ports="22,80,443,139,445"
    if [ -n "$additional_ports" ]; then
        all_ports="$all_ports,$additional_ports"
    fi
    IFS=',' read -ra ports <<<"$all_ports"

    tmp_icmp_file=$(icmp_scan "$network" "$ttl" "$packet_size")
    tmp_arp_file=$(arp_scan "$network")

    # Populate the machines_map array from ICMP results
    while read -r ip; do
        machines_map["$ip"]="ICMP"
    done <"$tmp_icmp_file"

    # Populate the machines_map and machines_mac arrays from ARP results
    while IFS= read -r line; do
        ip=$(echo "$line" | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}')
        mac=$(echo "$line" | awk '{print $4}')

        if [ -n "$ip" ]; then # Ensures a valid IP was found
            machines_mac["$ip"]="$mac"
            if [ "${machines_map["$ip"]}" == "ICMP" ]; then
                machines_map["$ip"]="ICMP/ARP"
            else
                machines_map["$ip"]="ARP"
            fi
        fi
    done <"$tmp_arp_file"

    # Port Scanning
    for ip in "${!machines_map[@]}"; do
        open_ports=""
        for port in "${ports[@]}"; do
            timeout 0.1 bash -c "echo >/dev/tcp/$ip/$port" 2>/dev/null && open_ports+="$port,"
            #sleep 0.2 # Introduce a delay of 0.3 seconds between port checks

        done
        # Remove the trailing comma, if exists
        open_ports=${open_ports%,}
        machines_ports["$ip"]=$open_ports
        open_ports_map["$ip"]=$open_ports

    done

    # Clean up temporary files
    rm "$tmp_icmp_file" "$tmp_arp_file"

    if ps -p $TCPDUMP_PID >/dev/null; then
        kill $TCPDUMP_PID
    fi

    # Display the mapping results with the open ports
    echo "Mapping Results with Ports:"
    # Display table with IP address, MAC address, method, open ports, and status
    echo "+-----------------+-------------------+----------+----------------------+--------+"
    echo "| IP Address      | MAC Address       | Method   | Open Ports           | Status |"
    echo "+-----------------+-------------------+----------+----------------------+--------+"
    for ip in "${!machines_map[@]}"; do
        printf "| %-15s | %-17s | %-8s | %-20s | Active |\n" "$ip" "${machines_mac["$ip"]}" "${machines_map["$ip"]}" "${machines_ports["$ip"]}"
    done
    echo "+-----------------+-------------------+----------+----------------------+--------+"
    read -p "Press Enter to return to the main menu..."
}

# Function to analyze bandwidth of Option 4
analyze_pcap() {
    echo "Analyzing pcap file..."

    if [ ! -r "/tmp/portscan_traffic.pcap" ]; then
        echo "The pcap file does not exist or is not readable. Please check."
        exit 1
    fi

    # Extracting file name and displaying data rate directly from capinfos
    file_name="/tmp/portscan_traffic.pcap"
    data_rate_info=$(capinfos -i "$file_name" | grep "Data bit rate")

    if [ -z "$data_rate_info" ]; then
        echo "Unable to determine data rate from pcap file."
    else
        echo "File name:           $file_name"
        echo "$data_rate_info"
    fi

    read -p "Press Enter to return to the main menu..."
}

# Function to perform banner grabbing
banner_grabbing() {
    # Desired ports for banner grabbing
    desired_ports=(22 80)
    # Ask for IP address or network range
    read -p "Enter the IP address or network range (default is $default_network): " ip_or_network
    ip_or_network=${ip_or_network:-$default_network}

    # Check if the input is a single IP or a network range
    if [[ $ip_or_network =~ "/" ]]; then
        # It's a network range
        IFS='/' read -r base_network prefix <<<"$ip_or_network"
        base_network="${base_network%.*}"
        num_host_bits=$((32 - prefix))
        num_addresses=$((2 ** num_host_bits - 2))

        for i in $(seq 1 $num_addresses); do
            ip="$base_network.$i"

            # Use open_ports_map to get the open ports for this IP
            IFS=',' read -ra target_ports <<<"${open_ports_map["$ip"]}"

            for port in "${target_ports[@]}"; do
                # If the port is not in desired_ports, skip
                if [[ ! " ${desired_ports[@]} " =~ " $port " ]]; then
                    continue
                fi

                if [ "$port" -eq 80 ]; then
                    banner=$(echo -e "GET / HTTP/1.1\r\nHost: $ip\r\nConnection: close\r\n\r\n" | nc -w 2 $ip $port 2>&1 | head -n 5)
                else
                    banner=$(echo "" | nc -w 2 $ip $port 2>&1 | head -n 5)
                fi

                if [[ ! $banner =~ "Connection refused" && ! $banner =~ "Connection timed out" && ! $banner =~ "No route to host" && -n "$banner" ]]; then
                    printf "| %-19s | %-4s | %-44s |\n" "$ip" "$port" "$banner"
                fi
            done
        done
    else
        # It's a single IP
        # Use open_ports_map to get the open ports for this IP
        IFS=',' read -ra target_ports <<<"${open_ports_map["$ip_or_network"]}"

        for port in "${target_ports[@]}"; do
            # If the port is not in desired_ports, skip
            if [[ ! " ${desired_ports[@]} " =~ " $port " ]]; then
                continue
            fi

            if [ "$port" -eq 80 ]; then
                banner=$(echo -e "GET / HTTP/1.1\r\nHost: $ip_or_network\r\nConnection: close\r\n\r\n" | nc -w 2 $ip_or_network $port 2>&1 | head -n 5)
            else
                banner=$(echo "" | nc -w 2 $ip_or_network $port 2>&1 | head -n 5)
            fi

            if [[ ! $banner =~ "Connection refused" && ! $banner =~ "Connection timed out" && ! $banner =~ "No route to host" && -n "$banner" ]]; then
                printf "| %-19s | %-4s | %-44s |\n" "$ip_or_network" "$port" "$banner"
            fi
        done
    fi

    read -p "Press Enter to return to the main menu..."
}

# Function to enumerate reverse DNS information for hosts in the network
enumerate_reverse_dns() {
    # Prompt user for network
    read -p "Enter the network in CIDR notation (default is $default_network): " network
    network=${network:-$default_network} # Set the default value if input is empty

    IFS='/' read -r base_network prefix <<<"$network"
    IFS='.' read -ra octets <<<"$base_network" # Split the base network into octets

    # Calculate the number of addresses based on the prefix length
    num_host_bits=$((32 - prefix))
    num_addresses=$((2 ** num_host_bits))

    echo "Enumerating reverse DNS information for hosts in the network..."

    # Display table header
    echo "+---------------------+----------------------------------------------+"
    echo "| IP Address          | DNS Name(s)                                  |"
    echo "+---------------------+----------------------------------------------+"

    for i in $(seq 1 $num_addresses); do
        ip="${octets[0]}.${octets[1]}.${octets[2]}.$i"
        dns_info=$(host $ip 2>/dev/null)
        if [[ $dns_info == *"pointer"* ]]; then
            dns_names=$(echo "$dns_info" | awk '{print $NF}' | tr '\n' ',' | sed 's/,$//')
            printf "| %-19s | %-44s |\n" "$ip" "$dns_names"
        fi
    done

    # Display table footer
    echo "+---------------------+----------------------------------------------+"

    read -p "Press Enter to return to the main menu..."
}

# Main menu
while true; do
    clear
    welcome_message
    read choice

    case $choice in
    1)
        enumerate_icmp
        ;;
    2)
        enumerate_arp
        ;;
    3)
        enumerate_and_map
        ;;
    4)
        enumerate_ports
        ;;
    5)
        banner_grabbing
        ;;
    6)
        enumerate_reverse_dns
        ;;
    7)
        analyze_pcap
        ;;
    8)
        echo -e "\033[31mExiting. Goodbye!\033[0m"
        exit 0
        ;;
    *)
        echo -e "\033[31mInvalid choice. Please select a valid option.\033[0m"
        sleep 2
        ;;
    esac
done
