# Walkthrough: EC2 and Jupyter

### Overview

 1.  Launch an EC2 instance
 2.  Install [Anaconda](https://www.continuum.io/downloads)
 3.  Setup [Jupyter](http://jupyter.org/) (with Python 2.7, Python 3.5, and R kernels)
 4.  Install [MySQL Server](https://www.mysql.com/)
 5.  Connect to AWS Public Data Sets (both on S3 and EBS snapshots) _**(will add in a later update)**_

#### Legend
* "Quotes" = Field names, labels to look for
* `Backticks` = Values you enter or select
* **Bold** = Buttons
* _Italics_ = General empahsis
* [Links](#) = More information

### Launching the EC2 Instance

When I [recorded this process](https://www.youtube.com/watch?v=9ee_tc_p9nU), I knew I would use this server for a long time, so I decided beforehand that I would buy a [Reserved Instance](https://aws.amazon.com/ec2/purchasing-options/reserved-instances/). I also chose the size of instance for my particular use case. It is a good idea to check out the [AWS cost calculator](http://calculator.s3.amazonaws.com/index.html) to decide for yourself what size and type of an instance you should choose and to determine whether it makes sense to reserve the instance or just pay the on-demand rate.

##### To buy the reserved instance:
 1.  Open the [AWS Console](http://console.aws.amazon.com/) and sign in
 2.  Select your desired region in the upper-right corner
     * I chose `US West (Oregon)` because I am currently living in Seattle
 3.  Click on **EC2** under "Compute" to enter the EC2 Dashboard
 4.  In the Left Panel, click on **Reserved Instances**
 5.  Click on **Purchase Reserved Instances**
 6.  Select the criteria you wish, then click on **Search**, I used the following:
     * _Instance Type:_ `m4.large`
     * _Term:_ `1 month - 12 months`
     * _Offering:_ `All Upfront`
     * _Availability Zone:_ `us-west-2b`
 7.  Click **Add to Cart** next for the type you would like to order
 8.  Click **View Cart**
 9.  Click **Purchase** to place the order for the Reserved Instance

##### To launch the EC2 instance itself:
 1.  In the left panel of the EC2 Dashboard, click on **Instances**
 2.  Click on **Launch Instance**
 3.  Choose which OS image you would like to run on, click **Select** next to your image of choice
     * I used the `Amazon Linux AMI 2015.09.1 (HVM), SSD Volume Type - ami-f0091d91` image
 4.  Choose an instance type, then click **Next: Configure Instance Details**
     * To match my Reserved Instance from earlier, I chose `m4.large`
 5.  Change the configuration details as needed, then click **Next: Add Storage**
     * If using a Reserved Instance, change the "Subnet" to match that instances's Availability Zone
     * I chose the Subnet item ending in `Default in us-west-2b`
 6.  Update the storage settings, then click **Next: Tag Instance**
     * I increased the default volume Size to 20GB and left the other attributes with their defaults
 7.  Create any tags for your instances as desired, then click **Next: Configure Security Group**
     * I did not create any tags, but these can be useful when managing large fleets of servers
 8.  Add additional rules to handle the Jupyter traffic, then click **Review and Launch**
     * Change the "Security Group Name" to something useful, I used `jupyter-notebook`
     * Leave the default SSH rule unchanged
     * Add a rule with "Type" = `HTTPS`
     * Add a rule with "Type" = `Custom TCP Rule`, a "Port Range" of `8888`, and "Source" = `Anywhere`
     * _Note: That last rule can be almost any port of your choosing, but it must match the port you configure your Jupyter Nobtebook to listen on_
 9.  On the "Review and Launch" page, double check the configuration, then click **Launch**
     * At the prompt for key pairs, create a new one named `jupyter-ec2`
     * Download the key pair and save in a safe, secure location
     * Click **Launch Instances** to launch the instance
 10.  Click on **View Instances** to jump back to the EC2 Instance Dashboard
 11.  _Optional:_ While waiting for the instance to fully launch, update setup a subdomain
      * _Note: I use [Hover](https://www.hover.com) as my domain registrar so your exact steps may vary from the following_
      * Go to the page for editing DNS records, for Hover this is **Your Account** -> **Domains** -> **zachmueller.com** -> **DNS**
      * Click the **Add New** button
      * Change "Hostname" to `jupyter`
      * Change "Record Type" to `CNAME`
      * Change "Target Host" to the instance "Public DNS" value (you can find this on the Instance Dashboard in the AWS Console, though you may need to wait for the instance to fully launch for this value to appear)
      * Click **Save**
      * _Note: this change may take some time to propogate through the global DNS system_


### Connecting (via PuTTY)

For servers running in the cloud, you will need to remotely access them to change what is running on them. For Windows servers, you typically would use Remote Desktop while for Linux servers you will likely use SSH. Since I use a Windows PC as my main device for getting things done locally, I use a tool called [PuTTY](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html) to be able to access Linux servers remotely via SSH. For accessing AWS EC2 instances via PuTTY, you need to convert the key pair file you downloaded earlier into a format that PuTTY can use.

##### Converting the Key Pair file for use by PuTTY:
 1.  Run puttygen.exe
 2.  Make sure "Type of Key" is `SSH-2 RSA`
 3.  Click **Load**, change file types to `All Files (*.*)`
 4.  Locate the .pem file (the key pair file you downloaded earlier) and open it
 5.  Click **Save private key** without a passphrase
 6.  Name it `jupyter-ec2` as well (it will be a .ppk file)
 7.  Close puttygen.exe

##### Connecting to your EC2 instance via PuTTY:
 1.  Run PuTTY
 2.  Change Hostname to `ec2-user@jupyter.zachmueller.com`
     * The portion of the hostname before the `@` symbol tells PuTTY what user name you wish to log in as. For most Amazon Linux images (distinct from RHEL, Ubuntu, etc.), `ec2-user` is the default user name. Click [here](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html) for default user names for other images.
     * If you do not have your own domain with a subdomain set up, use the "Public DNS" value that cound be found on the EC2 Dashboard - Instances page
 3.  Leave the "Port" value as `22`
 4.  Ensure "Connection Type" is `SSH`
 5.  In the left panel, navigate to **Connection** -> **SSH** -> **Auth**
 6.  Click **Browse** and find the .ppk file from earlier
 7.  Back on the "Session" tab (left panel), enter a name in the textbox below "Saved Sessions" then click **Save**
 8.  Click **Open** to connect to the instance
 9.  On your first connection, a prompt will appear asking whether you trust the SSH key data from the server, you can click **Yes** to proceed to connecting to the server

### Installing Anaconda

##### Create a new user:
For purposes of security, it is best to run commands as a user with the least amount of access as possible. To do this, we will first create a new user under which the Jupyter Notebook will be installed and run. Here is how we create the new user (the first line) and switch to be logged in as that user (the second line):
```bash
sudo adduser jpuser
sudo su - jpuser
```

##### Download and install Anaconda:
You can find the latest version of Anaconda on [this page](https://www.continuum.io/downloads) (under "Anaconda for Linux"). The below command can be used to download the Anaconda installer to your server. You may change the exact link within the single quotes to point to whichever version of Anaconda you're interested in downloading:

```bash
wget -O anaconda.sh 'https://3230d63b5fc54e62148e-c95ac804525aac4b6dba79b00b39d1d3.ssl.cf1.rackcdn.com/Anaconda3-2.4.1-Linux-x86_64.sh'
```

After the download completes, it is a good idea to check that the file you downloaded exactly matches what you intended to download. To do this, simply check its MD5 hash against the official [Anaconda archive](https://repo.continuum.io/archive/):

```bash
md5sum anaconda.sh
```

For the version I downloaded, the hash should match `45249376f914fdc9fd920ff419a62263`. After verifying you have the right installer, go ahead and run it:

```bash
bash anaconda.sh
```

You'll need to carefully tap the `Enter` through the license text and type in `yes` at the prompt. It will ask you to verify or change the install directory. Press `Enter` to leave it at the default (`/home/jpuser/anaconda3`). When prompted about prepending the folder to your PATH variable, type `yes` and press `Enter`. After the install has completed, run the following command to apply the changes to your PATH variable (so that you may more easily run the Anaconda commands):

```bash
source .bashrc
```

After you have installed Anaconda, run the following command to make sure all the packages within Anaconda are up to date:

```bash
conda update --all
```

### Configuring Jupyter

For configuring and running the Jupyter Notebook, we must first set up a self-signed SSL certificate. Enter the following commands to complete these steps (line by line):
 1. Log out from being logged in as `jpuser`
 2. Change your directory to the `/etc` directory
 3. Create a self-signed SSL certificate and key, both located within the file `jupyter.pem`

```bash
exit
cd /etc
sudo openssl req -x509 -nodes -days 365 -newkey rsa:1024 -keyout jupyter.pem -out jupyter.pem
```

Switch back to the `jpuser` user and create the default Jupyter config file by running:

```bash
sudo su - jpuser
jupyter notebook --generate-config
```

Before opening that config file and editing it, it will be good to generate the needed password hash so you can copy it to later paste into the config file. Simply run the following to open the Python interpreter:

```bash
python
```

Once inside the Python interpreter (you should see `>>>` at the left of your command line), run this code and enter your desired password that will be used to access your Jupyter Notebook in the browser (yes `passwd()` is missing the `o` and the `r`, that's just how they named it):

```python
from notebook.auth import passwd; passwd()
```

After entering your password twice, type `quit()` and press `Enter` to quit out of the Python interpreter. To copy the hash value, at least when using PuTTY, use your mouse to select the full text of the hash (include the single quotes on both ends of the string) then right-click on the mouse. This will both copy and simultaneously paste that text. You can simply delete the pasted text from the command line. You may optionally paste the hash value in a Notepad editor locally on your machine to ensure you don't accidentally overwrite your clipboard.

Now that you have your password hash, let's open up the Jupyter config file you created earlier and edit the settings (`nano` is the simplest of the command line text editors to learn for people just entering programming, I choose to not get myself entangled in the [emacs vs vim](https://en.wikipedia.org/wiki/Editor_war) war):

```bash
nano .jupyter/jupyter_notebook_config.py
```

From within `nano`, add the following lines anywhere to the file:

```python
c.NotebookApp.ip = '*'
c.NotebookApp.password = u'' (paste in the password here)
c.NotebookApp.open_browser = False
c.NotebookApp.port = 8888
c.NotebookApp.certfile = u'/etc/jupyter.pem'
c.NotebookApp.keyfile = u'/etc/jupyter.pem'
```

Hit `Ctrl + X` to quit out of `nano`, then press `Y` to confirm you want to save the changes, then press `Enter` to apply and save the changes.

Back at the command line, make a new directory for storing the work from within the Jupyter Notebook, move into that directory, then launch the Jupyter Notebook:

```bash
mkdir jupyter
cd jupyter
jupyter notebook
```

You can now access the Jupyter Notebook via your browser by going to a URL like this: 
https://ec2-12-34-567-980.us-west-2.compute.amazonaws.com:8888. The link you should use will vary, here are the key component:
 1.  `https://` will be required at the beginning of the URL because you are accessing the Notebook via the HTTPS protocol.
 2.  `ec2-12-34-567-980.us-west-2.compute.amazonaws.com` should be replaced with your EC2 instance's Public DNS value. This could also be your domain or whatever subdomain you configured on your domain.
 3.  `:8888` tells the browser to access the URL via port 8888, which is what we specified earlier in the process.

After you run the `jupyter notebook` command, you will need to press `Ctrl + C` then enter `y` to stop running the notebook. This will allow you to return to the command line to continue with setting up the system. _Note: this will also stop running your Jupyter Notebook, so you will no longer be able to access it via the browser until you run the command again._

### Installing Kernels

To install the kernels, ensure you are logged on as the `jpuser` and navigated to your home directory (you can use the `cd ~` command to navigate to your home directory from anywhere). To install the various kernels concurrently, you need to create virtual environments in which Python 2.7 vs Python 3.5 can run. Let's start by creating the Python 2.7 environment. Run the following commands and say `y`/`yes` to any prompts as they appear:

```bash
conda create -n py27 python=2.7 
source activate py27
conda install anaconda
ipython kernel install --user
```
This creates a new environment named `py27` using Python 2.7, activates that environment, installs all of the Python packages within Anaconda, then installs the IPython (Jupyter) kernel for Python 2.7. Similarly for Python 3.5 run:

```bash
conda create -n py35 python=3.5
source activate py35
conda install anaconda
ipython kernel install --user
source deactivate
```

Similarly to the `py27` steps, this creates a `py35` running Python 3.5 and installs the necessary kernel. It also deactivates the environment so you are back to your normal command line environment.

Next, we can also choose to install the R kernel for Jupyter. Luckily, Anaconda makes this task trivial:

```bash
conda install -c r r-essentials
```

Once all the kernels are installed, you may re-run the `jupyter notebook` command and test them out via your browser. _Note: whatever directory you are in when you run the `jupyter notebook` command determines where the Jupyter Notebook will default to when being accessed in the browser. You should use the `cd` command to change to the `jupyter` directory (or other directory of your choice) before launching the Notebook._

### Installing MySQL

While still logged in as `jpuser`, install the MySQL Python module (I have only found this to work under Python 2.7, hence activating the `py27` environment before installing the module):

```bash
source activate py27
pip install mysql-connector-repackaged
source deactivate
```

Now you will need to switch back to the `ec2-user` account in order to install the MySQL server to be accessible across the entire server (not just accessible by Jupyter). To switch back to the `ec2-user`, simply run the following command:

```bash
exit
```

As the `ec2-user`, execute the following command and type `y` at the prompt:

```bash
sudo yum install mysql55-server
```

Next, start up the mysql service with:

```bash
sudo service mysqld start
```

Then run the special shell script to securely setup your MySQL server:

```bash
sudo mysql_secure_installation
```

You will be walked through a few steps:
 1.  The current root password is blank, so just press `Enter` to skip through that
 2.  Type `Y` at all the remaining prompts, after the first one you will need to enter a new root password twice (do not forget this password)

Now that you have mysql setup and running, it's a good idea to configure your server to automatically start the `mysqld` service if the server ever reboots:

```bash
sudo chkconfig mysqld on
```

With MySQL fully configured, let's log into it and create a user that can access it from within Jupyter. First, run the command line command to log into MySQL (you will be prompted to enter the root password you just created):

```bash
mysql -u root -p
```

Within the MySQL command line (you should see a `>` on the left-hand side), run the following MySQL scripts:

```sql
CREATE DATABASE tempdb;
CREATE USER 'jupyter'@'localhost' IDENTIFIED BY 'testing123';
GRANT ALL ON tempdb.* TO 'jupyter'@'localhost';
FLUSH PRIVILEGES;
quit
```

You may change the name of the database (`tempdb`) and user name (`jupyter`) as desired. Also, be sure to set a more secure password (`testing123` in the above).

Check out some code examples of using the mysql-connector module in Python can be found [here](http://dev.mysql.com/doc/connector-python/en/connector-python-example-cursor-select.html).
