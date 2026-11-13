# Reverse Engineering

## Challenge Info

| Field          | Value                                                       |
| -------------- | ----------------------------------------------------------- |
| **Context**    | Introduction To Security (BSY) class at CTU in Prague, 2026 |
| **Challenges** | Curiouser and curiouser!; Can you stand on your head?       |
| **Category**   | Network Forensics & Reverse Engineering                     |
| **Objective**  | Capture traffic, reverse engineering,                       |
| **Tools Used** | Wireshark, xxd, dd, IDA, CyberChef                          |
## Curiouser and curiouser!
### Task Setup
Capture the traffic that comes to your container (for at least 15 minutes) and try to make sense of it. You will find your first flag in there. And do not pay attention to the traffic from past assignments.

### Task Solution
After capturing the traffic for a few minutes, I opened it in Wireshark and noticed a UDP packet containing some data.
![[/screenshots/Pasted image 20261112215629.png]]

By following the UDP stream and viewing the printable strings in hexdump, I found the text `bin 47.tar`, which clearly idicated tjat this was the file I was looking for. However, the file was malformed, so I downloaded it as `.raw` binary.
![[/screenshots/Pasted image 20261112214838.png]]

As I mentioned the `bin47.tar` file was the biggest clue. I therefore, found the magic number of the `.tar` files. The result showed the me bytes `1F 8B`, which were exactly what we were looking for.
![[/screenshots/Pasted image 20261112215455.png]]

On closer inspection of the header, I noticed that 11 different bytes comes before the `1F 8B` bytes. I therefore had to remove these additional bytes in the hope that the file will not be corrupted anymore.
![[/screenshots/Pasted image 20261112215116.png]]

I used the `dd` tool to remove the bytes, and after extracting the data, I finally found the first flag.
![[/screenshots/Pasted image 20261113101418.png]]

## Can you stand on your head?
### Task Setup
It seems that you have intercepted a secret file.
Pstt.. If you are wondering which way you ought to go, you should reverse the binary to get another flag.

### Task Solution
In the same folder as the first flag, I found a file named `bin`, which was the binary I needed to reverse engineer. When I ran it, it turned out that the program expects the flag to be passed as an argument:
![[/screenshots/Pasted image 20261113103333.png]]

I opened the file in `IDA`, generated the pseudocode and started analyzing it. The condition `strlen(s) == 10` shows that the flag must be exactly 10 characters long and function `check_flag` is the only function which is executed.
![[/screenshots/Pasted image 20261112214022.png]]

Analyzing `check_flag` showed that it simply XORs each character of the input with the corresponding byte of the array `just_an_array` and compares the result with the corresponding character of the string `bsy`. If every character matches, the flag is correct.
![[/screenshots/Pasted image 20261112214034.png]]

By inspecting both variables, I found that `bsy` is simply the string `?reversed!`, while `just_an_array` is an array of 10 bytes, displayed in hexadecimal: `79 17 17 04 1C 37 1D 0F 0B 58`.
![[/screenshots/Pasted image 20261112214052.png]]

I used **CyberChef** for the XOR operation, and the output was the 10-character string `FerryEnjoy`. To verify the result, I ran the binary with this string as an argument, and the program confirmed that it was the correct flag.
![[/screenshots/Pasted image 20261112213938.png]]
![[/screenshots/Pasted image 20261112223641.png]]