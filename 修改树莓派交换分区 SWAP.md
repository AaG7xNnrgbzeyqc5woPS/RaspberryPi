
[修改树莓派交换分区 SWAP](http://shumeipai.nxez.com/2017/12/18/how-to-modify-raspberry-pi-swap-partition.html)
[How to change Raspberry Pi's Swapfile Size on Raspbian](http://www.bitpi.co/2015/02/11/how-to-change-raspberry-pis-swapfile-size-on-rasbian/)

```

Commands

We will change the configuration in the file */etc/dphys-swapfile *:

sudo nano /etc/dphys-swapfile

The default value in Raspbian is:

CONF_SWAPSIZE=100

We will need to change this to:

CONF_SWAPSIZE=1024

CONF_MAXSWAP=4096 //noticed this

Then you will need to stop and start the service that manages the swapfile own Rasbian:

sudo /etc/init.d/dphys-swapfile stop
sudo /etc/init.d/dphys-swapfile start

// other ways:
sudo systemctl restart dphys-swapfile
sudo systemctl status dphys-swapfile


You can then verify the amount of memory + swap by issuing the following command:

free -m
// free -h

The output should look like:

total     used     free   shared  buffers   cached
Mem:           435       56      379        0        3       16
-/+ buffers/cache:       35      399
Swap:         1023        0     1023

Finished!

That should be enough swap to complete any future compiles I may do in the future.

```

