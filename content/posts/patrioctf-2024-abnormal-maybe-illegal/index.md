+++
title = 'PatriotCTF 2024 Abnormal Maybe Illegal'
date = 2024-09-24T23:46:16+02:00
categories = ['PatriotCTF 2024', 'Forensics']
+++

## Intro

This post covers *Abnormal Maybe Illegal* from *PatriotCTF 2024*. The description of the challenge is:

> We have recently discovered tons of traffic leaving our network. We have reason to believe they are using an abnormal method. Can you figure out what data they are exfiltrating?

Furthermore, two additional hints were released throughout the competition:

> TCP packets are constructed in a way, where certain combinations are possible/(legal) and others should raise alerts/(illegal)

> Now that you found the illegal packets, think more black and white

We'll take both of these into consideration, since no team had solved the challenge before both of these hints were dropped.

## Analyzing The Packet Capture File

We'll use [Scapy](https://github.com/secdev/scapy) to programmatically analyze the pcapng file we're given. Based on the hint *"TCP packets are constructed in a way, where certain combinations are possible/(legal) and others should raise alerts/(illegal)"* we should be looking for illegal TCP packets. One of the reasons why a packet could be considered illegal by the receiving server is due to having illogical combination of TCP control bits, commonly known as *flags*. 

The TCP flags are contained in the TCP header, which is defined by [RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293#name-header-format).

![Alt text](image.png)

Each flag is represented by one bit in the TCP header. let's count how many times each TCP flag appears in the capture.

```python
from scapy.all import rdpcap, TCP

packets = rdpcap('./abnormal_illegal.pcapng')

fin_count = 0
syn_count = 0
rst_count = 0
psh_count = 0
ack_count = 0
urg_count = 0
ece_count = 0
cwr_count = 0

def count_tcp_flags(tcp_layer):
    global fin_count, syn_count, rst_count, psh_count, ack_count, urg_count, ece_count, cwr_count
    
    flags = tcp_layer.flags

    if flags & 0x01:  # FIN
        fin_count += 1
    if flags & 0x02:  # SYN
        syn_count += 1
    if flags & 0x04:  # RST
        rst_count += 1
    if flags & 0x08:  # PSH
        psh_count += 1
    if flags & 0x10:  # ACK
        ack_count += 1
    if flags & 0x20:  # URG
        urg_count += 1
    if flags & 0x40:  # ECE
        ece_count += 1
    if flags & 0x80:  # CWR
        cwr_count += 1

for packet in packets:
    if TCP in packet:
        tcp_layer = packet[TCP]
        count_tcp_flags(tcp_layer)

print(f"Number of packets with FIN flag: {fin_count}")
print(f"Number of packets with SYN flag: {syn_count}")
print(f"Number of packets with RST flag: {rst_count}")
print(f"Number of packets with PSH flag: {psh_count}")
print(f"Number of packets with ACK flag: {ack_count}")
print(f"Number of packets with URG flag: {urg_count}")
print(f"Number of packets with ECE flag: {ece_count}")
print(f"Number of packets with CWR flag: {cwr_count}")
```

```
vboxuser@kali /mnt/repos/patriotctf-2024/abnormal-maybe-illegal $ python3 count-each-flag.py
Number of packets with FIN flag: 9712
Number of packets with SYN flag: 9713
Number of packets with RST flag: 81
Number of packets with PSH flag: 14434
Number of packets with ACK flag: 38337
Number of packets with URG flag: 0
Number of packets with ECE flag: 0
Number of packets with CWR flag: 0
```

As we can see, there are only FIN, SYNC, RST, PSH and ACK flags set in the whole capture, so we won't be worrying about the rest.

The TCP flags are responsible for managing the state of a TCP connection. While there's no official list of illegal flag combinations, some pairs don't make any logical sense to appear together. While there are many such combinations possible, we'll focus on the most known one which is SYN + FIN. The SYN flag is used to initiate a connection, while FIN is used to terminate it, so obviously both of them being present at the same time is contradictory. We'll check if this combination appears in our capture.

```python
from scapy.all import rdpcap, TCP

packets = rdpcap('./abnormal_illegal.pcapng')

fin_syn_packets_count = 0

def is_fin_syn(tcp_layer):
    flags = tcp_layer.flags
    # SYN + FIN (illegal combination)
    if flags & 0x03 == 0x03:
        return True
    return False

for packet in packets:
    if TCP in packet:
        tcp_layer = packet[TCP]
        if is_fin_syn(tcp_layer):
            fin_syn_packets_count += 1

print(f"Number of FIN+SYN packets: {fin_syn_packets_count}")
```

```
vboxuser@kali /mnt/repos/patriotctf-2024/abnormal-maybe-illegal $ python3 count-syn-fin.py
Number of FIN+SYN packets: 128
```

Great, there are 128 such packets.

## Uncovering The Hidden Message

Since we already know only RST, PSH and ACK flags appear in the capture besides FIN and SYN, let's check if they all appear in the illegal packets found.

```python
from scapy.all import rdpcap, TCP

packets = rdpcap('./abnormal_illegal.pcapng')

rst_count = 0
psh_count = 0
ack_count = 0

def is_fin_syn(tcp_layer):
    flags = tcp_layer.flags
    # SYN + FIN (illegal combination)
    if flags & 0x03 == 0x03:
        return True
    return False

for packet in packets:
    if TCP in packet:
        tcp_layer = packet[TCP]
        if is_fin_syn(tcp_layer):
            if tcp_layer.flags & 0x04:  # RST flag
                rst_count += 1
            if tcp_layer.flags & 0x08:  # PSH flag
                psh_count += 1
            if tcp_layer.flags & 0x10:  # ACK flag
                ack_count += 1

print(f"Number of packets with FIN+SYN and RST flag: {rst_count}")
print(f"Number of packets with FIN+SYN and PSH flag: {psh_count}")
print(f"Number of packets with FIN+SYN and ACK flag: {ack_count}")
```

```
vboxuser@kali /mnt/repos/patriotctf-2024/abnormal-maybe-illegal $ python3 count-flags-in-fin-syn.py 
Number of packets with FIN+SYN and RST flag: 80
Number of packets with FIN+SYN and PSH flag: 58
Number of packets with FIN+SYN and ACK flag: 0
```

ACK doesn't appear in packets with both FIN and SYN set in our capture.

The second hint *"Now that you found the illegal packets, think more black and white"* steers us towards thinking that the exfiltrated message is encoded in binary. Each illegal packet could carry 2 bits (RST and PSH) of the exfiltrated message. We'll take these 2 bits and concatenate them both into a binary and ASCII form.

```python
from scapy.all import rdpcap, TCP

pcap_file = 'abnormal_illegal.pcapng'
packets = rdpcap(pcap_file)

def is_fin_syn(tcp_layer):
    flags = tcp_layer.flags
    # SYN + FIN (illegal combination)
    if flags & 0x03 == 0x03:
        return True
    return False

def extract_flags(tcp_layer):
    flags = tcp_layer.flags
    rst = 1 if flags & 0x04 else 0
    psh = 1 if flags & 0x08 else 0
    return  f"{rst}{psh}"

combined_binary_string = ''

for packet in packets:
    if TCP in packet:
        tcp_layer = packet[TCP]
        if is_fin_syn(tcp_layer):
            combined_binary_string += extract_flags(tcp_layer)

def binary_to_ascii(binary_string):
    ascii_text = ''
    for i in range(0, len(binary_string), 8):
        byte = binary_string[i:i+8]
        if len(byte) == 8:
            ascii_text += chr(int(byte, 2))
    return ascii_text

decoded_message = binary_to_ascii(combined_binary_string)

print(f"Combined Binary String: {combined_binary_string}")
print(f"Decoded Message: {decoded_message}")
```

```
vboxuser@kali /mnt/repos/patriotctf-2024/abnormal-maybe-illegal $ python3 concatenate-rst-psh.py
Combined Binary String: 1011000010010011101110001001100110110111100100101001000110011101100111111011000110011110100100101001110010101111100110011001110010010010100110111011001110101111100100101011000110011010101011111001011010011100100111001001101010011011100100101001110010111110
Decoded Message: °¸·±¯³¯±¯¾
```

That didn't work. However, there's still the possibility that the bits from each packet could be parsed in reversed order, so we try that as well.

```python
from scapy.all import rdpcap, TCP

pcap_file = 'abnormal_illegal.pcapng'
packets = rdpcap(pcap_file)

def is_fin_syn(tcp_layer):
    flags = tcp_layer.flags
    # SYN + FIN (illegal combination)
    if flags & 0x03 == 0x03:
        return True
    return False

def extract_and_reverse_flags(tcp_layer):
    flags = tcp_layer.flags
    rst = 1 if flags & 0x04 else 0
    psh = 1 if flags & 0x08 else 0
    return  f"{psh}{rst}" # We reversed this compared to the previous script

combined_binary_string = ''

for packet in packets:
    if TCP in packet:
        tcp_layer = packet[TCP]
        if is_fin_syn(tcp_layer):
            combined_binary_string += extract_and_reverse_flags(tcp_layer)

def binary_to_ascii(binary_string):
    ascii_text = ''
    for i in range(0, len(binary_string), 8):
        byte = binary_string[i:i+8]
        if len(byte) == 8:
            ascii_text += chr(int(byte, 2))
    return ascii_text

decoded_message = binary_to_ascii(combined_binary_string)

print(f"Combined Binary String: {combined_binary_string}")
print(f"Decoded Message: {decoded_message}")
```

```
vboxuser@kali /mnt/repos/patriotctf-2024/abnormal-maybe-illegal $ python3 concatenate-rst-psh-reversed.py 
Combined Binary String: 0111000001100011011101000110011001111011011000010110001001101110011011110111001001101101011000010110110001011111011001100110110001100001011001110111001101011111011000010111001001100101010111110110100101101100011011000110010101100111011000010110110001111101
Decoded Message: pctf{abnormal_flags_are_illegal}
```

We got the flag! *pctf{abnormal_flags_are_illegal}*

## References
- https://datatracker.ietf.org/doc/html/rfc9293