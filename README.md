Setting Up Network Access Control with FreeRADIUS, CoovaChilli, and MySQL on Ubuntu 22.04
________________________________________
Prerequisites
Before starting, ensure you have:
•	Hardware:
o	A computer with Ubuntu 22.04 LTS installed.
o	Two network interfaces:
	One for the internet (e.g., eth0, connected to a router/modem).
	One for Wi-Fi clients (e.g., wlan0, a wireless card that supports Access Point mode).
o	Check if your wireless card supports AP mode by running:
iw phy | grep -A 5 -i 'Supported interface modes' | grep '* AP'
If you see * AP, your card is compatible.
•	Software:
o	Internet access on the internet-facing interface.
o	A terminal to run commands (open it by pressing Ctrl + Alt + T).
•	Access:
o	Ability to use sudo (administrator privileges).
•	Note: This guide uses eth0 for the internet and wlan0 for Wi-Fi. Replace these with your actual interface names (find them using ip addr).
________________________________________
Step-by-Step Instructions
Step 1: Update Your System
To ensure your system has the latest software and security updates, run the following command in the terminal. This prevents issues during installation.
1.	Open the terminal (Ctrl + Alt + T).
2.	Type and run:
sudo apt update && sudo apt upgrade -y
3.	Enter your user password when prompted.
4.	Wait for the updates to complete (this may take a few minutes).
Step 2: Install Required Software
We need to install several programs to make this system work:
•	FreeRADIUS: Manages user authentication.
•	MySQL: Stores user data.
•	CoovaChilli: Creates the captive portal.
•	Hostapd: Turns your wireless card into a Wi-Fi access point.
•	Dnsmasq: Handles IP addresses and DNS for clients.
•	Nginx and PHP: Serves the login webpage.
Run this command to install everything:
sudo apt install -y freeradius freeradius-mysql mysql-server mysql-client hostapd dnsmasq nginx php8.1-fpm php8.1-mysql build-essential libssl-dev libjson-c-dev gengetopt devscripts debhelper haserl
•	If prompted, press Y to confirm installation.
•	This command installs all necessary packages. It may take some time.
Step 3: Set Up MySQL Database
MySQL will store user login information. We’ll create a database and a user for FreeRADIUS.
3.1 Secure MySQL
1.	Run the MySQL secure installation script to set a root password and secure the database:
sudo mysql_secure_installation
2.	Follow the prompts:
o	Set root password: Choose a password (e.g., mysqlsecret). Write it down; you’ll need it later.
o	Remove anonymous users: Type Y.
o	Disallow root login remotely: Type Y.
o	Remove test database: Type Y.
o	Reload privilege tables: Type Y.
3.2 Create Database and User
1.	Log in to MySQL:
sudo mysql -u root -p
Enter the MySQL root password (mysqlsecret) you set above.
2.	Run these commands one by one (copy and paste each, then press Enter):
3.	CREATE DATABASE radius;
4.	CREATE USER 'radius'@'localhost' IDENTIFIED BY 'mysqlsecret';
5.	GRANT ALL PRIVILEGES ON radius.* TO 'radius'@'localhost';
6.	FLUSH PRIVILEGES;
EXIT;
o	This creates a database named radius and a user radius with password mysqlsecret.
3.3 Set Up FreeRADIUS Tables
1.	Import the FreeRADIUS database schema to create tables for user data:
sudo mysql -u root -p radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql
2.	Enter the MySQL root password (mysqlsecret) when prompted.
3.	This sets up tables like radcheck for user credentials.
Step 4: Install CoovaChilli
CoovaChilli may not be in Ubuntu’s default repositories, so we’ll download and build it from source.
1.	Download CoovaChilli:
2.	cd /tmp
3.	wget https://github.com/coova/coova-chilli/releases/download/1.6/coova-chilli-1.6.tar.gz
4.	tar -xzf coova-chilli-1.6.tar.gz
cd coova-chilli-1.6


