# **TrustedBSD-Installing-Biba-Policy**
<br><br>

What is Biba? Biba is a Multi-Level Security(MLS) policy model for protecting integrity. Integrity clearances of subjects and integrity classifications of objects can be different from their confidentiality clearances and classifications. For example, a highly confidential document classified as Top Secret for confidentiality may come from sources that may not have been fully vetted, and hence the document may only have an integrity classification of C or U. An example below explains the Biba rules more closely.
<br><br>

Biba Summary: "No write up, no read down"<br><br>

Biba Write rule: A subject with an integrity-clearance level of ```high``` may only have write access to objects classified at that integrity level or lower.<br>
<br>```"Subject John has a biba level of high, so he can only write to objects with his integrity level or lower"```<br><br>

```"Subject John cannot have a biba level of low and write up to any objects with integrity levels above his"```
<br><br><br><br>

Biba Read rule: A subject with an integrity-clearance level of ```low``` may only read objects classified at his integrity level or higher<br><br>
```"Subject John has a biba level of low can only read to objects classified at his integrity level or higher"```.<br><br>

```"Subject John cannot have a biba level of high and read down to any objects with integreity levels below his"```

<br><br>



***Step 1.*** Install FreeBSD and make sure to install the UFS filesystem option
<br><br><br><br>


***Step 2.*** Open up the ```/boot/loader.conf``` file and put these in there:
```
mac_biba_load="YES"
security.mac.biba.trust_all_interfaces=1

```
<br><br><br><br>

***Step 3.*** Edit the ```/etc/fstab``` and set the root partition to ```ro``` and then reboot.
<br><br><br><br>

***Step 4.*** Run ```tunefs -l enable /```  and reboot after
<br><br><br><br>

***Step 5.*** As the root user run ```mount -urw /``` and change the ```ro``` back to ```rw``` in the /etc/fstab and then reboot again.
<br><br><br><br>

***Step 6.*** Edit the ```/etc/login.conf``` towards the top for our default user and make sure it looks like this below. The line we are specifically adding to this file is this

```
:label=biba/high(high-high):\
```
<br><br>

The whole ```default``` section should look like this now:

```
default:\
	:passwd_format=sha512:\
	:copyright=/etc/COPYRIGHT:\
	:welcome=/etc/motd:\
	:setenv=MAIL=/var/mail/$,BLOCKSIZE=K:\
	:path=/sbin /bin /usr/sbin /usr/bin /usr/games /usr/local/sbin /usr/local/bin ~/bin:\
	:nologin=/var/run/nologin:\
	:cputime=unlimited:\
	:datasize=unlimited:\
	:stacksize=unlimited:\
	:memorylocked=64K:\
	:memoryuse=unlimited:\
	:filesize=unlimited:\
	:coredumpsize=unlimited:\
	:openfiles=unlimited:\
	:maxproc=unlimited:\
	:sbsize=unlimited:\
	:vmemoryuse=unlimited:\
	:swapuse=unlimited:\
	:pseudoterminals=unlimited:\
	:priority=0:\
	:ignoretime@:\
	:umask=022:
	:label=biba/high(high-high):\
	:charset=UTF-8:\
	:lang=C.UTF-8:
  
  ```
  <br><br><br><br>
  
  ***Step 7.*** Continue to edit the ```/etc/login.conf``` file and scroll all the way to the bottom and add ```myuser``` like so:
  
  ```
  myuser:\
	:label=biba/low(low-high):\
	:tc=default:
  
  ```
  <br><br><br><br>
  
  ***Step 8.*** Once you are done editing the ```/etc/login.conf``` file we will now map our users to the users in the file. First we will map our regular user named ```john``` to our ```myuser``` in the file like so:
  
  
```
pw usermod john -L myuser

```

By doing this our user John whenever he logs into the system will have a clearance of ```biba/low(low-high)```

<br><br><br><br>
***Step 9.*** Now we will map our root user to the default user in ```/etc/login.conf``` like so:

```
pw usermod root -L default
```


By doing this our root user whenever he logs into the system will have a clearance of ```biba/high(high-high)```
<br><br><br><br>


***Step 10.*** Remember the two fundamentals of Biba Policy are ```No write up, No read down```. Let's test the policy out by trying to write up and see if it really works. Let's log into our regular user 'john'. If you run the following command you will see he has a label of ```biba/low(low-high)```

```
getpmac
biba/low(low-high)
```

<br><br>

Biba Policy states that our regular user ```john``` should not be able to write to integrity levels higher than his. To test this let us log out and log back in as the ```root``` user. Our label should now be ```biba/high(high-high)```. To test this run the following command:

```
getpmac
biba/high(high-high)
```

<br><br><br><br>

***Step 11.*** Create a file and write something in the file. In my example I will create a file called mytest and just write ```This is a test!``` inside of the file:

```
touch mytest
echo "This is a test!" >> mytest
```

<br><br><br><br>


***Step 12.*** Log out and log back in as the user ```john```. Change into the ```/root``` directory and you will see the file ```mytest```. Now let's try writing to the file as our regular user with a label of ```biba/low(low-high)```:

```
echo "This is not a test!" >> mytest
-sh: cannot create mytest: Permission denied
```

As you can see it won't let us write to the file. You can also open up the file instead of using the echo statement and it will say ```[File 'mytest' is unwritable ]```

<br><br><br><br>


***Step 13.*** Since we proved that the Biba policy won't let us 'write up' we will try the other one--which is 'no read down'. Since we are still logged in as our regular user ```john``` go ahead and create a file named ```mytest2``` and write inside the file ```this is another test!```

```
touch mytest2
echo "This is another test!" >> mytest2
```
<br><br>
Log back out and log in as ```root```. Change into the ```/home/john``` directory and you will be denied.

```
/home/john: Permission denied
```

<br><br><br><br>


***Step 14.*** We can't even get into the directory to be able to read the file. If you log out and log in with the ```john``` user and run the following command you will see their home directory is labeled with a ```biba/high``` so you would think since the root user also has a ```biba/high``` it would be able to change into the directory and read the file but nope it won't work.

```
getpmac
biba/low(low-high)
```
<br><br>
Reason being is that since our user ```john``` has a label of ```biba/low(low-high)``` the root user can't read into his directory. Even if john changes the label of his ```mytest2``` file to ```biba/high``` the ```root user``` still will not be allowed access to view the contents of ```/home/john``` because the root user doesn't have an integrity level of 'low' at all in their name. Their integrity level/label is ```biba/high(high-high)```.

<br><br>
Remember our 'no read down' rule that our Biba policy has. Subject John cannot have a biba level of high and read down to any objects with integreity levels below his.













