# The Story

![](./res/pic1.png)

Check out Saqib's  video walkthrough for Day 20 [here](https://www.youtube.com/watch?v=1qc7C4h36ZQ)!

We can now learn more about the mysterious device found in Santa's workshop. Elf Forensic McBlue has successfully been able to find the `device ID`. Now that we have the hardware `device ID`, help Elf McSkidy reverse the encrypted firmware and find interesting endpoints for IoT exploitation.

# Learning Objectives
- What is firmware reverse engineering
- Techniques for extracting code from the firmware
- Extracting hidden keys from an encrypted firmware
- Modifying and rebuilding a firmware

# What is Firmware Reverse Engineering
Every embedded system, such as cameras, routers, smart watches etc., has pre-installed firmware, which has its own set of instructions running on the hardware's processor. It enables the **hardware to communicate with other software running on the device**. The firmware provides low-level control for the designer/developer to make changes at the root level. 

Reverse engineering is working your way back through the code to figure out how it was built and what it does. Firmware reverse engineering is extracting the original code from the firmware binary file and verifying that the code does not carry out any malicious or unintended functionality like undesired network communication calls. **Firmware reversing is usually done for security reasons** to ensure the safe usage of devices that may have critical vulnerabilities leading to possible exploitation or data leakage. Consider a smart watch whose firmware is programmed to send all incoming messages, emails etc., to a specific IP address without any indication to the user.

# Firmware Reversing Steps
- The firmware is first obtained from the vendor's website or extracted from the device to perform the analysis.
- The obtained/extracted firmware, usually a binary file, is first analysed to figure out its type (bare metal or OS based). 
- It is verified that the firmware is either encrypted or packed. The encrypted firmware is more challenging to analyse as it usually needs a tricky workaround, such as reversing the previous non-encrypted releases of the firmware or performing hardware attacks like [Side Channel Attacks (SCA)][1] to fetch the encryption keys. 
- Once the encrypted firmware is decrypted, different techniques and tools are used to perform reverse engineering based on type.

# Types of Firmware Analysis
Firmware analysis is carried out through two techniques, Static & Dynamic.

## Static Analysis
Static analysis involves an essential examination of the binary file contents, performing its reverse engineering, and reading the assembly instructions to understand the functionality. This is done through multiple commonly used command line utilities and binary analysis tools such as:
- [BinWalk][2]: A firmware extraction tool that extracts code snippets inside any binary by searching for signatures against many standard binary file formats like `zip, tar, exe, ELF,` etc. Binwalk has a database of binary header signatures against which the signature match is performed. The common objective of using this tool is to extract a file system like `Squashfs, yaffs2, Cramfs, ext*fs, jffs2,` etc., which is embedded in the firmware binary. The file system has all the application code that will be running on the device.
- [Firmware ModKit (FMK)][3]: FMK is widely used for firmware reverse engineering. It extracts the firmware using `binwalk` and outputs a directory with the firmware file system. Once the code is extracted, a developer can modify desired files and repack the binary file with a single command. 
- [FirmWalker][4]: Searches through the extracted firmware file system for unique strings and directories like `etc/shadow`, `etc/passwd`, `etc/ssl`, special keywords like `admin, root, password,` etc., vulnerable binaries like `ssh, telnet, netcat` etc.

## Dynamic Analysis
Firmware dynamic analysis involves running the firmware code on actual hardware and observing its behaviour through emulation and hardware/ software based debugging. One of the significant advantages of dynamic analysis is to analyse unintended network communication for identifying data pilferage. The following tools are also commonly used for dynamic analysis:
- [Qemu][5]: Qemu is a free and open-source emulator and enables working on cross-platform environments. The tool provides various ways to emulate binary firmware for different architectures like Advanced RISC Machines (ARM), Microprocessors without Interlocked Pipelined Stages (MIPS), etc., on the host system. Qemu can help in full-system emulation or a single binary emulation of ELF (Executable and Linkable Format) files for the Linux system and many different platforms.
- [Gnu DeBugger (GDB)][6]: GDB is a dynamic debugging tool for emulating a binary and inspecting its memory and registers. GDB also supports remote debugging, commonly used during firmware reversing when the target binary runs on a separate host and reversing is carried out from a different host.

# Shall We Reverse the Firmware? Let's Do It!
Launch the virtual machine by clicking `Start Machine` at the top right of this task. The machine will load in a split-screen view. If it does not, click the `Show Split View` button to view the machine. Wait for 1-2 minutes for the machine to load completely. In case you refresh your web browser page, it will appear as if the target machine has restarted/rebooted (as it shows the default terminal output that is shown after booting up). In reality, the machine state remains the same, and your progress is not lost. When reversing the firmware, use the password `Santa1010` if prompted for a **sudo** password. 