https://github.com/coova/coova-chilli/archive/refs/tags/1.6.tar.gz
5.	Remove Dependency (to avoid build issues):
sed -i '/haserl/d' debian/control
6.	Build and Install:
7.	sudo dpkg-buildpackage -b -uc
8.	cd ..
sudo dpkg -i coova-chilli_1.6_amd64.deb
o	If you get errors about missing dependencies, run:
sudo apt-get install -f
9.	Enable and Start CoovaChilli:
10.	sudo systemctl enable chilli
sudo systemctl start chilli
Step 5: Configure Network Interfaces
Your system needs to route traffic between the internet (WAN) and Wi-Fi clients (LAN).
1.	Find Interface Names: Run:
ip addr
Look for your interfaces (e.g., eth0 for internet, wlan0 for Wi-Fi). Note their names.
2.	Enable IP Forwarding: To allow your system to route traffic:
sudo sysctl -w net.ipv4.ip_forward=1
3.	Make this setting permanent:
sudo nano /etc/sysctl.conf
o	Find or add this line:
net.ipv4.ip_forward=1
o	Save (Ctrl+O, Enter, Ctrl+X to exit).
o	Apply changes:
sudo sysctl -p
Step 6: Configure Hostapd
Hostapd makes your wireless card act as a Wi-Fi access point.
1.	Create Hostapd Configuration:
sudo nano /etc/hostapd/hostapd.conf
Add (replace wlan0 with your Wi-Fi interface):
interface=wlan0
driver=nl80211
ssid=Hotspot
hw_mode=g
channel=6
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
wpa_passphrase=hotspotpass
o	ssid: Name of your Wi-Fi (e.g., Hotspot).
o	wpa_passphrase: Wi-Fi password (e.g., hotspotpass).
o	Save (Ctrl+O, Enter, Ctrl+X).
2.	Link Configuration:
sudo nano /etc/default/hostapd
o	Find or add:
DAEMON_CONF="/etc/hostapd/hostapd.conf"
o	Save and exit.
3.	Start Hostapd:
4.	sudo systemctl enable hostapd
sudo systemctl start hostapd
Step 7: Configure FreeRADIUS
FreeRADIUS will authenticate users against the MySQL database.
1.	Enable MySQL Module:
sudo ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/sql
2.	Configure SQL Module:
sudo nano /etc/freeradius/3.0/mods-available/sql
o	Update these lines:
o	driver = "rlm_sql_mysql"
o	dialect = "mysql"
o	server = "localhost"
o	port = 3306
o	login = "radius"
o	password = "mysqlsecret"
radius_db = "radius"
o	Ensure read_clients = yes is uncommented (remove # if present).
o	Save and exit.
3.	Configure FreeRADIUS Clients:
sudo nano /etc/freeradius/3.0/clients.conf
o	Add:
o	client localhost {
o	    ipaddr = 127.0.0.1
o	    secret = radtesting123
o	    require_message_authenticator = no
o	    nas_type = other
}
o	Save and exit.
4.	Start FreeRADIUS:
5.	sudo systemctl enable freeradius
sudo systemctl start freeradius
6.	Test FreeRADIUS:
o	Add a test user:
echo "INSERT INTO radcheck (username, attribute, op, value) VALUES ('testuser', 'Cleartext-Password', ':=', 'testpass');" | mysql -u radius -pmysqlsecret radius
o	Test authentication:
radtest testuser testpass 127.0.0.1 0 radtesting123
	If you see Access-Accept, FreeRADIUS is working.
Step 8: Configure CoovaChilli
CoovaChilli manages the captive portal and integrates with FreeRADIUS.
1.	Edit Configuration:
sudo nano /etc/chilli.conf
o	Add or update (replace eth0 and wlan0 with your interfaces):
o	HS_WANIF=ens33
o	HS_LANIF=ens37
o	HS_NETWORK=10.1.0.0
o	HS_NETMASK=255.255.255.0
o	HS_UAMLISTEN=10.1.0.1
o	HS_UAMPORT=3990
o	HS_UAMUIPORT=4990
o	HS_NASID=nas01
o	HS_RADIUS=localhost
o	HS_RADIUS2=localhost
o	HS_RADSECRET=radtesting123
o	HS_UAMSECRET=uamtesting123
o	HS_UAMALLOW=10.1.0.0/24
o	HS_DNS1=10.1.0.1
o	HS_DNS2=208.67.220.220
o	HS_TCP_PORTS="80 443"
o	HS_ADMUSR=admin
HS_ADMPWD=adminpass
o	Save and exit.
2.	Set Up Firewall: Allow HTTP and HTTPS traffic:
3.	sudo iptables -A INPUT -i tun0 -p tcp -m tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -i tun0 -p tcp -m tcp --dport 443 -j ACCEPT
4.	Save Firewall Rules:
5.	sudo apt install iptables-persistent -y
sudo iptables-save > /etc/iptables/rules.v4
o	Press Y if prompted to save rules.
Step 9: Set Up Captive Portal with Nginx
Create a login page for users to authenticate.
1.	Create Login Page:
2.	sudo mkdir -p /var/www/html/chilli
sudo nano /var/www/html/chilli/hotspotlogin.php
o	Add:
o	<?php
o	$uamip = $_GET['uamip'];
o	$uamport = $_GET['uamport'];
o	$challenge = $_GET['challenge'];
o	$mac = $_GET['mac'];
o	$ip = $_GET['ip'];
o	?>
o	<!DOCTYPE html>
o	<html>
o	<head>
o	    <title>Hotspot Login</title>
o	</head>
o	<body>
o	    <h2>Login to Hotspot</h2>
o	    <form action="http://<?php echo $uamip; ?>:<?php echo $uamport; ?>/logon" method="post">
o	        <input type="hidden" name="challenge" value="<?php echo $challenge; ?>">
o	        <input type="hidden" name="uamip" value="<?php echo $uamip; ?>">
o	        <input type="hidden" name="uamport" value="<?php echo $uamport; ?>">
o	        <label>Username: <input type="text" name="UserName"></label><br>
o	        <label>Password: <input type="password" name="Password"></label><br>
o	        <input type="submit" value="Login">
o	    </form>
o	</body>
</html>
o	Save and exit.
3.	Configure Nginx:
sudo nano /etc/nginx/sites-available/chilli
o	Add:
o	server {
o	    listen 80;
o	    server_name 10.1.0.1;
o	    root /var/www/html/chilli;
o	    index hotspotlogin.php;
o	    location ~ \.php$ {
o	        include snippets/fastcgi-php.conf;
o	        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
o	        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
o	        include fastcgi_params;
o	    }
}
o	Save and exit.
4.	Enable Nginx Site:
5.	sudo ln -s /etc/nginx/sites-available/chilli /etc/nginx/sites-enabled/
sudo systemctl restart nginx
6.	Link Login Page in CoovaChilli:
sudo nano /etc/chilli.conf
o	Add:
HS_UAMHOMEPAGE=http://10.1.0.1/hotspotlogin.php
o	Save and restart CoovaChilli:
sudo systemctl restart chilli
Step 10: Configure Dnsmasq
Dnsmasq assigns IP addresses to Wi-Fi clients.
1.	Edit Configuration:
sudo nano /etc/dnsmasq.conf
o	Add:
o	interface=wlan0
dhcp-range=10.1.0.2,10.1.0.254,255.255.255.0,12h
o	Save and exit.
2.	Start Dnsmasq:
3.	sudo systemctl enable dnsmasq
sudo systemctl start dnsmasq
Step 11: Add Users
Add a user to the MySQL database for testing.
echo "INSERT INTO radcheck (username, attribute, op, value) VALUES ('user1', 'Cleartext-Password', ':=', 'pass1');" | mysql -u radius -pmysqlsecret radius
•	This creates a user user1 with password pass1. Repeat for more users, changing the username and password.
Step 12: Test the System
1.	Connect to Wi-Fi:
o	On a phone or laptop, connect to the Wi-Fi network Hotspot with password hotspotpass.
2.	Open a Browser:
o	Try visiting any website (e.g., google.com).
o	You should be redirected to http://10.1.0.1/hotspotlogin.php.
3.	Log In:
o	Enter user1 and pass1.
o	If successful, you should get internet access.
4.	Check Logs:
o	FreeRADIUS logs:
sudo tail -f /var/log/freeradius/radius.log
Look for Access-Accept.
o	CoovaChilli clients:
sudo chilli_query list
Step 13: Optional - Limit User Access Time
To limit users to 30 minutes per day:
1.	Enable SQL Counter:
sudo nano /etc/freeradius/3.0/mods-available/sqlcounter
o	Add:
o	sqlcounter dailycounter {
o	    counter_name = Daily-Session-Time
o	    check_name = Max-Daily-Session
o	    reply_name = Session-Timeout
o	    sqlmod_inst = sql
o	    key = username
o	    reset = daily
o	    query = "SELECT SUM(acctsessiontime) FROM radacct WHERE username = '%{%k}' AND UNIX_TIMESTAMP(acctstarttime) + acctsessiontime > '%b'"
}
o	Save and exit.
o	Enable it:
sudo ln -s /etc/freeradius/3.0/mods-available/sqlcounter /etc/freeradius/3.0/mods-enabled/
2.	Set Time Limit:
echo "INSERT INTO radcheck (username, attribute, op, value) VALUES ('user1', 'Max-Daily-Session', ':=', '1800');" | mysql -u radius -pmysqlsecret radius
o	This limits user1 to 1800 seconds (30 minutes) daily.
3.	Restart FreeRADIUS:
sudo systemctl restart freeradius
Step 14: Troubleshooting
If something doesn’t work:
•	Check Service Status:
sudo systemctl status freeradius chilli hostapd nginx dnsmasq
o	Ensure all services are active (running).
•	View Logs:
o	FreeRADIUS: sudo tail -f /var/log/freeradius/radius.log
o	CoovaChilli: sudo tail -f /var/log/chilli.log
o	Nginx: sudo tail -f /var/log/nginx/error.log
•	Common Issues:
o	No Redirect to Login Page: Check HS_UAMHOMEPAGE in /etc/chilli.conf and ensure Nginx is running.
o	Authentication Fails: Ensure HS_RADSECRET in /etc/chilli.conf matches secret in /etc/freeradius/3.0/clients.conf.
o	No IP Address: Verify dnsmasq is running and configured for wlan0.
________________________________________
Final Notes
•	Security: Use strong passwords for MySQL, FreeRADIUS, and CoovaChilli in a real environment.
•	Advanced Management: Consider installing daloRADIUS for a web interface to manage users:
sudo apt install daloradius
Configure it to connect to the radius database.
•	Testing: Connect a device, log in, and ensure internet access works. Check logs for confirmation.
