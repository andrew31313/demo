## Setting up a Gateway to the outside World, Careful it can be dangerous out there.. 

 
                                                                  !!!WARNING: LINUX CLI AHEAD!!!


In this lab, we will going through the steps and procedures of setting up a router, that will allow two different networks to communicat. Pay attention, because this may get a bit messy if you are not familiar with the processes we performed in lab 2, dont feel obligated to do this lab, but if you would like a challenge here it is. 


## Host setup

1. To start, we are going to need a second interface for NAT, in lab 15-2 we disabled these mainly due to the fact we were generating our own internal network (10 points if you can explain what NAT is used for and how it works). The following steps should be performed on all 3 vm's, The reason we did not clone these steps is due to keeping the first few lessons a little easier to work through. We will see the additonal steps and differneces later on in this lesson.
	
	a. As we did in lab15-2 you will want to add another interface to your machine in virtualbox, but instead of an internal network assignment we will want to assign it as NAT under ADAPTER_2 so, adapter 2 should looks as follows. (Virtual_box > Client_Server_Router > Settings > Network > ADAPTER_2). 
	
		* Enable Network Adapter 
		
		* Attached to: NAT
		
		* Cable Connected
	
2. Now that we have NAT setup, lets boot into the machine and see if it is visible, we will be introducing `nmcli` in this step
	
	a. If we run the command `ip addr` we should see 3 interfaces, our local inet (enp0s3), the NAT (enp0s8) and the local address (lo) (this might very depending on setup). If we open nmtui, we would see only 1 interface shows up that we can edit or activate. Strange right, or is it? lets dig deeper into linux networking. 

	b. lets run this command, `nmcli device`. Now we should see all 3 device, one that is "connected", one that is "diconnected" and another that is "unmanaged". 

	c. To activate this interface lets run `nmcli device connect enp0s8` (substitiue enp0s8 for what ever your disconnected interface might be). 
	
	d. repeat these steps for the server and the gateway. we will make some additonal changes to the server and the gateway below, so you might want to read ahead and edit your configuration accodingly, this is where things get a little messy. 

								    Server Setup

As mentioned in the description of the lab, we are connecting 2 `different` networks together, thus we must change the current network interface of the `server`

1. In the `Adapter 1` section we will want to change the interface to a different name (VirtualBox > Server > Settings > Network > Adapter 1)

	a. Enable Network Adapter 
	
	b. Attach the network adapter to `Internal Network`
	
	c. Name: $intnet-2
	
	d. Under advanced ensure the cable is connected
            

2. we will also want to change th ip, just to ensure we are infact on a different network so lets boot up `server` and run `nmtui`

	a. we will want to run nmtui 
	
	b. we will want to `Edit Connection`
	
	c. select enp0s3 
	
	d.edit the ip address to 10.0.1.11/8 <- !!!note the subnet!!!
	
	f.edit the gateway address to 10.0.1.1
	
	g. if you have issues pinging the ip try rebooting the machine or running `nmcli connection down enp0s3 && nmcli connection up enp0s3`


Great that should be sufficient now on to the Router

								    Router Setup
1. okay, so we got the two machines setup, so now they need to see one another. The router will need to see both device, so we will need to add intnet-2, to our network adapter on the gateway. (VirtualBox > Router > Settings > Network > Adapter 3) (Keep in mind adapter 1 and 2 should already be assigned to intnet-1 and NAT). Once in Adapter 3 we will want to 

	a. Select adapter 3
	
	b. Enable Network Adapter 
	
	c. Attach the network adapter to `Internal Network`
	
	d. Name: $intnet-2
	
	e. Under advanced ensure the cable is connected



2. Again, this is going to be our way to the outside world, so lets go to adapter 4 and add a bridge to our host! (VirtualBox > Router > Settings > Network > Adapter 4)

	a.Select Adapter 4

	b. Enable Network Adapter

	c. Attach the network adapter to `Bridged Adapter`
	
	d. Name `$en1 Wi-Fi (airport)` <- this is the local adapter on your computer
	
	e. Promiscuous Mode: `Allow VM's`

	f. [x] Cable Connected
	
4. now to connect the downed devices on the router, (intnet-2, nat and the bridge )

								Note
