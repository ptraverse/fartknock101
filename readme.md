# FartKnocker Solution

aka Learning to be a script kiddie 101

From [https://www.vulnhub.com/entry/tophatsec-fartknocker,115/]
Prerequisites:
* OracleVM VirtualBox
* The FartKnocker.ova file loaded into oracle box (File->Import Appliance), up and running (cant login)
* Another oracleVM box of any linux distro that you can control to attack the fartKnocker

#### 1 - arp-scan

identify IP - many methods (easiest is `arp`)

`arp`

192.168.1.254 and .71

`nmap -A` to list open ports

`nmap -A 192.168.1.254`

<code>

PORT     STATE    SERVICE     VERSION

20/tcp   filtered ftp-data

21/tcp   filtered ftp

22/tcp   filtered ssh

23/tcp   filtered telnet

80/tcp   open     tcpwrapped

</code>

80 is http --> lets see what he's hosting --> in browser = http://192.168.1.254
its the router!? haha idiot wrong IP

`nmap -A 192.168.1.71` !? nope

`sudo apt-get install arp-scan` Important Tool!
`sudo arp-scan --interface eth0 --localnet`
	Interface: eth0, datalink type: EN10MB (Ethernet)
	Starting arp-scan 1.8.1 with 256 hosts (http://www.nta-monitor.com/tools/arp-scan/)
	192.168.1.70	08:00:27:3d:0d:c8	CADMUS COMPUTER SYSTEMS
	192.168.1.71	5c:51:4f:53:b3:99	(Unknown)
	192.168.1.67	b8:27:eb:a7:45:1a	(Unknown)
	192.168.1.64	6c:ad:f8:d0:ae:e2	(Unknown)
	192.168.1.68	6c:88:14:dd:9a:80	(Unknown)
	192.168.1.69	ec:35:86:06:7b:97	(Unknown)
	192.168.1.254	20:76:00:a3:8c:c8	(Unknown

i can see router page .. process of elimination .. 192.168.1.70

80->open go to that address in browser -> get pcap file!

###### SIDE NOTE
i shouldn't have just gone right away with a browser since i dont trust the source and dont want to get hacked myself (duh)

To be safe, first inspect it with ? wget+cat? moving along...

#### 2 - Wireshark

wireshark lets you open up and look directly at packets. Important Tool!

open up pcap file in wireshark! :) (install separately)

Looks like mr .101 and .102 are talking to each other. What are all these rows and columns?
* ICMP echo's are nothing important unless there's billions of only them (literally what you send when you ping - to determine network latency)
* ARP's are Address Resolution Protocol and are packets between clients and router to decide who is who. (can be important depending on what you are doing)
* TCP and UDP are what all the actual internet "data" layer lives (important stuff lives here!)
* MDNS? Looks like its related to ARP - Stands for multicast DNS (probably not important)
* DHCP - Moar Netowkring

You can also view all ports in a conversation within wireshark. makes life easy. Plenty of other fun things you can do here.

The only interesting thing in here are a few TCP UDP packets so we'll check them out!

First pair of TCP is SynAck/Ack on .101:8888 to .102:41072 !? interesting. SynAck/Ack === 3 way handshake in TCP

###### SIDE NOTE - SynFinAck are 3 T/F bits in the header of the packet usually on a connection start/finish
* Syn = Synchronize/Initiate connection
* Fin = Final/Terminate connection
* Ack = Acknowledge received data
* [More Theory](http://packetlife.net/blog/2010/jun/7/understanding-tcp-sequence-acknowledgment-numbers/)

Next bunch is .101:9000 to .102:5563

Next bunch is .101:8000 to .102:43634

Next bunch is .101:7000 to .102:42770

They're all some kind of synAck so whats going on here. We want to check out these ports.

#### 3 - Netcat

"Netcat aka nc is a computer networking service for reading from and writing to network connections using TCP or UDP". Important Tool!

`netcat 192.168.1.70 7000 -v`

netcat: connect to 192.168.1.70 port 7000 (tcp) failed: Connection refused

`netcat 192.168.1.70 8000 -v`

netcat: connect to 192.168.1.70 port 8000 (tcp) failed: Connection refused

`netcat 192.168.1.70 9000 -v`

netcat: connect to 192.168.1.70 port 9000 (tcp) failed: Connection refused

`netcat 192.168.1.70 8888 -v`

Connection to 192.168.1.70 8888 port [tcp/*] succeeded!

/burgerworld/

Woohoo! Now check out that port in the browser? wget+cat to be safe?

No Results. --> the text /burgerworld/ is an artificial clue ..

so go there it is pcap2 file the browser - sure enough there's more pcap files to play with in wireshark

Here we have some new packets, now with DHCP - another networking protocol, shouldn't matter

Some big fatty data packets at the bottom have extra big size. ports are 8080/44299. One of those packets has a PSH bit set. RFC 793: "When a receiving TCP sees the PUSH flag, it must not wait for more data from the sending TCP before passing the data to the receiving process."

Follow TCP Stream in WireShark and you get a nice asci art of beavis


	                      MMMMMMM           MMMMMMH
	                HMMMMM:::::::.MMMMMMMMMM:::::.TMM
	              MMMI:::::::::::::::::::MMH::::::::TM
	            MMIi::::::::::::.:::::::::::::::::::::MMMM
	           MT::::.::::::::::::::::::::::::::::::.::=T.IMMM
	         MMMi:::::::::::::::::::::::::::::::::::::::::::MT)MM
	     MMMI.:::::::::::::::::::::::::::::::::::::::::::.:::M= MM
	   XMXi::::::::::::::::::::::.:::::::::::::::::::::::::::::::=MM
	   MMi::::::::::::::::::::::::::::::::::::::::::::::::::.::..:=MMM
	  MM:MMT:::::::::.:::::::::::::::::.:::::::::::::::::::::::::::MiMM
	   MMM::::::::::::::::::.::::::::::::::::::::::::::.::::::::::.TM.MM
	   MMi::::::::::::::.::::::::::::::::::::::::::::::::::::::.:::.:: M
	   MM:::.::::::::::::::::::::::::::::::::.:.:::::::::::::::::::::: XM
	 MM:MT::.::::::::::::::::::::::::::::::::::::::::::::::::::::::::::XM
	IMM:::.::::::::::::::::::::::::::::::::::: :::::::::::::::::::::::.=M
	 MM::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::: :::M
	 XMT:::::::::::::::::::::: ::::::::::::::::: : ::::::::::::::::::: iM
	   MiMi:::::::::: :::::::::::::::::::::::::::::::: ::::::::::::::.:IM
	     M::::::HH::::::::::::::::::::::::::::::::::::::::::::::::::::: M
	     MT:::::iM::::::::Hi:iXH:::ii::XH:::::::::::::.::::::::::::::.:.M
	      MX:::::iMX:i::::iMi:iMH::XH::Mi:::::::::::::::::::::::::::::: M
	        Mii::::HMH:::::iMH::MH=:MM=TMi::::::::::::::::::::::::::::::MM
	          MMMMMMMMMMMXTi:MMHi:HMMIMMMMii::::::::::::::::::::::::::::XM
	           XXOXMMT:. ::T= :IMMMMMMM=iXMii:::::::::::::::::::::::::: MM
	            MMMH:::.:::::::.::::.::::.:XMi::::::::::::::::::::::::::MM
	           XMM::.:.:..::..:.:.::.:.::: ::XMi::::::::::::::::::::::::MX
	          XMMT::::.:.::.::::.::.::::::::.::XH:::::::::::::::::::::: M
	          HMX::...:..::..:.:.::::::..... :::XX::::::::::::::.:::::. M
	          MM:::....:::::.::::::..:::::.:..:::HX::::::::::::::::::::=M
	          MX::::::::::::::::::::::..::::.:..::X::::::::::::::::::::IM
	         XMI..  .:.::....:..::::.:: ::...::.:.MH:::::::::::::::::.: M
	         MM:. ::..::....::.::::::....:.:...:..MT::.   ::::::::: :..IM
	         MM=:::::.::.:::::..::::.: .::..::..::Mi:::::::::::::::::: MM
	         MMI:::...:  .::..::::::.:::::::.::::TM:::::::::::::::::::=MO
	          MH.: .::::.::.. .:::::iLMXX=::::.:.Mi::::::: ::::::::::.MM
	          MX:.:..:: .:.:.:.: :MMM:::..:::::.HM:::: :::::::::::::.MM
	          MM:::...::....: ::IMT:::.:...:.::.MT::::::: ::::::::: MM
	           M=::..::::..:::MM:i:..::.:...: ::M:::: ::: ::::::::::MI
	           MH::: :.:.: MMMM=:::.:.:...:....iM::: ::::::  ::::::LM
	          MMMMT.::. ::TM:::::..::::::::.::.IM::::HH:::::::::::.MO
	           MM:LM::T:MT.:: .......:....:.:: TMMiXMT.MH:::.::::.:M=
	            M:. :::MMi:::MMMM=::::::.::..::=MMMMMMXMH:::.:::::MM
	           XMI: :..::=MX  :M::.......:...:::.MXTHM MH:::.: :.XM
	           MM XMMI IM    M   ................:: :MIIM:::::::MMO
	            MMXXMILM  .ML.= :.:::....:.:..::.:..:::MMT:::::TMM
	              MXMLMMMT::.:...:........ ....::.:.=.MMMM:::::MM
	              MHM=:: :.:::...::::.:...:.....:: =MMM==Mi::::M
	              MM=:::.......:.:.::.:.::...:.: ::  . ::=M:: MM
	             MMi:=XMMMi::::...:::::.::.:::::::::..: ::Mi:=MT
	            MM=:I::  :iMH==:::::.::.:::::::::::::::.::MT:XMT
	           MT=:=MMMMMMM=HM::::.::::::MMT=Mi::::::..:::MI=MM
	          M ::::::.=I= .MX:..: ::::.::MX::::.:::.:.  .XMMM
	         M:MMMMMMM=.::::  ::.::...:.MMIM::.:::.::..::::M
	                 M=:: : ::::.==XMMM:XMMM=:::.::.:.::::.M
	                 M=.IMMM )X   M  MMMMMM=:::..::..:::.::M
	                 MM  X  MMM:MMMMMMMMM=:::.:.:.. .:.::::M
	                  MIMMMMMMMMMMMMMMI::::::::.:::.:...:.:M
	                MMMMMMMMMMMMMX:.   .:..::....:...:::.:iM
	               MMMMMMMMMMI::::::.:.::...:....:.....:.:=M
	           MMMMMMMMMI:::::.:.. :.::.::..........:..:..:M
	            M=:  :..::..::.........::.......::.:.....: M
	             MMMi::::::.:.:==MMMMMMMMMT:.:.:::..:::..: OM
	               MM=::..: OMMMM         MMMT:::....:.::: :M
	                M=::::MM                MMI:::........:OM
	                 MMMMM                   MMH:::..::MMMMMM
	                                          MMMMMMMMMMMMMMM


                     CAN YOU UNDERSTAND MY MESSAGE?!



...  eins drei drei sieben


# 4 - knock-knock

That text above is german for ... 1337

So, lets TCP check that port with netcat again... Nothing. Give up? No! I am the great cornholio!! Move on to some better tools: knock-knock (name of the thing is fartknocker...)
* [knock-knock](http://www.shortbus.ninja/default-knockd-cloaking-configurations/)
* knockd is basically a tool to enable portknocking, which means you only open a port to a client if they "knock" in a special sequence with syn/acks on other ports. this tool knock-knock will kick its ass. Lesson- don't use knockd because it's a piece of shit :) But it's better than nothing. It's usually a good idea to hide your port 22 (SSH), that is if you must keep it open at all (VPN is better way to secure your server).
* Get it from [github](https://github.com/hack1thu7ch/knock-knock.git) `git clone https://github.com/hack1thu7ch/knock-knock.git`
* after running setup.sh, i try to use `python knockdefault.py` but get an error on python module scapy
* `sudo apt-get install python-pcapy`
* this is getting annoying... having to [do it the hard way](http://www.secdev.org/projects/scapy/doc/installation.html)

`sudo ./knockdefault.py ../iplist.txt output.txt`

[-] Scanning 192.168.1.70 with Nmap, this could take a minute...go get some coffee

#### Time for bed more tomorrow