After identifying the device id, McSkidy extracted the encrypted firmware from the device. To reverse engineer it, she needs an unencrypted version of the firmware first - luckily, she found it online. Open the terminal and run the `dir` command. You will see the following directories:

```
ubuntu@machine$ dir
bin  firmware-mod-kit  bin-unsigned
```

The `bin` folder contains the firmware binary, while the `firmware-mod-kit` folder contains the script for extracting and modifying the firmware.
In this exercise, we will primarily be using two tools:
- **Binwalk**: For verifying encryption and can also be used to decrypt the firmware (Usage: `binwalk -E -N`)
- **Firmware Mod Kit (FMK)**: Library for firmware extraction and modification (Usage: `extract-firmware.sh`)

Now coming over to the task, we will perform reversing step by step.

## Step 1: Verifying Encryption
In this step, McSkidy will verify whether the binary `firmwarev2.2-encrypted.gpg` is encrypted through [file entropy analysis][7]. First, change the directory to the `bin` folder by entering the command `cd bin`. She will then use the `binwalk` tool to verify the encryption using the command `binwalk -E -N firmwarev2.2-encrypted.gpg`.

```
ubuntu@machine:-/bin$ binwalk -E -N firmwarev2.2-encrypted.gpg 

DECIMAL       HEXADECIMAL     ENTROPY
--------------------------------------------------------------------------------
0             0x0             Rising entropy edge (0.988935)
```

In the above output, the `rising entropy edge` means that the file is probably encrypted and has increased randomness. 

## Step 2: Finding Unencrypted Older Version
Since the latest version is encrypted, McSkidy found an older version of the same firmware. The version is located in the `bin-unsigned` folder. Why was she looking for an older version? Because she wants to find encryption keys that she may use to decrypt the original firmware and reverse engineer it. McSkidy has decided to use the famous `FMK` tool for this purpose. To extract the firmware, change the directory by entering the command `cd ..` and then `cd bin-unsigned`. She extracted the firmware by issuing the following command.

```
extract-firmware.sh firmwarev1.0-unsigned 
```

![](./res/pic2.png)

## Step 3: Finding Encryption Keys
The original firmware is [gpg][8] protected, which means that we need to find a public, and private key and a paraphrase to decrypt the originally signed firmware. We know that the unencrypted firmware is extracted successfully and stored in the `fmk` folder. The easiest way to find keys is by using the `grep` command. The `-i` flag in the grep command ignores case sensitivity while the `-r` operator recursively searches in the current directory and subdirectories.  

```
ubuntu@machine:~bin-unsigned$ grep -ir key
Binary file firmwarev2.2-encrypted.gpg matches
Binary file firmwarev1.0-unsigned matches
Binary file fmk/image_parts/rootfs.img matches
Binary file fmk/rootfs/usr/sbin/dropbearmulti matches
Binary file fmk/rootfs/usr/sbin/dhcp6ctl matches
Binary file fmk/rootfs/usr/sbin/dhcp6c matches
Binary file fmk/rootfs/usr/sbin/xl2tpd matches
Binary file fmk/rootfs/usr/sbin/pppd matches
Binary file fmk/rootfs/usr/sbin/dhcp6s matches
Binary file fmk/rootfs/usr/bin/httpd matches
fmk/rootfs/gpg/public.key:-----BEGIN PGP PUBLIC KEY BLOCK-----
fmk/rootfs/gpg/public.key:-----END PGP PUBLIC KEY BLOCK-----
fmk/rootfs/gpg/private.key:-----BEGIN PGP PRIVATE KEY BLOCK-----
fmk/rootfs/gpg/private.key:-----END PGP PRIVATE KEY BLOCK-----
```

Bingo! We have the **public and private keys**, but what about the **paraphrase** usually used with the private key to decrypt a gpg encrypted file?

Let's find the paraphrase through the same `grep` command.

```
ubuntu@machine:~bin-unsigned$ grep -ir paraphrase
fmk/rootfs/gpg/secret.txt:PARAPHRASE: [OUTPUT INTENTIONALLY HIDDEN]
```

McSkidy has finally located the public and private keys and the paraphrase as well.

## Step 4: Decrypting the Encrypted Firmware
Now that we have the keys, let's import them using the following command:

```
ubuntu@machine:~bin-unsigned$ gpg --import fmk/rootfs/gpg/private.key 
gpg: key 56013838A8C14EC1: "McSkidy " not changed
gpg: key 56013838A8C14EC1: secret key imported
gpg: Total number processed: 1
gpg:              unchanged: 1
gpg:       secret keys read: 1
gpg:  secret keys unchanged: 1
```