intnet-2 will likely fail its first go, but we have to activate it so we can run a few commands in nmcli to configure the interface approriatly. this will take place in step 3, if one of these commands hang press key combo `ctrl + c` to exit. 

	a. first we will want to get adapater 3 online, `nmcli device` will show us the interface name, ensure both NAT($enp0s8) and intnet-2($enp0s9) are up
	
	b. `nmcli device connect enp0s8` 
	
	c. `nmcli device connect enp0s9`
	
	d. `nmcli device connect enp0s10`
	
	e. nmcli device 

3. If we run `nmcli device` again we should see that $enp0s9 is pending configuration. We are going to make a few configuration changes using nmcli, ones that we have been making in nmtui, so keep your eyes peeled and see if you can decode what is taking place in these commands

	a.`nmcli connection modify enp0s9 ipv4.addresses 10.0.1.1/8` <-- !!Note the subnet!!
	
	b.`nmcli connection mod enp0s9 ipv4.method manual`
	
	c.`nmcli con mod enp0s9 ipv4.dns 8.8.8.8`
	
	d. `nmcli c mod enp0s9 ipv4.never-default true`
	
	f. `nmcli c down enp0s9`
	
	g. `nmcli c up enp0s9`

4.dont forget, this is a clone of the old system, so we need to edit the intnet1 adapter as well, we can use ip addr to see which ip has the 10.0.0.10 address assigned to it 

	a.ip addr` -> for me it is enp0s3 given intnet-1 is still assigend to adaapter 1
	
	b.`nmcli connection modify enp0s3 ipv4.addresses 10.0.0.1/8`
	
	c. `nmcli c mod enp0s3 ipv4.never-default true`
	
	c. `nmcli c mod enp0s3 ipv4.gateway 0` <- I have done this just to clear out the gateway config
	
	c. `nmcli con down enp0s3 && nmcli con up enp0s3`


	
## Packet Forwarding
To start, lets reboot all the systems, just to ensure they are up to date and that there are no  lagging configurations 

1. Now if you try running your handy dandy ping command across the vms ip addresses, you might notice some odd behaviour.. The linux vm is not setup as a router just yet, at this point it is just a desktop host, which do not forward packets. Although, from the host and the server, you should be able to ping the gatway of those two nodes (10.0.0.1 & 10.0.1.1), but agian, expect some odd behaviour. 

2. Now lets configure our router to forward IP packets.. this will give you a little more exposure to linux ;) 

	a. if we run the command `/sbin/sysctl net.ipv4.ip_forward` we would likely see the output net.ipv4.ip_forward = 0, this means ip forwarding is disabled, lets go ahead and enable that by running `/sbin/sysctl -w net.ipv4.ip_forward=1`

	b. this will disappear after reboot if you want this configuration to stick so that the router works on reboot, run `echo net.ipv4.ip_forward=1 >> /usr/lib/sysctl.d/50-defualt.conf`
		i. now we will want to reboot.
		ii. after reboot if we run `/sbin/sysctl net.ipv4.ip_forward` we should see the output of `net.ipv4.ip_forward=1`


## Firewalls
There still some configuration left. We might have been able to enbale port forwarding, but our security system seems to be blocking us so our client will still be unable to see or talk to the server
	
Security, we will have to open up some holes in our systems to allow this information to flow freely, that being said, this is only for learning purposes, anything you do on your own home network you perform at your own risk. I encourage you to make these changes on virtual machines and then destroy machines whenever you are done experementing. Dont open up more holes in your system than what is necessary. Otherwise you open yourself up to secuirty vulnerabilities.  

I found that I had to poke holes in both the router and in the server with the following commands 
	
		firewall-cmd --direct --permanent --add-rule ipv4 nat POSTROUTING 0 -o ext1 -j MASQUERADE; 

		firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i int1 -o ext1 -j ACCEPT; 

		firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i ext1 -o int1 -m state --state RELATED,ESTABLISHED -j ACCEPT;
		
		reboot


That was a lot, and don't feel down if some of this is not clear, there was a lot of linux related knowledge in there and this is a networking course, not linux. BUT, if you made it through congrats! 