While importing the private key, you will be asked to enter the paraphrase. Enter the one you found in **Step 3**.

Importing the public key.

```
ubuntu@machine:~bin-unsigned$ gpg --import fmk/rootfs/gpg/public.key 
gpg: key 56013838A8C14EC1: "McSkidy " not changed
gpg: Total number processed: 1
gpg:              unchanged: 1
```

We can list the secret keys.

```
ubuntu@machine:~bin-unsigned$ gpg --list-secret-keys
/home/ubuntu/.gnupg/pubring.kbx
-------------------------------
sec   rsa3072 2022-11-17 [SC] [expires: 2024-11-16]
      514B4994E9B3E47A4F89507A56013838A8C14EC1
uid           [ unknown] McSkidy 
ssb   rsa3072 2022-11-17 [E] [expires: 2024-11-16]
```

Once the keys are imported, McSkidy decrypts the firmware using the `gpg` command. Again change the directory by entering the command `cd ..` and then `cd bin`.

```
ubuntu@machine:~bin$ gpg firmwarev2.2-encrypted.gpg
gpg: WARNING: no command supplied.  Trying to guess what you mean ...
gpg: encrypted with 3072-bit RSA key, ID 1A2D5BB2F7076FA8, created 2022-11-17
      "McSkidy "
```

After decryption, once you issue the `ls` command, the decrypted file result will look like the following:

```
ubuntu@machine:~bin$ ls -lah

-rw-rw-r-- 1 test test 3.9M Dec  7 19:10 firmwarev2.2-encrypted
-rw-rw-r-- 1 test test 3.6M Dec  1 05:45 firmwarev2.2-encrypted.gpg
```

## Step 5: Reversing the Original Encrypted Firmware
This is the simplest step, and we can use `binwalk` or `FMK` to extract code from the recently unencrypted firmware. In this example, we will be using `FMK` to extract the code.

```
extract-firmware.sh firmwarev2.2-encrypted
```

![](./res/pic3.png)

McSkidy has finally been able to reverse the complete firmware and extract essential files she will use for IoT exploitation (next room). She has used the keys from an older version (1.0) to decrypt the latest version (2.2) of the same firmware. The `Camera` folder in the `fmk/rootfs` directory will contain all the necessary files we will be using in the next task.

[1]:https://en.wikipedia.org/wiki/Side-channel_attack
[2]:https://github.com/ReFirmLabs/binwalk
[3]:https://www.kali.org/tools/firmware-mod-kit/
[4]:https://github.com/craigz28/firmwalker
[5]:https://www.qemu.org/
[6]:https://www.sourceware.org/gdb/
[7]:https://en.kali.tools/?p=1634
[8]:https://en.wikipedia.org/wiki/GNU_Privacy_Guard

===============================================================================

# Questions

> What is the flag value after reversing the file firmwarev2.2-encrypted.gpg?  
> **Note**: The flag contains underscores - if you're seeing spaces, the underscores might not be rendering.

    Answer: THM{WE_GOT_THE_FIRMWARE_CODE}

> What is the Paraphrase value for the binary firmwarev1.0_unsigned?

    Answer: Santa@2022

> After reversing the encrypted firmware, can you find the build number for **rootfs**?

    Answer: 2.6.31

> Did you know we have a wonderful community [on Discord][9]? If you join us there, you can count on nice conversation, cyber security tips & tricks, and room help from our mods and mentors. Our Discord admin has some rooms out, too - you can try an [easy one][10] or a [hard one][11]!

    This task has no answer needed.

[9]:https://discord.gg/tryhackme
[10]:https://tryhackme.com/room/githappens
[11]:https://tryhackme.com/room/shaker

===============================================================================

To begin, deploy the virtual machine. Then open the split view and follow the walkthrough to find all the answers.

### Key points from the Challenge Walkthrough below.

**NOTE**: for any sudo command, do `Santa1010` as the password.

Paraphrase used with thep rivate key to decrype a gpg encrypted file:

![](./res/paraphrase.png)

After getting the paraphrase, import the gpg key using the following:  
`gpg --import fmk/rootfs/gpg/private.key`  
then enter the paraphrase:

![](./res/key_import.png)

Once imported, go back to `bin` folder, and decrypt the firmware using the following:  
`gpg firmwarev2.2-encrypted.gpg`

Finally run the following to reverse and extract the firmware:  
`extract-firmware.sh firmwarev2.2-encrypted`

![](./res/reverse_successful.png)

For the flag, run the following after reversing the firmware:  
`cat ~/bin/fmk/rootfs/flag.txt`

![](./res/flag.png)

For the build number, run the following:  
`ls -la ~/bin/fmk/rootfs/* | grep -i "build"`

![](./res/build_number.png)